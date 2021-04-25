# Trusted Credentials

## Motivation

Verifiable credentials make claims about a subject. While it is possible for a verifier to deterministically check that a credential actually came from its listed issuer without modifications, the decision of whether that issuer is _relevant_ for the claims is subjective.

For example, a police officer in Germany might verify that a driving license from Canada is genuine, but that doesn't mean they will accept the Canadian government as a relevant authority and let you drive. That decision is up to the police and based e.g. on international agreements between countries.

In some cases, verifiers' criteria are as simple as recognizing a set of authorities for certain types of claims or credentials. In other cases, the decision involves delegation and roots of authority. Such a setup is similar to the Public Key Infrastructure (PKI) for X.509 certificates, but with different chains of trust depending on the type of information – the authorities for academic credentials are not the same as for citizenship or banking information.

## Definitions and concepts

### Authority

An authority for a type of claim (e.g. identity document number, phone number, email address) is an entity that is trusted by a verifier to issue reliable claims of that type.

Being or not an authority for a given type of claim depends on each verifier: a same entity is an authority to some verifiers but not to others.

### Trusted claim

A claim is trusted by a verifier if either:

- The verifier considers the issuer of the containing credential to be an authority for that claim (_delegated trust_).
- Or the verifier decides to trust the claim directly, based on their own rules (_diret trust_).

### Delegation: depths of authority

Some entities, such as governments, have the authority to issue claims themselves but choose to delegate that authority to other entities.

**Examples:**

- A car reseller may attest to a car's functioning state but may not delegate that authority to another entity (authority with delegation depth 0).
- A car brand may give authority to car resellers to attest to a car's functioning state (delegation depth 1).
- A university may issue diplomas but may not delegate that authority (delegation depth 0).
- The Ministry of Education of a country may give authority to universities to issue diplomas (delegation depth 1).
- The government of a country may give authority to the Ministry of Education to give that authority to universities (delegation depth 2).

## Big picture

![chainoftrust.png](img/chainoftrust.png)

## Technical specification

### Schema

