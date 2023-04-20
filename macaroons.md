# Macaroon Minting & Verification

## Overview

In this chapter, we outline the specification of the Macaroon component of a Lightning API key \(L402\). To recap, an L402 is composed of a Macaroon, which specifies the allowed capabilities of the credential, and a preimage, which serves as the credential’s proof of payment. Macaroons are perfect candidates for L402 as they are tamper-proof, support easy rotation, have attributes, and can even be further attenuated in order for applications that integrate an L402 enabled service to delegate any additional capabilities. We’ll cover how Macaroons are created, attenuated, and verified as part of L402. This chapter will require an understanding of how Macaroons work and how they are useful in the context of authentication. It may be useful to skim the [introductory research paper on Macaroons](https://research.google/pubs/pub41892/).

## Minting Macaroons

Macaroons have three basic components: a public identifier that maps to a root key, a list of caveats \(which is how attenuation is achieved\), and a signature. Minting a new Macaroon only requires the public identifier and the root key it corresponds to.

Each Macaroon must have its own cryptographically random root key that must not be disclosed to prevent forgeability of Macaroons. Each root key consists of 32 bytes, which provides a reasonable tradeoff between entropy and size. Root keys are essential to the verification of Macaroons, so they must be stored securely and reliably.

### Macaroon Identifier

The key mapping to the secret is the SHA-256 hash of the Macaroon’s public identifier, which contains static information about the Macaroon itself. In the initial construction, the public identifier should include the following in the listed order:

> Note: All data specified here are encoded in big-endian unless otherwise stated.

* **Version** - A version allows for an iterative Macaroon design. It must be encoded in 2 bytes unsigned integer.
* **Payment Hash** - A payment hash links an invoice’s payment request to the Macaroon. Once the payment request is fulfilled, the payer receives its corresponding preimage as proof of payment. This proof can then be provided along with the Macaroon to ensure an L402 has been paid for without checking whether the invoice has been fulfilled.
* **User Identifier** - A unique user identifier allows services to track users across distinct Macaroons serving useful in the context of service level metering. A user identifier of 32 random bytes is used instead of the mMcaroon’s identifier because the latter can be revoked, e.g., in the case of a service tier upgrade.

## Attenuation Through Caveats

Caveats are predicates that restrict a Macaroon’s authority, as well as the context in which it may be successfully used. When verifying a Macaroon, each caveat predicate must be evaluated and hold true. Due to their flexibility, additional context found within the request to a service may be necessary for proper evaluation.

Caveats of a version 0 Macaroon are represented as key-value pairs encoded as a string where the key and value are separated by a single `=` character. The key uniquely identifies a caveat and provides context as to how it should be evaluated, while the value provides context for the evaluation itself. There aren't any further restrictions on how caveats should be formed, but Lightning Labs services will mostly impose three types of caveats which are covered below.

### Target Services

The target services the Macaroon is allowed to access is represented as a caveat. The caveat key is `services`, while the value consists of a comma-separated list of target services. Each target service is composed of a two-tuple consisting of the service name and its tier. Tiers are service specific and must start from 0, which serves as the base tier. If a specific service tier has its capabilities and/or constraints updated, there needs to be a way to detect when a Macaroon of the same tier with the now outdated capabilities and/or constraints is being used. By committing to the service tier, it is possible to detect such cases and seamlessly upgrade the stale Macaroon \(assuming it is valid\) by revoking it and minting a new Macaroon with the newer capabilities and/or constraints.

When verifying this caveat, if a Macaroon is attempting to access a service that it does not commit to, then it should be considered invalid. If multiple services caveats exist, then verification should ensure each occurrence of the caveat restricts more access than the previous.

### Service Capabilities

Each service can further be restricted by the capabilities the service provides and these are also represented as another caveat. These caveats have a key ending in `_capabilities` that is prefixed with the service name, while the value consists of a comma-separated list of the allowed service capabilities. This type of caveat allows certain Macaroons to only have access to a subset of a service's features.

If a capabilities caveat for a particular service is not present, then the Macaroon is able to access any capability of the service. If multiple capabilities caveats exist for the same service, then verification should ensure each occurrence of the caveat restricts more access than the previous.

### Service Constraints

Each service can define its own set of constraint caveats for a given tier to further restrict the capabilities of a Macaroon. Each constraint takes the form of a caveat, where the key is prefixed with the service capability it applies to, and the remainder of the key includes context for the service on how to evaluate the constraint. The caveat value specifies the parameters for the constraint evaluation.

If multiple caveats of the same constraint are found within a Macaroon, then verification should ensure each occurrence of the constraint restricts more access than the previous.

As an example, a base tier Lightning Loop Macaroon with access to a 2 BTC monthly volume for Loop Out and unlimited volume for Loop In would look like:

```text
identifier:
    version = 0
    user_id = fed74b3ef24820f440601eff5bfb42bef4d615c4948cec8aca3cb15bd23f1013
    payment_hash = 163102a9c88fa4ec9ac9937b6f070bc3e27249a81ad7a05f398ac5d7d16f7bea
caveats:
    services = lightning_loop:0
    lightning_loop_capabilities = loop_out,loop_in
    loop_out_monthly_volume_sats = 200000000
```

Due to the flexibility of the design, a Macaroon holder is able to further attenuate a Macaroon if they wish to share it with a third party under more restrictive permissions. Following the example above, the Macaroon holder can restrict the Macaroon’s capabilities to only allow access to Loop In \(and not Loop Out\) with a monthly volume of 1 BTC by adding the following caveats:

```text
lightning_loop_capabilities = loop_in
loop_in_monthly_volume_sats = 100000000
```

## Macaroon Verification

Verifying a Macaroon consists of a three step process and involves two parties: the minter of the Macaroon and the authorizer, which is the service the Macaroon targets. The minter of the Macaroon performs signature checks to ensure the Macaroon is valid, while the authorizer ensures the Macaroon has the required permissions to access the service.

The first step ensures a Macaroon was minted by the minter and it has not been tampered with. This is done by computing the HMAC chain of the Macaroon, starting from its root key and identifier, and including any further attenuation through its caveats to arrive at the terminal signature of the Macaroon. If these do not match, the Macaroon is invalid. If there isn’t a valid root key corresponding to the Macaroon, it is also considered invalid.

The second step ensures a Macaroon has provided a valid proof of payment \(preimage\) and is performed by the minter as well. Since Macaroons commit to the payment hash of an invoice, this is a trivial step.

The final step ensures a Macaroon is authorized to access a service. This is done by ensuring the service-specific caveat predicates of a Macaroon hold true for the service being accessed. If only one of these caveats doesn’t hold true, then the Macaroon is invalid. In the case of an unknown caveat, its evaluation must be skipped by the authorizer as the Macaroon holder can further attenuate the Macaroon for other applications.

## Macaroon Revocation

To prevent abusers of a Macaroon-based authenticated service, a Macaroon should be able to be revoked. This can be achieved by having the minter remove the Macaroon’s corresponding root key. By doing so, the minter will never be able to verify the signature of a revoked Macaroon, ensuring it will never reach its targeted service.

Revocation also serves useful when performing a service tier upgrade on a Macaroon. The prior Macaroon is revoked to ensure only the upgraded one can be used going forward.

