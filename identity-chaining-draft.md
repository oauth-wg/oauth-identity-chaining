---
stand_alone: true
ipr: trust200902
cat: info
submissiontype: IETF
area: sec
wg: oauth

docname: draft-identity-chaining-00

title: Identity Chaining across Trust Domains
abbrev:
lang: en
kw: []
# date: 2022-02-02 -- date is filled in automatically by xml2rfc if not given
author:
- name: Arndt Schwenkschuster
  org: Microsoft
  email: arndts@microsoft.com
- name: Pieter Kasselmann
  org: Microsoft
  email: pieter.kasselman@microsoft.com
- name: Kelley Burgin
contributor:
- name: Atul Tulshibagwale
  org: SGNL
  email: atuls@sgnl.ai
- name: George Fletcher
  org: Capital One
  email: george.fletcher@capitalone.com


normative:
  RFC6749: # OAuth 2.0 Authorization Framework
  RFC6750: # The OAuth 2.0 Authorization Framework: Bearer Token Usage
  RFC8693: # OAuth 2.0 Token Exchange
  RFC7521: # Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants
  RFC7523: # JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants

informative:

  SD-JWT:
    title: Selective Disclosure for JWTs (SD-JWT)
    target: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-selective-disclosure-jwt-04
    author:
    - name: Daniel Fett
      org: yes.com
    - name: Kristina Yasuda
      org: Microsoft
    - name: Brian Campbell
      org: Ping Identity


--- abstract

This specification defines a protocol for Authorization Servers and Resource Servers by defining how to receive access tokens from different domains in context of the user or the client.

--- middle

# Introduction

Since the introduction of the OAuth 2.0 Authorization Framework {{RFC6749}} in 2012 it has found brought adoption in the industry and is still popular today.

Whilst clients, resource servers, resources and the authorization server are all part of one trust domain, developers are often faced with the situation that a protected resource is located in a different trust domain and thus protected by a different authorization server. A request can even take multiple hops across various trust domains until it finds its destination at a resource.

All protected resources along the way are challenged to correctly authorizing the end user and may require its claims from the previous trust domains to achieve that.

This situation is referred to as `Identity Chaining` in this memo.

## Requirements Language

{::boilerplate bcp14-tagged}

# Identity Chaining Flow

This specification describes a combination of OAuth 2.0 Token Exchange {{RFC8693}} and Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants {{RFC7521}} to achieve `Identity Chaining`.
A client requests an authorization grant from its own authorization server via a token exchange. Received grant is then presented as an asseration to the authorization server of the protected resource.

## Overview 

The Identity Chaining flow outlined below describes how these specification are used and work together. Two examples in the suffix give more concrete examples. One where a resource server acts as the client and one where an authorization server acts as the client.

~~~~
┌─────────────┐                                    ┌─────────────┐  ┌─────────┐
│Authorization│              ┌────────┐            │Authorization│  │Protected│
│Server       │              │Client  │            │Server       │  │Resource │
│Domain A     │              │Domain A│            │Domain B     │  │Domain B │
└──────┬──────┘              └───┬────┘            └──────┬──────┘  └────┬────┘
       │                         │────┐                   │              │     
       │                         │    │ (A) discover      │              │     
       │                         │<───┘ Authorization     │              │     
       │                         │      Server            │              │     
       │                         │      Domain B          │              │     
       │                         │                        │              │     
       │                         │                        │              │     
       │   (B) exchange token    │                        │              │     
       │   [RFC 8693]            │                        │              │     
       │<─────────────────────────                        │              │     
       │                         │                        │              │     
       │(C) <authorization grant>│                        │              │     
       │ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ >                        │              │     
       │                         │                        │              │     
       │                         │ (D) perform asseration │              │     
       │                         │ [RFC 7521]             │              │     
       │                         │ ──────────────────────>│              │     
       │                         │                        │              │     
       │                         │   (E) <access token>   │              │     
       │                         │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │              │     
       │                         │                        │              │     
       │                         │                  (F) access           │     
       │                         │ ─────────────────────────────────────>│     
       │                         │                        │              │     
       │                         │                        │              │               
~~~~
{: title='Identity Chaining Flow'}

The flow illustrated in Figure 1 shows the step the client needs to perform to access a protected resource in a different trust domain. It includes the following steps:

* (Pre) A client of Domain A needs to access a protected resource of Domain B.

* (A) The client of Domain A needs discovers the authorization server of Domain B. See [Discover Authorization Server](#discover-authorization-server).

* (B) The client exchanges its token to an authorization grant at the authorization server of its own domain. See [Token Exchange](#token-exchange).

* (C) Authorization server of Domain A processes the request and, if federation with Domain B is allowed returns an authorization grant.

* (D) The client uses receive authorization grant to perform an asseration at the authorization server of Domain B. See [Assertion](#assertion).

* (E) Authorization server of Domain B validates the authorization grant and returns an access token.

* (F) The client now possesses an access token to access the protected resource in Domain B.

## Discover Authorization Server

The first step is the discovery of authorization server of domain B (the external authorization server). This specification does not cover the details of such but a client MAY contact the resource unauthenticated and leverage the WWW-Authentication response (see section 3 of {{RFC6750}}), maintain a static mapping or use other means to identify the authorization server.

## Token Exchange

Once the authorization server is identified the client performs a token exchange {{RFC8693}} at its own authorization server in order to receive an authorization grant as specified in section 1.3 of {{RFC6749}}.

### Request

The parameters described in section 2.1 of {{RFC8693}} apply here with the following restrictions:

{:vspace}
subject_token
: REQUIRED Access token of the subject that will be used to call the protected resource in domain B

requested_token_type
: OPTIONAL according to {{RFC8693}}. In the context of this specification this parameter SHOULD NOT be used. See [Authorization grant type](#authorization-grant-type).

resource
: OPTIONAL. URI of authorization server of targeting domain (domain B). REQUIRED if audience is not set.

audience
: OPTIONAL. Well known/logical name of authorization server of targeting domain (domain B). REQUIRED if resource is not set.

scope
: OPTIONAL. Additional scopes to indicate scopes included in returned authorization grant. See [Transcribing claims](#transcribing-claims).

### Resource and audience

Either one of resource or audience is required to indicate for what authorization server the authorization grant should be audienced. The authorization server MUST deny the request if neither is set. An authorization server SHOULD deny the request on an unknown resource/audience. In cases where federation to any authorization server is deliberate unknown resource or scope identifiers MAY be allowed.

<!-- reference on whats the difference? -->

### Processing rules

* The authorization server MAY deny the request due to policy. For instance if federation to the requested domain/authorization server is not permitted. 
<!-- add specific response code -->
* The authorization server MAY add, remove or change claims. See [Transcribing claims](#transcribing-claims).

<!-- add more? -->

### Authorization grant type

The authorization grant format and content is part of a contract between the authorization servers. To achieve a maintainable and flexible systems clients SHOULD NOT request a specific `requested_token_type` during the token exchange and SHOULD NOT require a certain format or parse the authorization grant (for instance in caes of JWT). The `issued_token_type` parameter in the response indicates the type and SHOULD be passed into the asseration request. This allows flexibility for authorization servers to change format and content.

Authorization servers MAY pick an existing grant type such us `urn:ietf:params:oauth:grant-type:jwt-bearer` to indicate a JWT or `urn:ietf:params:oauth:grant-type:saml2-bearer` to indicate SAML. Other grant types MAY be used to indicate other formats.

### Transcribing claims

Authorization servers MAY choose to:

* Remove or hide certain claims in the authorization grant. This could be due to privacy requirements or reduced trust towards domain B. To hide and enclose claims {{SD-JWT}} MAY be used.

* Change existing claims. This MAY be necessary to help domain B to identify the subject. For instance: A user has different identifiers in domain A (johndoe@a.org) and domain B (doe.john). Domain A maintains the mapping and thus needs to replace the subject to allow domain B to correctly identify the subject.

<!-- is reducing scope a valid scenario? Can AS A request "scopes" via grant? -->

* Add additional claims. The peer authorization server of domain B may require additional claims to find the correct subject or to transfer it to the access token for the protected resource.

Clients MAY use the scope parameter to control transcribed claims and thus the claims exposed to domain B. Authorization Servers SHOULD verify that requested scopes are not higher priveleged than the scopes of presented subject_token.

### Response 

All of section 2.2 of {{RFC8693}} applies. In context of this specification the following applies in addtion:

* Returned authorization grant MUST be audienced to the requested authorization server. This corresponds with [RFC 7523 Section 3, Point 3](https://datatracker.ietf.org/doc/html/rfc7523#section-3) and is there to reduce missuse and to prevent clients from presenting their (for other use cases intented) access tokens as asseration.

* Response authorization grant MAY be audienced to multiple Authorization Servers if federation happens to multiple trust boundaries. The authors of this specification reccomended that only one audience is used to prevent Authorization Server B to abuse and present the token to Authorization Server C.

### Example

The example belows shows a command the client invokes to perform token exchange at authorization server of domain A (https://a.org/auth) to receive an authorization grant for authorization server of domain B (https://b.org/auth).

~~~
curl --location 'https://a.org/auth/token' \
--form 'grant_type="urn:ietf:params:oauth:grant-type:token-exchange"' \
--form 'subject_token="ey.."' \
--form 'subject_token_type="urn:ietf:params:oauth:token-type:access_token"' \
--form 'resource="https://b.org/auth"'
~~~

## Assertion

The client uses the received authorization grant from steps B and C as an asseration towards authorization server of domain B.

### Request

Everything in {{RFC7521}} (Section 4.1 in specific) applies. In case of JWT authorization grant all of {{RFC7523}} applies in addition. For the purpose of this specification the following descriptions apply:

{:vspace}
grant_type
: REQUIRED. In context of this specification clients SHOULD use the type identifer returned by the token exchange (`issued_token_type` response). See [authorization grant type](#authorization-grant-type) for more details.

asseration
: REQUIRED. Authorization grant returned by the token exchange (`access_token` response).

scope
: OPTIONAL.

<!-- TODO: How does a client indicate the resource it wants to access? -->

### Processing rules

All of {{RFC7521}} (Section 5.2 in specific) applies. In context of this specification the following rules apply in addition:

* The request MUST be denied presented authorization grant is not audiencd to the authorization server that processes the request
* The authorization server SHOULD deny the request if it is not able to identify the subject
* Due to policy the request MAY be denied (for instance if the federation from domain A is not allowed)

### Transcribing claims

The authorization server MAY leverage claims from the present authorization grant for claim population of the access token. The populated claims SHOULD be namespaced or validated to prevent the injection of invalid claims.

The specifics (such as the format) of returned access token is not part of this specification.

### Response

The authorization server response with an access token as described in section 5.1 of {{RFC6749}}.

### Example

The example belows shows a command the client invokes to present an authorization grant to authorization server (https://b.org/auth) to receive an access token for a protected resource.

~~~
curl --location --request GET 'https://b.org/auth/token' \
--form 'grant_type="urn:ietf:params:oauth:grant-type:jwt-bearer"' \
--form 'asseration="ey..."'
~~~

# IANA Considerations {#IANA}

To be added.

# Security Considerations {#Security}

To be added.

--- back

# Examples

This section contains 2 examples, both are show-casing this specifiction in different environments with specific requirements: 

## Resource server acting as client

Resources servers may act as clients if the following is true:

* Authorization Server B is reachable by the resource server by network and is able to perform the appropiate client authentication (if required).
* The resource server has the ability to determine the authorization server of the protected resource outside its trust domain.

The flow would look like this:

~~~
┌─────────────┐              ┌────────┐            ┌─────────────┐ ┌─────────┐
│Authorization│              │Resource│            │Authorization│ │Protected│
│Server       │              │Server  │            │Server       │ │Resource │
│Domain A     │              │Domain A│            │Domain B     │ │Domain B │
└──────┬──────┘              └───┬────┘            └──────┬──────┘ └────┬────┘
       │                         │                        │             │     
       │                         │     (A) access (unauthenticated)     │     
       │                         │ ────────────────────────────────────>│     
       │                         │                        │             │     
       │                         │      (B) <WWW-Authenticate header>   │     
       │                         │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ── ─ ─│     
       │                         │                        │             │     
       │   (C) exchange token    │                        │             │     
       │   [RFC 8693]            │                        │             │     
       │<─────────────────────────                        │             │     
       │                         │                        │             │     
       │(D) <authorization grant>│                        │             │     
       │ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ >                        │             │     
       │                         │                        │             │     
       │                         │ (E) perform asseration │             │     
       │                         │ [RFC 7521]             │             │     
       │                         │ ──────────────────────>│             │     
       │                         │                        │             │     
       │                         │   (F) <access token>   │             │     
       │                         │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │             │     
       │                         │                        │             │     
       │                         │                  (G) access          │     
       │                         │ ────────────────────────────────────>│     
       │                         │                        │             │     
       │                         │                        │             │     
~~~
{: title='Resource server acting as client'}

The flow contains the following steps:

(A) Resource server of domain A needs to access protected resource in Domain B. It requires an access token to do so which it does not posses. To receive information about the authorization server which protected the resource in domain B it calls the resource unauthenticated.

(B) The protected resource returns the WWW-Authenticate header to indicate its authorization server.

(C) Now, after the resource server has identified the targeting authorization server it request an authorization grant for it at its own authorization server (Domain A). This happens via the token exchange protocol.

(D) If successful, the authorization server returns the authorization grant to the resource server.

(E) The resource server uses received authorization grant to perform an asseration at the authorization server of Domain B.

(F) Authorization server of Domain A uses claims of authorization grant to identify the user and its access. If access is granted an access token is returned.

(G) The resource server uses the access token to access the protected resource at Domain B.

## Authorization server acting as client

Authorization servers may act as clients too. This can be necessary because of following reasons:

* Resource servers MAY not have knowledge of other authorization servers or are MAY not be allowed to posses that knowledge.
* Resource server MAY not have network access to other authorization server
* A strict access control on resources outside the trust domain is required and enforced by authorization servers.
* Authorization servers requires client authentication and managing clients for resource servers outside of the trust domain is not intended.

The flow when authorization servers act as client would look like this:

~~~
┌────────┐          ┌─────────────┐              ┌─────────────┐ ┌─────────┐
│Resource│          │Authorization│              │Authorization│ │Protected│
│Server  │          │Server       │              │Server       │ │Resource │
│Domain A│          │Domain A     │              │Domain B     │ │Domain B │
└───┬────┘          └──────┬──────┘              └──────┬──────┘ └────┬────┘
    │ (A) request token for│                            │             │     
    │ protected resource   │                            │             │     
    │ in domain B.         │                            │             │     
    │ ─────────────────────>                            │             │     
    │                      │                            │             │     
    │                      │────┐                       │             │     
    │                      │    │ (B) determine                       │     
    │                      │<───┘ authorization server B              │     
    │                      │                                          │     
    │                      │                                          │     
    │                      │────┐                                     │     
    │                      │    │ (C) issue                           │     
    │                      │<───┘ authorization grant                 │     
    │                      │      ("internal token exchange")         │     
    │                      │                                          │     
    │                      │                            │             │     
    │                      │     (D) perform asseration │             │     
    │                      │     [RFC 7521]             │             │     
    │                      │ ──────────────────────────>              │     
    │                      │                            │             │     
    │                      │       (E) <access token>   │             │     
    │                      │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─               │     
    │                      │                            │             │     
    │  (F) <access token>  │                            │             │     
    │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                            │             │     
    │                      │                            │             │     
    │                      │           (G) access       │             │     
    │ ───────────────────────────────────────────────────────────────>     
    │                      │                            │             │     
    │                      │                            │             │     
~~~
{: title='Authorization server acting as client'}

The flow contains the following steps:

(A) Resource server of Domain A request a token for protected resource in Domain B. This specifiction does not cover this step in particular. A profile of Token Exchange {{RFC8693}} may be used.

(B) The authorization server (of Domain A) determines the authorization server (of Domain B). This could have been passed by the client, is statically maintained or dynamically resolved.

(C) Once the authorization server is determined an authorization grant is issued internally. This reflects to [Token exchange](#token-exchange) of this specification and can be seen as an "internal token exchange".

(D) The issued authorization grant is used to perform an asseration at the authorization server of Domain B. This asseration happens between the authorization servers and authorization server A may be required to provide client authentication too while doing so.

(E) Authorization server of Domain B returns an access token to access the protected resource.

(F) Authorization server of Domain A returns that access token to the resource server.

(G) The resource server uses received access token to access the protected resource.

# Acknowledgements {#Acknowledgements}
{: numbered="false"}

<!-- 
OPEN TOPICS:
- P1: How does a client indicate the protected resource to AS B?

-->