The following schema is used in a Verifiable Credential issued by "entity A" to say that "entity B" is authoritative for a given claim type. See [schema.org issue #2854](https://github.com/schemaorg/schemaorg/issues/2854) for the latest discussion about the vocabulary.

```json
  "issuer": "did:xxx:entityA",
  "credentialSubject": {
    "id": "did:xxx:entityB"
    "hasIssuingAuthority": {
      "@type": "IssuerScope",
      "issuerFor": "http://schema.org/driverLicense", # The type of claim the entity is authoritative for
      "delegationDepth": 2 # Number of delegation hops, defaults to 0
    }
  }
```

### Trusting claims

A claim can be trusted either directly or through delegated trust. The sections below detail how trust is determined in either case. Delegated trust is defined recursively, with direct trust being the exit case of the recursivity.

#### Delegated trust

Delegated trust applies when the issuer of the credential is considered an authority for a claim.

As an example, let's say the verifier has access to a credential such as the one in the example from the "Schema" section above. If the verifier trusts (directly or through delegation) that `hasIssuingAuthority` claim, then they will delegate their trust to credential issuer "entity B" for the following types of claims:

 - Any `driverLicense` claim.
 - Any `hasIssuingAuthority` claim with `delegationDepth` strictly lower than 2, and about `driverLicense` claims.

Example credentials for either case:

```json
  "issuer": "did:xxx:entityB",
  "credentialSubject": {
    "id": "did:xxx:entityC",
    "driverLicense": { ... }
  }
```

```json
  "issuer": "did:xxx:entityB",
  "credentialSubject": {
    "id": "did:xxx:entityC",
    "hasIssuingAuthority": {
      "@type": "IssuerScope",
      "issuerFor": "http://schema.org/driverLicense",
      "delegationDepth": 1
    }
  }
```

Example scenario:

  - Recruiter X receives a credential claiming that John Doe has a Doctorate in Rocket Science (modelled as "diploma").
    - That credential is issued by University of the North.
  - University of the North, in a separate credential, claims to be an authority for claims of type "diploma" with default delegation depth 0.
    - That second credential was issued by Ministry of Education.
  - Ministry of Education, in a separate credential, claims to be an authority for "diploma" claims, with delegation depth 1.
    - That third credential is signed by Government of Country X.
  - Government of Country X, in a separate credential, claims to be authoritative for claim "diploma" with delegation depth 3.
    - That third credential is directly trusted by the verifier (see below).
  - As a result, Recruiter X trusts John Doe's diploma through delegated trust.


#### Direct trust

A claim is trusted _directly_ when the verifier makes the decision based on its own (business, regulatory, etc.) rules. This is the simplest case.

Example scenarios:

1. An identity wallet may allow the user to manually trust a specific credential.
2. A verifier may choose to accept self-issued credentials (i.e. the subject is the issuer) for some claims.
3. _Big Buck Bank_ chooses to trust directly a specific financial institution as an authority for credit score claims.
4. _Recruiter X_ knows the DIDs of recognized universities and decides to trust any diplomas issued by those DIDs, for a specific range of issuance date.
5. _Ask Y_ chooses to trust a specific public institution for `hasIssuingAuthority` claims and a given delegation depth, making that issuer a "Root authority" in a chain of trust.

## Notes

### Transitivity of trust delegation: friend of a friend of a friend of...

This specification defines delegated trust in a recursive way, with direct trust being the exit condition. However, to prevent infinite loops and to avoid ridiculously long trust chains, verifiers may decide to put a practical limit to how many hops they support.

### Distribution of authority credentials

This data model relies on the verifier's access to "trust credentials" in addition to the credential of first interest. This is similar to SSL's requirement for the server to distribute the complete chain of trust during handshake. Although this data model doesn't define a way for the holder to distribute the relevant trust credentials, a good practice might be to include all relevant credentials in a Verifiable Presentation. That being said, depending on the context the holder can assume that the verifier already trusts some of the involved authorities and thus avoid "stating the obvious".

### Value constraints

This model doesn't place any constraints on the value of the claims. For example, University of the North may not be an authority for Doctorates in Rocket Science but only for Masters in Literature.

However, that limitation shouldn't be a problem thanks to the trust model. If an entity abuses their authority and starts signing certificates that they shouldn't, or otherwise fails to demonstrate that they're following a rigorous issuance process, they will take the risk of losing their status as an authority. Note the similarity with the inclusion of Root CAs by browsers in traditional PKI.

### Relation with eIDAS

Verifiable Credentials may contain a `levelOfAssurance` attribute as part of their metadata (i.e. at the same level as `credentialSubject`). The value of that property indicates how reliable the claims contained in the credential are.

The `hasIssuingAuthority` claims discussed in this specification play nicely with a credential's level of assurance, because the level of assurance of such credentials indicates the level of assurance given to that issuer by a higher-level authority.

#### Example 1: level of assurance for a normal credential

The credential below claims the name of subject `did:xxx:abc` to be John Doe, with a level of assurance "High" set by issuer `did:xxx:def`.

```json
{
  "@type": "VerifiableCredential",
  "credentialSubject": {
    "@context": "http://schema.org/",
    "@id": "did:xxx:abc",
    "name": "John Doe"
  },
  "levelOfAssurance": "High",
  "issuer": "did:xxx:def"
}
```

#### Example 2: level of assurance for an authority

The credential below claims that subject `did:xxx:def` (issuer of the credential above) is an authority for claims of type `name`, with a level of assurance "High" set by higher-level authority `did:xxx:ghi`.

```json
{
  "@type": "VerifiableCredential",
  "credentialSubject": {
    "@context": "http://schema.kaytrust.id/",
    "@id": "did:xxx:def",
    "hasIssuingAuthority": {
      "@type": "IssuerScope",
      "issuerFor": "http://schema.org/name"
    }
  },
  "levelOfAssurance": "High",
  "issuer": "did:xxx:ghi"
}
```

If the verifier trusts `did:xxx:ghi` with that level of assurance, then they will also trust John Doe's credential with the same level of assurance, per eiDAS rules.
