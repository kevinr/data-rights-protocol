# Introduction

In the last DRP Technical working group meeting someone raised a request for a sort of "implementation guide" for the DRP specification, and I took an attempt at outlining the "minimum viable product" components for both the Authorized Agent and PIP interfaces and tried to put to words some of the design philosophy which fed in to the specification's drafting. I plan on fleshing this out to be more legible this week since a lot of this is short hand, but it gives you a sort of broad understanding of the technical complexity involved and we wanted to share it out.

The **primary consideration** of DRP as a technical solution is that it is a standard interchange format for Data Subject Requests.

What are the minimal logical components of a DRP stack? What would it take to implement the DRP protocol?

We should be careful when and consider not crafting specific user-flows in to the protocol to allow for the development of a healthy community of Authorized Agents and Privacy Infrastructure Providers, comprehensive and niche. This advice maybe flies in the face of the `needs_user_verification` semantics as a theoretical manhole-cover in to a world of cumbersome identity verification stages.

DRP doesn't aim to define the shape of the Privacy Infrastructure Provider's product as far as their interaction with Covered Businesses entail, nor the consumer flow of an Authorized Agent. The goal of DRP is simply to make sure the AA and PIP and CB are speaking the same language and can trust their sources. [xxx] expand this to talk about how CBs would/could/should ideally be able to "turn on DRP" without writing code or integrations.

the Covered Business is the one who is ultimately operating a product the PIP is responsible for enacting the request. It could be a CSV download of DSRs and an SFTP upload provided to the business, or it could be a full cloud automation suite, or it could send emails to paralegals. Be innovative!

a covered business can implement DRP on their own and work to be included in the DRP implementer's group.

# A Minimal Privacy Infrastructure Provider

## HTTP API

### Implement `/exercise`

Store the inbound request in the data store, notify the covered business, plug request in to existing Data Subject Management product solution.

### Implement `/revoke`

end users have the ability to revoke or cancel a request that is in an non-terminal state. Plug this in to your existing workflow engine

### Implement Authorized Agent status callback

Authorized agents are encouraged to run a web server which is capable of being POST'd a Data Rights Status JSON object, and PIPs are encouraged to notify one if a Data Rights Request includes a `status_callback` key.

### Implement `/status`

Backend-less authorized agents request the status of a DSR using this API to refresh the user agent. This status is described in the specification section 3.02.

### Probably some `need_user_verification` product

A covered business may want to bring their own account verification system, but a PIP could integrate a form-builder in to their product and manage the `user_verification_url`.

## Workflow Management

### The PIP "owns" the status of the request

a covered business is basically moving the data request through a "unacknowledged → verifying → in-progress → end state" workflow powered by the PIP's DSR tools, and the API reflects this in a standardized fashion.

### What information does the covered business have access to?

-   information about the authorized agent (provided out of band, extracted from JWT iss as that is signed, or API authorization key)
-   identity attributes (there is a level of assurance/trust that has to be assured out of band. 🤞 for eKYC), ✌verified✌ attributes in the meantime
-   legal regime, actions requested
-   important dates (45 days after ack for CCPA)
-   (??)


### What actions can a CB be expected to take?

-   update a request's state with information about the status of a request in progress
-   request information or verification by providing a URL
-   a covered business could be running an OIDC server they want consumers to authenticate against
-   in theory there may be ways for the Covered Business to trust a token generated by popular third-party OIDC providers, provided by the AA?
-   (??)

### What is expected of Covered Businesses?

[xxx incomplete xxx]

-   bring their identity provider or the IDps/attributes they trust
-   interop is not merely technical, it's business and legal too
-   how do we encode those things so that onboarding could be "automated"
-   (??)

### Status Tracking

[xxx incomplete xxx] ties in to reporting rules in eventual System Rules 

## OAuth2 client token handshake

The current protocol specifies only a limited authentication method for the API, but this cannot survive forever.

It's a reasonable assumption that sooner or later a Privacy Infrastructure Provider will need to act as an OAuth2 server which AAs get API keys from to sign/authenticate requests.

# A Minimal Authorized Agent

An authorized agent is responsible for verifying the identity attributes of a user, presenting them with one or more covered businesses, their supported actions, and to submit requests to those covered businesses with those identity attributes and actions embedded in to them using a few HTTP endpoints described in the PIP section below.

## A bare-bones Authorized Agent server architecture

-   a server implementing [non-DRP API]/forms that can generate data rights requests
    -   collect identity attributes for the user in one page
    -   select rights to exercise, regulatory regime to submit under, pick a PIP/CB to send them to, etc
