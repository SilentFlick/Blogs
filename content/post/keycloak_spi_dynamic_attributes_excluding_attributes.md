+++
author = "TMH"
email = "dev@tanminhho.com"
title = "Fine Tuning Keycloak Dynamic Attributes: Exclusion Lists"
date = "2025-05-18"
tags = [
    "keycloak", "keycloak-spi"
]
series = [
	"Keycloak"
]
+++

In my previous post, I built an [SPI]({{< relref "keycloak_spi_dynamic_attributes.md" >}}) that automatically inject all user
attributes into OIDC tokens or UserInfo. While powerful, I realized there are scenarios where I might want to omit
certain sensitive or irrelevant attributes. In this follow-up, I'll extend the SPI to support a configurable exclusion
list, giving me granular control over which attributes get mapped.

## SPI configuration changes

First, I add a new configuration option in my mapper that lets me list out attributes to skip.

{{<highlight java>}}
public static final String EXCLUDE_ATTRIBUTES = "exclude.attributes";
private static final List<ProviderConfigProperty> configProperties;

static {
	List<ProviderConfigProperty> props = new ArrayList<>( );
	OIDCAttributeMapperHelper.addIncludeInTokensConfig( props , AttributesMapper.class );

	ProviderConfigProperty excludeProperty = new ProviderConfigProperty( );
	excludeProperty.setName( EXCLUDE_ATTRIBUTES );
	excludeProperty.setLabel( "Attributes to exclude" );
	excludeProperty.setHelpText( "Comma-separated list of attribute name to exclude" );
	excludeProperty.setType( ProviderConfigProperty.STRING_TYPE );
	props.add( excludeProperty );

	configProperties = Collections.unmodifiableList( props );
}
{{</highlight>}}

This adds a new "Attribute to exclude" field in the mapper settings, where I can enter something like `duns, lastLogin`.

## Mapping logic with exclusions

Next, I update the core mapping logic to parse that list and skip any matching attribute keys:

{{<highlight java>}}
@Override
protected void setClaim(IDToken token , ProtocolMapperModel mappingModel , UserSessionModel userSession ,
                        KeycloakSession keycloakSession , ClientSessionContext clientSessionCtx) {
	var user = userSession.getUser( );
	var attributes = user.getAttributes( );
	if ( attributes == null ) return;

	String excludeList = mappingModel.getConfig( ).get( EXCLUDE_ATTRIBUTES );
	Set<String> excludeSet = new HashSet<>( );
	if ( excludeList != null && !excludeList.isEmpty( ) ) {
		for (String attribute : excludeList.split( "," )) {
			excludeSet.add( attribute.trim( ) );
		}
	}

	var dynamicMapping = new ProtocolMapperModel( );
	dynamicMapping.setId( mappingModel.getId( ) );
	dynamicMapping.setName( mappingModel.getName( ) );
	dynamicMapping.setProtocol( mappingModel.getProtocol( ) );
	dynamicMapping.setProtocolMapper( mappingModel.getProtocolMapper( ) );
	Map<String, String> config = mappingModel.getConfig( ) != null
			? new HashMap<>( mappingModel.getConfig( ) )
			: new HashMap<>( );

	for (var entry : attributes.entrySet( )) {
		var key = entry.getKey( );
		if ( excludeSet.contains( key ) ) continue;
		var value = entry.getValue( );
		if ( value == null || value.isEmpty( ) ) continue;
		var claimValue = (value.size( ) == 1) ? value.get( 0 ) : value;
		config.put( "claim.name" , key );
		config.put( "aggregate.attrs" , "true" );
		if ( value.size( ) > 1 ) {
			config.put( "multivalued" , "true" );
		}
		dynamicMapping.setConfig( config );
		OIDCAttributeMapperHelper.mapClaim( token , dynamicMapping , claimValue );
	}
{{</highlight>}}

## The complete mapper class

For reference, here's the full updated AttributesMapper with exclusion support:

{{<highlight java>}}
package com.tmh.attributes;

import org.keycloak.models.ClientSessionContext;
import org.keycloak.models.KeycloakSession;
import org.keycloak.models.ProtocolMapperModel;
import org.keycloak.models.UserSessionModel;
import org.keycloak.protocol.oidc.mappers.*;
import org.keycloak.provider.ProviderConfigProperty;
import org.keycloak.representations.IDToken;

import java.util.*;

public class AttributesMapper extends AbstractOIDCProtocolMapper
		implements OIDCAccessTokenMapper, OIDCIDTokenMapper, UserInfoTokenMapper {
	public static final String PROVIDER_ID = "oidc-tmh-attributes";
	public static final String EXCLUDE_ATTRIBUTES = "exclude.attributes";
	private static final List<ProviderConfigProperty> configProperties;

	static {
		List<ProviderConfigProperty> props = new ArrayList<>( );
		OIDCAttributeMapperHelper.addIncludeInTokensConfig( props , AttributesMapper.class );

		ProviderConfigProperty excludeProperty = new ProviderConfigProperty( );
		excludeProperty.setName( EXCLUDE_ATTRIBUTES );
		excludeProperty.setLabel( "Attributes to exclude" );
		excludeProperty.setHelpText( "Comma-separated list of attribute name to exclude" );
		excludeProperty.setType( ProviderConfigProperty.STRING_TYPE );
		props.add( excludeProperty );

		configProperties = Collections.unmodifiableList( props );
	}

	@Override public String getDisplayCategory() {
		return TOKEN_MAPPER_CATEGORY;
	}

	@Override public String getDisplayType() {
		return "Dynamic User Attributes";
	}

	@Override public String getHelpText() {
		return "Dynamically add all user attributes to userinfo";
	}

	@Override public List<ProviderConfigProperty> getConfigProperties() {
		return configProperties;
	}

	@Override public String getId() {
		return PROVIDER_ID;
	}


	@Override
	protected void setClaim(IDToken token , ProtocolMapperModel mappingModel , UserSessionModel userSession ,
	                        KeycloakSession keycloakSession , ClientSessionContext clientSessionCtx) {
		var user = userSession.getUser( );
		var attributes = user.getAttributes( );
		if ( attributes == null ) return;

		String excludeList = mappingModel.getConfig( ).get( EXCLUDE_ATTRIBUTES );
		Set<String> excludeSet = new HashSet<>( );
		if ( excludeList != null && !excludeList.isEmpty( ) ) {
			for (String attribute : excludeList.split( "," )) {
				excludeSet.add( attribute.trim( ) );
			}
		}

		var dynamicMapping = new ProtocolMapperModel( );
		dynamicMapping.setId( mappingModel.getId( ) );
		dynamicMapping.setName( mappingModel.getName( ) );
		dynamicMapping.setProtocol( mappingModel.getProtocol( ) );
		dynamicMapping.setProtocolMapper( mappingModel.getProtocolMapper( ) );
		Map<String, String> config = mappingModel.getConfig( ) != null
				? new HashMap<>( mappingModel.getConfig( ) )
				: new HashMap<>( );

		for (var entry : attributes.entrySet( )) {
			var key = entry.getKey( );
			if ( excludeSet.contains( key ) ) continue;
			var value = entry.getValue( );
			if ( value == null || value.isEmpty( ) ) continue;
			var claimValue = (value.size( ) == 1) ? value.get( 0 ) : value;
			config.put( "claim.name" , key );
			config.put( "aggregate.attrs" , "true" );
			if ( value.size( ) > 1 ) {
				config.put( "multivalued" , "true" );
			}
			dynamicMapping.setConfig( config );
			OIDCAttributeMapperHelper.mapClaim( token , dynamicMapping , claimValue );
		}
	}
}
{{</highlight>}}
