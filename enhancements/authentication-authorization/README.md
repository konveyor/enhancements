---
title: adding-extensible-authentication-with-oidc-and-oauth
  - "@shawn-hurley"
reviewers:
  - @pranavgaikwad 
  - @ibolton336 
  - @rromannissen 
  - @mansam 
  - @aufi 
approvers:
  - @jortel 
  - @dymurray 
  - @sjd78 
creation-date: 2024-06-10
last-updated: 2024-06-13
status: implementable
---

# Extend Authentication with OIDC and OAuth

## Release Signoff Checklist

- [ x ] Enhancement is `implementable`
- [ x ] Design details are appropriately documented from clear requirements
- [ x ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]
* Need Help from Scott to  enhance the UI section to fully capture what is needed
* Need a review from Jeff and Sam on the high-level Hub changes and make sure they are sane.
* Verify with Scott and Ian that the user refresh token is stored in the browser cache.


## Summary

As we move to more and more integrations with MTA, we need to consider the many different
types of auth solutions that exist. This means integrating with OpenID Connect(OIDC) and OAuth identity authentication
servers as defined in [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) and [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html). This will move authorization roles and scopes into the hub for administrator management. Keycloak will move to an optional component, that a user can bring, that will be responsible for the mapping of
LDAP/Kerberos into roles and will be used as any other OIDC system.

## Motivation

As we gain both more users, and more integrations, supporting generic authentication that would exist in a company
is going to make the user experience more complete. This will enable easier setup of a secure MTA environment by allowing
existing authentication mechanisms that already have all the user data. Admin users will now not have to manage user authentication data. 

### Goals

The goal of this enhancement is to make authentication generic, such that any authentication provider that supports OAuth and OIDC is usable as authorization and authentication. The goal is for admin users to have an easy configuration, end users have some default access when logging on and when using OIDC you can map roles to internal roles. Each of these is important to keep the user experience sane and not cumbersome. 

The other goal is to make integration with things like RHDH easier, by allowing users to share authentication between systems. 

### Non-Goals

* To connect or support native Kerbos/LDAP authentication or authorization

## Proposal

### User Stories [optional]

#### Story 1

As an admin user, I want to use an OAuth provider for migrator, architect and admin users to log in.

#### Story 2

As an admin user, I want to use an OIDC provider for migrator, architect and admin users to log in.

#### Story 3

As an admin user, I want to be able to give end user who logged in with a provider a specific role.

#### Story 4

As an admin user, I want to have a default role that a user would be granted once logged in, for immediate access.

#### Story 5

As an admin user, I want to map the roles from an OIDC provider to internal roles for automatic access.

#### Story 6

As an admin user, I want to create and update roles and change the scopes that a role will have access to.

### Details

#### Hub Changes

The hub will introduce four new tables:

1. mapping table for the user refresh token and associated identity fields (name, email, ID etc.) to the role that it has been granted. 
2. mapping table for a resource and HTTP verb to a given role. 
3. table for the provider configuration.
4. mapping table OIDC provider configuration roles mapping to internal roles

The hub will introduce new APIs that will be responsible for:
1. Returning the scopes for a given user token
2. A callback API that will be responsible for saving the user identity information
3. A basic password login API that will be used for the system admin default account
4. Saving/Getting the oauth provider configuration including mapping
5. Getting a list of the users with out the tokens and roles associated
6. Get/Saving User to internal role 
7. Get/Saving Scope to internal role 

Internally, the hub authorization system will need to add:
1. On each call, the ability to validate the saved refresh token or authorization token
2. Retrieve an updated refresh token, if it has expired.
3. Validate for a given resource and HTTP verb that the user has access to the resource being requested.

**The Hub will also need to create a system admin user, that is accessible over normal password authentication so that an
admin user can set up the external authentication providers.**

#### UI Changes

The UI will need a login page, for the system admin user to log in. It will also need three new admin screens.

##### Authentication Provider Configuration

This screen is responsible for saving the provider configuration. The screen will also need to be able to handle the OIDC role to internal role mapping. The mapping would be optional for an OIDC provider.

##### User Role Mapping

This screen is responsible, for getting the user information, and the internal roles, and allowing for mapping between the user and the roles. 

##### Scope to Role Mapping

This screen is going to be responsible for mapping all the available scopes (resource and HTTP Verb combinations).

#### Operator Changes

The operator is going to be responsible for handling the upgrade scenarios. See below for more details.

#### Analyzer/Provider changes

No changes to this are needed.

### Security, Risks, and Mitigations

This is a change to the security system and therefore will need to be treated with extreme care. We will need to have a 
very good test plan, and we will need to test with multiple auth providers. The key thing that we must do, is always fail closed, always check the authorization/refresh token for validity and have a test plan that covers all the scenarios.

## Design Details

### Test Plan

The test plan for this feature must be comprehensive and added to our CI stack. This means changes to the golang tests to:
1. Set up Keycloak with users for all the built-in roles and calls to endpoints that each role does not have access to, and verify a 403.
3. Set up Kubernetes OAuth, configure it, and do the same process.
4. Create a new Role, with different scopes, and verify that the user has access. 
5. verify that the removal of a role from a user, even if they have a token already, removes access to an endpoint.

### Upgrade / Downgrade Strategy

The operator is going to have to be responsible for the upgrade scenario. It will have to handle a couple of 
high-level concerns.

1. Must detach Keycloak from being managed by the operator if present
2. Must configure Keycloak to correctly serve OIDC
4. Must configure the OIDC provider for the new hub APIs

Users should not have to re-login, but this means that the hub will have to handle the case, where:
1. An access/refresh token is given in a request.
2. The hub does not find a user for this token.

To handle this, the hub will have to use the [user info](https://openid.net/specs/openid-connect-core-1_0-errata2.html#UserInfo) endpoints in OIDC to get the information required. Then it will have to add the user entry and use the mapping the operator sets up for the role. 

Then it can continue the process and verify the access.

## Implementation History

* Initial Draft of the enhancement

## Drawbacks

The biggest drawback of this approach is that an authorization system is being built in the hub. This means that we are responsible for things like revocation of roles, mappings for roles for OIDC, and more user interaction in the hub to manage this. Today, this is all in Keycloak, and if a user wants to manage any of this, they must go to the Keycloak admin page.

## Alternatives

We did consider moving to using Kubernetes built-in RBAC. The biggest drawback with this approach is that users of MTA would have to have logins for Kubernetes. 