-   An HTTP client within the server that can be triggered to assemble and send those rights requests
    -   assembles DRR from the identity attributes created in signup, and the rights etc in the second form, signs JWT, etc
    -   `POST /exercise` to the chosen CB
-   a server endpoint implementing status callback URL
-   Status tracking of those in-flight requests with polling `GET /status` and updating it on that first status callback URL
-   as yet unclear `need_user_verification` ([here](https://github.com/consumer-reports-digital-lab/data-rights-protocol/blob/main/data-rights-protocol.md#3021-need_user_verification-state-flow-semantics)) flow implementation, but that's probably just some crafty HTTP redirects

## User Identity Verification system

`JWT` tokens are signed by the submitting Authorized Agent but no real assurances can be attested right now; in theory eKYC and even self-sovereign "decentralized identity" solutions could bring real value to this system in the future.

Until then, an Authorized Agent is responsible for verifying the identity of a consumer, and to use "verified" claims of email, address, or phone number only if they've been verified... Trust network, liability stuff, non-technical guard-rails will protect the network until truly decentralized solutions can arrive.

## User Agent

DRP is not prescriptive here, at all

but right now the protocol and especially the push-based call-back Status Tracking implies that there is a backend services

### user verification flows

the user is expected to be able to go through a web-based verification flow [described in 3.02.1](https://github.com/consumer-reports-digital-lab/data-rights-protocol/blob/main/data-rights-protocol.md#3021-need_user_verification-state-flow-semantics), but basically involves redirecting the user to a PIP-specified URL with some extra URL parameters attached to it.

This only has been designed in-theory and implementation may shift wildly.

### Auto-discovery

DRP specifies that Covered Businesses choosing to implement DRP can signal this through the use of a [.well-known resource](https://github.com/consumer-reports-digital-lab/data-rights-protocol/blob/main/data-rights-protocol.md#201-get-well-knowndata-rightsjson-data-rights-discovery-endpoint), the Data Rights Discovery endpoint. The operative mode of this network is going to be a closed group of known implementers for a while, so I don't expect this to be necessary for quite a while, but is an important part of the long-term growth of this API.

[xxx NEXT add a bit talking about how the closed network will fashion]

### API Keys and Security Considerations

[xxx draft this]

## Generating Identity Tokens

Identity tokens need to be signed by a certificate that is never in un-trusted hands, as the certificate the token is signed against is the lynchpin of the trust network. [3.04 of the protocol spec describes this in detail](https://github.com/consumer-reports-digital-lab/data-rights-protocol/blob/main/data-rights-protocol.md#304-schema-identity-encapsulation).

## Status Tracking

While there is a way to fetch the status for a data rights request from the PIP, PIPs are expected to support pushing request statuses to the Authorized Agent through an HTTP POST documented [in 2.04 of the protocol specification](https://github.com/consumer-reports-digital-lab/data-rights-protocol/blob/main/data-rights-protocol.md#204-post-status_callback-data-rights-status-callback-endpoint).

## Batching Backend

We have been careful to avoid thinking about batching but in our prototyping phase it's possible that multiple requests may be submitted or queried in bulk.

## OIDC client ... fetching JWT identity tokens from covered business identity provider

In a perfect world, JWTs provided by the Covered Business's identity provider could be used to attest

[xxx need to develop this still]

# "Known Unknowns"

This guide provides some very high level information about what goes in to a DRP implementation, as far as the author is aware, but there is a fair amount of work that needs to be input before a 1.0 protocol and public release.

There are some situations in the specification which we know are under-specified, which are unknown, which are hypothetical. These sorts of things should be considered carefully during implementation. During prototyping, end-to-end testing, implementation, these sorts of concerns should take the backseat to getting requests flowing between actors, but there will be a period during the evaluation of prototypes where we'll try to integrate our learning + experience back into the spec.

-   structure/composition of DRR contents
    -   request batching etc
    -   hints for subsidiaries, unique relationships
    -   how to represent narrow access/deletion/correction requests?
-   what is the `data-rights.json` equivalent for Authorized Agents?
-   OIDC
-   OAuth2/API keys
-   JWT security/encryption/etc

# Technical/BLG overlaps

Not all of the things listed in this document are purely technical ... things like API key management, well-known resources, token validation, etc will need to be coordinated with governance onboarding and the operation of the trust network.

The governance organization will be responsible for ensuring the actors within the trust network have the information needed to trust other parties, in essence

AAs and PIPs need to interchange

-   JWT signing key
-   API tokens
-   level of assurance, agent registration, etc
-   power of attorney documentation clearing house?

This will probably be done through a static machine-legible directory managed by the DRP governance organization.
