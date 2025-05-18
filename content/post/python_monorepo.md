+++
author = "TMH"
email = "dev@tanminhho.com"
title = "Cracking the Python Monorepo"
date = "2025-05-18"
tags = [
    "python", "monorepo"
]
+++

I recently set out to organize multiple Python services and libraries into a single monorepo. Drawing inspiration from
[Daniel Gafni's approach](https://gafni.dev/blog/cracking-the-python-monorepo/), I want to build and package each
service, including optional UIs, using a single, consistent Dagger pipeline, eliminating per-package scripts and
simplifying workflows.

## Workspace layout

A minimal structure looks like this:

{{<highlight bash>}}
/.dagger/         # Dagger SDK and config
/pyproject.toml   # workspace config, lists all packages
/uv.lock          # unified lockfile for all packages
/packages/
  ├─ service_a/
  ├─ lib_common/
  └─ service_b/
/ui/              # optional frontends
  ├─ service_a/
  └─ service_b/
{{</highlight>}}

- Root `pyproject.toml` declares all workspace members under `[tool.uv.workspace]` and `uv.lock` is the combined
  lockfile.
- Each package in `/packages/<project>` has its own `pyproject.toml` and source tree.
- Frontends (e.g. React/Angular/Vue/etc. apps) live under `/ui/<project>` and are built and mounted into the service
  container.
  
## High level build flow

1. Build UI (if applicable)
2. Prepare build container with Python, UV and workspace files
3. Parse `uv.lock` to collect the target package and its workspace dependencies
4. Copy code into the build container
5. Install and sync only the selected package and its extras
6. Assemble the final image

### Build entry point

{{<highlight python>}}
IGNORE = Ignore(
    [
        ".env",
        ".env.*",
        ".git",
        "**/.venv",
        "**__pycache__**",
        ".dagger/sdk",
        "**/.pytest_cache",
        "**/.ruff_cache",
        "**/*.tar",
        "**/*.csv",
        ".task",
        "**/node_modules",
    ]
)

PYTHON_VERSION = "3.12"
ALPINE_VERSION = "alpine"
BASE_IMAGE = f"python:{PYTHON_VERSION}-{ALPINE_VERSION}"

@object_type
class SctDagger:
    @function
    async def build(self, root_dir: Annotated[Directory, DefaultPath("."), IGNORE], project: str) -> Container:
        """
        Build the container for the specified project.

        The build sequence is as follows:
        1. Build the UI (if available).
        2. Set up the build container.
        3. Collect project sources including transitive dependencies.
        4. Copy source directories into the container.
        5. Set the working directory, mount static assets, and install dependencies.
        6. Set the final container entrypoint.

        :param root_dir: The root directory of the project containing dagger.json.
        :param project: The name of the project.
        :return: A configured container with the appropriate context and entrypoint.
        """
        ui_dist = await self.build_ui(root_dir=root_dir, project=project)
        container = self.prepare_build_container(
            pyproject_toml=root_dir.file("pyproject.toml"),
            uv_lock=root_dir.file("uv.lock"),
            project=project,
        )
        project_sources = await self.get_project_sources(uv_lock=root_dir.file("uv.lock"), project=project)
        container = self.copy_source(container=container, root_dir=root_dir, project_sources=project_sources)
        container = (
            container.with_workdir(path=f"/src/{project_sources[project]}")
            .with_directory(path=f"/src/{project_sources[project]}/src/{project}/static", directory=ui_dist)
            .with_exec(args=["uv", "sync", "--inexact", "--package", project])
        )
        return (
            dag.container()
            .from_(address=BASE_IMAGE)
            .with_env_variable(name="PATH", value="/src/.venv/bin:$PATH", expand=True)
            .with_directory(path="/src", directory=container.directory(path="/src"))
            .with_entrypoint(args=["python3", "-m", f"{project}.main"])
        )
{{</highlight>}}

- `build_ui` returns a build `dist` directory or an empty placeholder if no UI exists.
- `prepare_build_container` installs the `uv` cli and copy the `pyproject.toml` and `uv.lock` into the base image.
- `get_project_sources` parses `uv.lock` to collect editable workspace paths for the target package and its
  dependencies.
- `copy_sources` mounts each local package directory into `/src/<path>`.
- Finally, we create a slim runtime image with only the selected package.

### Prepare build container

{{<highlight python>}}
@staticmethod
def prepare_build_container(pyproject_toml: File, uv_lock: File, project: str) -> Container:
    """
    Prepare the build container for the specified project with necessary files.

    :param pyproject_toml: The pyproject.toml file of the project.
    :param uv_lock: The uv.lock file of the project.
    :param project: The project name.
    :return: A container configured for the build process.
    """
    uv_container = dag.container().from_(f"ghcr.io/astral-sh/uv:python{PYTHON_VERSION}-{ALPINE_VERSION}")
    return (
        dag.container()
        .from_(address=BASE_IMAGE)
        .with_file(path="/bin/uv", source=uv_container.file(path="/usr/local/bin/uv"))
        .with_file(path="/bin/uvx", source=uv_container.file(path="/usr/local/bin/uvx"))
        .with_env_variable(name="UV_COMPILE_BYTECODE", value="1")
        .with_env_variable(name="UV_LINK_MODE", value="copy")
        .with_env_variable(name="UV_FROZEN", value="1")
        .with_env_variable(name="UV_PYTHON_DOWNLOADS", value="0")
        .with_workdir(path="/src")
        .with_file(path="/src/pyproject.toml", source=pyproject_toml)
        .with_file(path="/src/uv.lock", source=uv_lock)
        .with_new_file(path="/src/README.md", contents=project)
        .with_mounted_cache(path="/root/.cache/uv", cache=dag.cache_volume("uv-cache"))
        .with_exec(args=["uv", "sync", "--no-install-workspace", "--no-dev", "--all-extras", "--package", project])
    )
{{</highlight>}}

- Installs `uv` and `uvx` binaries from the official UV image.
- Configures caching and environment variables for reproducible wheels.
- Seeds only the selected package into the virtual environment under `/src/.venv`.

### Gather sources

{{<highlight python>}}
@staticmethod
async def get_project_sources(uv_lock: File, project: str) -> dict[str, str]:
    """
    Retrieve the source directories for the project and its transitive dependencies.

    This method parses the uv.lock file to identify all packages that are part of the workspace,
    starting from the given project package and recursively collecting dependencies.
    Only dependencies that are workspace members and have an 'editable' source are included.

    :param uv_lock: The File object containing uv.lock data.
    :param project: The target project whose sources and dependencies are to be gathered.
    :return: A mapping from package names to their corresponding editable source paths.
    """
    uv_lock_dict = tomli.loads(await uv_lock.contents())
    members = set(uv_lock_dict["manifest"]["members"])
    packages = uv_lock_dict["package"]
    # Directory for quick look-up
    package_dict = {pkg["name"]: pkg for pkg in packages}

    def collect_deps(pkg_name: str, collected: set) -> None:
        if pkg_name in collected:
            return
        if pkg_name not in package_dict:
            return
        collected.add(pkg_name)
        pkg = package_dict[pkg_name]
        for dep in pkg.get("dependencies", []):
            dep_name = dep.get("name") if isinstance(dep, dict) else dep
            if dep_name in members:
                collect_deps(dep_name, collected)

    collected_deps = set()
    collect_deps(pkg_name=project, collected=collected_deps)

    project_sources = {}
    for deps in collected_deps:
        pkg = package_dict.get(deps)
        if pkg and "source" in pkg and "editable" in pkg["source"]:
            project_sources[deps] = pkg["source"]["editable"]

{{</highlight>}}

- Processes the lockfile to gather the target package plus any in‑workspace dependencies.
- Returns a mapping from package name to its local path (e.g. `packages/lib_common`).

### Copy sources

{{<highlight python>}}
@staticmethod
def copy_source(
    container: Container, root_dir: Annotated[Directory, DefaultPath("."), IGNORE], project_sources: dict[str, str]
) -> Container:
    """
    Copy the project source directories into the container.

    For each source path in project_sources, the corresponding directory is added to the container
    at the designated path.

    :param container: The container into which the source directories will be copied.
    :param root_dir: The project root directory used to locate the source directories.
    :param project_sources: A mapping of project/package names to their editable source paths.
    :return: The container updated with the source directories.
    """
    for project_source_path in project_sources.values():
        container = container.with_directory(
            path=f"/src/{project_source_path}",
            directory=root_dir.directory(project_source_path),
        )
    return container
{{</highlight>}}

Each local package directory is mounted into the build container, preserving relative paths.

### Optional UI build

{{<highlight python>}}
@function
async def build_ui(self, root_dir: Annotated[Directory, DefaultPath("."), IGNORE], project: str) -> Directory:
    """
    Build the UI for the given project if it exists.

    If the project UI directory is missing, return an empty directory.
    When the UI exists, the build process involves:
     - Using a Node container image (node:iron-alpine3.20).
     - Configuring environment variables and mounting a cache for PNPM.
     - Copying over the UI source files (excluding node_modules).
     - Running installation and build commands via PNPM.
     - Returning the built directory (/app/dist).

    :param root_dir: The root directory of the project containing dagger.json.
    :param project: The name of the project.
    :return: The built directory or an empty directory if not present.
    """
    ui = await root_dir.entries(path="ui")
    if f"{project}/" not in ui:
        return dag.directory()
    return (
        dag.container()
        .from_(address="node:iron-alpine3.20")
        .with_env_variable(name="PNPM_HOME", value="/pnpm")
        .with_mounted_cache(path="/pnpm/store", cache=dag.cache_volume("pnpm-store"))
        .with_directory(path="/app", directory=root_dir.directory(f"ui/{project}"), exclude=["node_modules/"])
        .with_workdir(path="/app")
        .with_exec(args=["corepack", "enable"])
        .with_exec(args=["pnpm", "install", "--frozen-lockfile"])
        .with_exec(args=["pnpm", "run", "build"])
        .directory(path="/app/dist")
    )
{{</highlight>}}

- Builds the UI with PNPM in a lightweight Node image.
- Mounts the output directory for later inclusion in the Python service.

## Task runner integration

Use a simple Taskfile (Taskfile.yml) to orchestrate install, lint, build, and run:

{{<highlight yaml>}}
version: "3"

env:
  MODE: '{{default "local" .MODE}}'
  DAGGER_NO_NAG: 1
  DO_NOT_TRACK: 1
dotenv: [ '.env/.env.{{.MODE}}' ]

tasks:
  install:
    desc: Install all dependencies.
    preconditions:
      - test -f pyproject.toml
      - command -v uv
    cmds:
      - uv sync --all-groups --all-packages
    sources:
      - pyproject.toml
      - packages/**/pyproject.toml
    generates:
      - uv.lock

  update:
    desc: Update all dependencies
    preconditions:
      - test -f pyproject.toml
      - command -v uv
    cmds:
      - uv sync --all-groups --all-packages --upgrade

  lint:
    desc: Lint
    preconditions:
      - command -v uv
    cmds:
      - uv run ruff format
      - uv run ruff check --fix

  build-ui-*:
    desc: Build ui
    vars:
      PACKAGE: "{{index .MATCH 0}}"
    cmds:
      - dagger call build-ui --project {{.PACKAGE}} export --path {{.ROOT_DIR}}/ui/{{.PACKAGE}}/dist

  start-package-*:
    internal: true
    desc: Start package
    vars:
      PACKAGE: "{{index .MATCH 0}}"
    env:
      UI_BUILD: "{{.ROOT_DIR}}/ui/{{.PACKAGE}}/dist/{{.PACKAGE}}/browser"
    cmds:
      - |
        if [[ -d ui/{{.PACKAGE}} && -f ui/{{.PACKAGE}}/package.json ]]; then
          sleep 5
        fi
      - uv run python3 -m {{.PACKAGE}}.main

  watch-ui-*:
    internal: true
    desc: Watch ui
    vars:
      PACKAGE: "{{index .MATCH 0}}"
    preconditions:
      - command -v pnpm
    cmds:
      - |
        if [[ -d ui/{{.PACKAGE}} && -f ui/{{.PACKAGE}}/package.json ]]; then
          cd ui/{{.PACKAGE}}
          pnpm run watch:dev
        fi

  serve-*:
    desc: Serve package
    vars:
      PACKAGE: "{{index .MATCH 0}}"
    preconditions:
      - command -v uv
    deps:
      - task: watch-ui-{{.PACKAGE}}
        silent: true
      - task: start-package-{{.PACKAGE}}
        silent: true

  build-package-*:
    desc: Build package
    vars:
      PACKAGE: "{{index .MATCH 0}}"
    preconditions:
      - command -v dagger
    cmds:
      - dagger call build --project {{.PACKAGE}} export --path {{.PACKAGE}}.tar
{{</highlight>}}

This Taskfile provides a unified, repeatable set of commands for dependency management, linting, UI builds, and image
packaging.
