+++
author = "TMH"
email = "dev@tanminhho.com"
title = "Keycloak SPI: Dynamic Attributes"
date = "2025-03-15"
tags = [
    "keycloak", "keycloak-spi"
]
series = [
	"Keycloak"
]
+++

I have been working with Keycloak for Identity and Access Management (IAM) for quite some time. While it does an
excellent job handling access control, I often found myself frustrated with the way it handles user attributes. By
default, Keycloak disables unmanaged attributes, so adding custom attributes to users, roles, or groups isn’t as
straightforward as it could be.

## The Problem: Manual Attribute Mapping

Every time I needed a custom attribute to appear in a token (like UserInfo, AccessToken, or IdToken), I had to follow a
tedious four-step process:

1. Create a client scope
2. Create a client mapper with the type `User Attribute`
3. Configure the mapper
4. Add the scope to the client.

Not only was this process repetitive, but it also meant that in our multiple environments (development, testing,
production), each new attribute forced me to redo these steps. Even though I managed to automate part of it with a
Python script using the Keycloak Admin REST API, it still felt like a workaround rather than a solution.

## The Lightbulb Moment: A Custom Protocol Mapper

I thought, "What if I could make this process truly dynamic?" I decided to build a custom protocol mapper that
automatically maps all user attributes. This not only saves time but also keeps our configuration consistent across
different environments.

### Step 1: Configure the Custom Mapper

First, I set up the custom mapper to choose where the attributes should be added (UserInfo, AccessToken, or
IdToken). This configuration was pretty straightforward:

{{<highlight java>}}
private static final List<ProviderConfigProperty> configProperties = new ArrayList<>( );

static {
	OIDCAttributeMapperHelper.addIncludeInTokensConfig( configProperties , AttributesMapper.class );
}
{{</highlight>}}

This simple configuration allows my custom mapper to include dynamic attributes in the tokens.

### Step 2: Dynamically Mapping User Attributes

The real magic happens in the mapping logic. Using `mapClaim` from `OIDCAttributeMapperHelper`, I built a dynamic
mapping model that loops through each user attribute and adds it to the token. Here’s how I did it:

{{<highlight java>}}
@Override
protected void setClaim(IDToken token , ProtocolMapperModel mappingModel , UserSessionModel userSession ,
                        KeycloakSession keycloakSession , ClientSessionContext clientSessionCtx) {
	var user = userSession.getUser( );
	var attributes = user.getAttributes( );
	var dynamicMapping = new ProtocolMapperModel( );
	dynamicMapping.setId( mappingModel.getId( ) );
	dynamicMapping.setName( mappingModel.getName( ) );
	dynamicMapping.setProtocol( mappingModel.getProtocol( ) );
	dynamicMapping.setProtocolMapper( mappingModel.getProtocolMapper( ) );
	var config = new HashMap<>(
		Optional.ofNullable( mappingModel.getConfig( ) ).orElse( Collections.emptyMap( ) )
	);
	if ( attributes == null ) return;
	for (var entry : attributes.entrySet( )) {
		var key = entry.getKey( );
		var value = entry.getValue( );
		if ( value == null || value.isEmpty( ) ) continue;
		var claimValue = (value.size( ) == 1) ? value.get( 0 ) : value;
		config.put( "claim.name" ,  key );
		config.put( "aggregate.attrs" , "true" );
		if ( value.size( ) > 1 ) {
			config.put( "multivalued" , "true" );
		}
		dynamicMapping.setConfig( config );
		OIDCAttributeMapperHelper.mapClaim( token , dynamicMapping , claimValue );
	}
}
{{</highlight>}}

In this method, I:

+ Retrieved all the user's attributes.
+ Constructed a dynamic mapping model for each attribute.
+ Used the helper method to add each attribute to the token.
+ Handled both single and multi-valued attributes seamlessly.

## Constraints and Considerations

While the custom protocol mapper has streamlined my workflow, there are a couple of trade-offs to keep in mind:

1. **Value Type Limitation**:
The attribute values are always treated as strings. Automatically mapping these values to their correct types (like
numbers or booleans) could significantly increase code complexity. For the sake of simplicity and maintainability, I
chose to stick with strings.

2. **Token Size Concerns**:
Including too many attributes—especially groups or roles—can have unintended side effects. Adding large numbers of these
attributes to the access token might push the token size over the recommended limit of 4096 bytes. This is something to
be cautious about when designing the attribute strategy.

## Reflections

Building this custom protocol mapper has proven to be a practical solution that simplifies my daily tasks and reduces
repetitive configuration steps. However, it's important to be aware of the inherent trade-offs. If you’re encountering
similar challenges or are looking for a more efficient way to manage dynamic attributes in Keycloak, I hope my
experience offers valuable insights.

Happy coding!
