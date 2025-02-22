# Trust Store and Trust Policy Specification

Notary Project currently supports X.509 based PKI and identities, and uses a trust store and trust policy to determine if a signed artifact is considered authentic.

This document consists of the following sections:

- **[Trust Store](#trust-store)**: Contains a set of trusted identities through which trust is derived for the rest of the system. For X.509 PKI, the trust store typically contains a set of root certificates.
- **[Trust Policy](#trust-policy)**: A policy language which indicates which identities are trusted to produce artifacts. Both trust store and trust policy need to be configured by users/administrators before artifact signature can be evaluated. Notation supports different trust policies for verifying different types of artifacts like signed OCI images and arbitrary blobs, etc.
- **[Signature Verification](#signature-verification)**: Describes how signatures are evaluated using the policy, to determine if a signed artifact is authentic. This section is meant for implementations of the [Notary Project verification specification](./signing-and-verification-workflow.md).

Other types of identities and trust models may be supported in future, which may introduce other constructs/policy elements to support signature evaluation.

## Scenario

All examples use the actors defined in the Notary Project [scenarios](../requirements/scenarios.md#scenario-0-build-publish-consume-enforce-policy-deploy)

- Wabbit Networks company builds, signs and distributes their `net-monitor` software.
- ACME Rockets consumes the `net-monitor` software from Wabbit Networks.

## Trust Store

Contains a set of trusted identities through which trust is derived for the rest of the system. For X.509 PKI, the trust store typically contains root certificates.

- The Notary Project trust store consists of multiple named collections of certificates, called named stores.
- Following certificate formats are supported - Files with extension .pem, .crt and .cer, the files are expected to contain certificate(s) in DER (binary) format or PEM format (base-64 encoded DER).
- The trust store is a directory location, under which each sub directory is considered a named store, that contains zero or more certificates. The name of this sub directory is used to reference the specific store in trust policy.
- Symlinks are not supported for the named store directories or certificate files. Implementation MUST validate that the named store directory or certificate files are not symlinks, and fail if it does not meet this condition.
- Certificates in a trust store are root certificates. Placing intermediate certificates in the trust store is not recommended as this is a form of certificate pinning that can break signature verification unexpectedly anytime the intermediate certificate is rotated.

The Notary Project uses following directory structure to represent the trust store. The example shows named stores `acme-rockets` and `wabbit-networks`, which are subsequently references in the trust policy. Without this reference, presence of a named store and certificates in it does not confer trust automatically to the named store. The trust store is configured ahead of verification time, by an out of band mechanism that is beyond the scope of this document. Different entities and organizations have their own processes and policies to configure and distribute trust stores.

```text
$XDG_CONFIG_HOME/notation/trust-store
    /x509
        /ca
            /acme-rockets
                cert1.pem
                cert2.pem
                  /sub-dir       # sub directory is ignored
                    cert-3.pem   # certs under sub directory is ignored
            /acme-rockets-ca2
                cert4.pem
            /wabbit-networks
                cert5.crt
        /signingAuthority
            /acme-network-sa
                cert10.pem
        /tsa
            /publicly-trusted-tsa
                tsa-root.pem
```

The Trust store currently supports three kinds of identities, additional identities may be supported in future:

- **Certificates**: The `x509/ca` trust store contains named stores that contain Certificate Authority (CA) root certificates.
- **SigningAuthority Certificate**: The `x509/signingAuthority` trust store contains named stores that contain Signing Authority's root certificates.
- **Timestamping Certificates**: The `x509/tsa` trust store contains named stores that contain Timestamping Authority (TSA) root certificates.

Any additional sub directories under a named store and certificates in it are ignored. **NOTE**: Implementation SHOULD warn if it finds sub directories with certificates under a named store, to help diagnose misconfigured store.

## Trust Policy

Users who consume signed artifacts from OCI registries, or signed arbitrary blobs stored on file system, use trust policies to specify trusted identities that signed the artifacts, and level of signature verification to enforce. Trust policy files are JSON documents.

> [!NOTE]
> Users are required to setup separate trust policy files for verifying different types of signed artifacts. Notation expects OCI trust policy file to be placed at `{NOTATION_CONFIG}/trustpolicy.oci.json`. For backward compatibility reasons, Notation accepts `{NOTATION_CONFIG}/trustpolicy.json` as OCI trust policy file as well. For arbitrary blobs, Notation expects trust policy file to be placed at `{NOTATION_CONFIG}/trustpolicy.blob.json`.

### Trust Policy Properties

#### OCI Trust Policy

##### Version 1.0

- **`version`**(*string*): This REQUIRED property is the version of the trust policy. This MUST be `1.0` at the moment.
- **`trustPolicies`**(*string-array of objects map*): This REQUIRED property represents a collection of trust policies.
  - **`name`**(*string*): This REQUIRED property represents the name of the trust policy.
  - **`registryScopes`**(*array of strings*): This REQUIRED property determines which trust policy is applicable for the given artifact.
    The scope field supports filtering based on fully qualified repository URI `${registry-name}/${namespace}/${repository-name}`.
    For more information, see [trust policy constraints](#oci-trust-policy-constraints) section.
  - **`signatureVerification`**(*object*): This REQUIRED property dictates how signature verification is performed.
  An *object* that specifies a predefined verification level, with an option to override the Notary Project trust policy defined verification level if user wants to specify a [custom verification level](#custom-verification-level).
    - **`level`**(*string*): A REQUIRED property that specifies the verification level, supported values are `strict`, `permissive`, `audit` and `skip`. Detailed explanation of each level is present [here](#signatureverification-details).
    - **`override`**(*map of string-string*): This OPTIONAL map is used to specify a [custom verification level](#custom-verification-level).
    - **`verifyTimestamp`**(*string*): An OPTIONAL property that specifies timestamp verification configuration. Supported values are `always` and `afterCertExpiry`. The default value is `always`, which means always verify timestamp countersignature. `afterCertExpiry` means only verify timestamp countersignature if at least one certificate in the signing certificate chain has expired at the time of verification.
  - **`trustStores`**(*array of string*): This REQUIRED property specifies a set of one or more named trust stores, each of which contain the trusted roots against which signatures are verified. Each named trust store uses the format `{trust-store-type}:{named-store}`. Currently supported values for `trust-store-type` are `ca`, `signingAuthority` and `tsa`. To enable timestamp verification, type `tsa` MUST be configured.
  - **`trustedIdentities`**(*array of strings*): This REQUIRED property specifies a set of identities that the user trusts. For X.509 PKI, it supports list of elements/attributes of the signing certificate's subject. For more information, see [identities constraints](#trusted-identities-constraints) section. A value `*` is supported if user trusts any identity (signing certificate) issued by the CA(s) in `trustStore`. This field only contains trusted identities issued by CA(s) or Signing Authorities. 
  
#### Blob Trust Policy

##### Version 1.0

- **`version`**(*string*): This REQUIRED property is the version of the trust policy. This MUST be `1.0` at the moment.
- **`trustPolicies`**(*string-array of objects map*): This REQUIRED property represents a collection of trust policies.
  - **`name`**(*string*): This REQUIRED property represents the name of the trust policy. This will be used to select the policy to verify a signed blob.
  - **`signatureVerification`**(*object*): This REQUIRED property dictates how signature verification is performed.
  An *object* that specifies a predefined verification level, with an option to override the Notary Project trust policy defined verification level if user wants to specify a [custom verification level](#custom-verification-level).
    - **`level`**(*string*): A REQUIRED property that specifies the verification level, supported values are `strict`, `permissive`, `audit` and `skip`. Detailed explanation of each level is present [here](#signatureverification-details).
    - **`override`**(*map of string-string*): This OPTIONAL map is used to specify a [custom verification level](#custom-verification-level).
    - **`verifyTimestamp`**(*string*): An OPTIONAL property that specifies timestamp verification configuration. Supported values are `always` and `afterCertExpiry`. The default value is `always`, which means always verify timestamp countersignature. `afterCertExpiry` means only verify timestamp countersignature if at least one certificate in the signing certificate chain has expired at the time of verification.
  - **`globalPolicy`**(*boolean*): This OPTIONAL property flags the policy as the global trust policy if set to `true`. This policy will be used for verification if the user does not select any policy by its name during verification.
  - **`trustStores`**(*array of string*): This REQUIRED property specifies a set of one or more named trust stores, each of which contain the trusted roots against which signatures are verified. Each named trust store uses the format `{trust-store-type}:{named-store}`. Currently supported values for `trust-store-type` are `ca`, `signingAuthority` and `tsa`. To enable timestamp verification, type `tsa` MUST be configured.
  - **`trustedIdentities`**(*array of strings*): This REQUIRED property specifies a set of identities that the user trusts. For X.509 PKI, it supports list of elements/attributes of the signing certificate's subject. For more information, see [identities constraints](#trusted-identities-constraints) section. A value `*` is supported if user trusts any identity (signing certificate) issued by the CA(s) in `trustStore`. This field only contains trusted identities issued by CA(s) or Signing Authorities. 

### Trust Policy Examples

#### Trust Policy for OCI artifacts

1. Trust policy for a simple scenario where ACME Rockets consumes only the OCI artifacts signed by their own CI/CD. Any third party artifacts consumed by ACME Rockets are also signed and vetted by their CI/CD.

```jsonc
{
    "version": "1.0",
    "trustPolicies": [
        {
            // Policy for all artifacts, from any OCI registry location.
            "name": "wabbit-networks-images",   // Name of the policy.
            "registryScopes": [ "*" ],          // The registry artifacts to which the policy applies.
            "signatureVerification": {          // The level of verification - strict, permissive, audit, skip.
              "level" : "audit"
            },
            "trustStores": ["ca:acme-rockets"], // The trust stores that contains the X.509 trusted roots.
            "trustedIdentities": [              // Identities that are trusted to sign the artifact.
              "x509.subject: C=US, ST=WA, L=Seattle, O=acme-rockets.io, OU=Finance, CN=SecureBuilder"
            ]
        }
    ]
}
```

2.Trust policy for a scenario where ACME Rockets consumes some OCI artifacts signed by Wabbit Networks and some signed by ACME Rockets.

```jsonc
{
    "version": "1.0",
    "trustPolicies": [
        {
            // Policy for set of artifacts signed by Wabbit Networks
            // that are pulled from ACME Rockets repository
            "name": "wabbit-networks-images",
            "registryScopes": [
              "registry.acme-rockets.io/software/net-monitor",
              "registry.acme-rockets.io/software/net-logger"
            ],
            "signatureVerification": {
              "level" : "strict" 
            },
            "trustStores": ["ca:wabbit-networks"],
            "trustedIdentities": [ 
              "x509.subject: C=US, ST=WA, L=Seattle, O=wabbit-networks.io, OU=Security Tools"
            ]
        },
        {
            // Exception policy for a single unsigned OCI artifact pulled from
            // Wabbit Networks repository
            "name": "unsigned-image",
            "registryScopes": [ "registry.wabbit-networks.io/software/unsigned/net-utils" ],
            "signatureVerification": {
              "level" : "skip" 
            }
        },
        {
            // Policy that uses custom verification level to relax the strict verification.
            // It logs expiry and skips revocation check for a specific OCI artifact.
            "name": "use-expired-image",
            "registryScopes": [ "registry.acme-rockets.io/software/legacy/metrics" ],
            "signatureVerification": {
              "level" : "strict",
              "override" : {
                "expiry" : "log",
                "revocation" : "skip"
              }
            },
            "trustStores": ["ca:acme-rockets"],
            "trustedIdentities": ["*"]
        },
        {
            // Policy for all other OCI artifacts signed by ACME Rockets
            // from any registry location. The policy also specified multiple trust stores.
            "name": "global-policy-for-all-other-images",
            "registryScopes": [ "*" ],
            "signatureVerification": {
              "level" : "audit"
            },
            "trustStores": ["ca:acme-rockets", "ca:acme-rockets-ca2"],
            "trustedIdentities": [
              "x509.subject: C=US, ST=WA, L=Seattle, O=acme-rockets.io, OU=Finance, CN=SecureBuilder"
            ]
        }
    ]
}
```

3. Trust policy for a scenario where ACME Rockets consumes some OCI artifacts signed by Wabbit Networks with timestamp countersignature.

```jsonc
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "wabbit-networks-images",
            "registryScopes": [ "*" ],
            "signatureVerification": {
              "level" : "strict",
              "verifyTimestamp": "afterCertExpiry"  // Only verify any timestamp countersignature when codesigning certificate chain has expired
            },
            "trustStores": ["ca:acme-rockets", "tsa:trusted-tsa"], // The trust store type `tsa` MUST be configured to enable timestamp verification.
            "trustedIdentities": [
              "x509.subject: C=US, ST=WA, L=Seattle, O=acme-rockets.io, OU=Finance, CN=SecureBuilder"
            ]
        }
    ]
}
```

#### Trust Policy for Blobs

1. Trust policy for the scenario where ACME Rockets consumes arbitrary blobs signed by Wabbit Networks and some signed by ACME Rockets.

```jsonc
{
    "version": "1.0",
    "trustPolicies": [
        {
            // Policy for set of blobs signed by Wabbit Networks
            "name": "wabbit-networks-blobs",
            "signatureVerification": {
              "level" : "strict"
            },
            "trustStores": ["ca:wabbit-networks"],
            "trustedIdentities": [
              "x509.subject: C=US, ST=WA, L=Seattle, O=wabbit-networks.io, OU=Security Tools"
            ]
        },
        {
            // Policy that uses custom verification level to relax the strict verification.
            // It logs expiry and skips the revocation check.
            "name": "use-expired-blobs",
            "signatureVerification": {
              "level" : "strict",
              "override" : {
                "expiry" : "log",
                "revocation" : "skip"
              }
            },
            "trustStores": ["ca:acme-rockets"],
            "trustedIdentities": ["*"]
        },
        {
            // Global policy for all arbitrarily blobs
            "name": "global-policy-for-all-blobs",
            "signatureVerification": {
              "level" : "audit"
            },
            "globalPolicy": true,
            "trustStores": ["ca:acme-rockets", "ca:acme-rockets-ca2"],
            "trustedIdentities": [
              "x509.subject: C=US, ST=WA, L=Seattle, O=acme-rockets.io, OU=Finance, CN=SecureBuilder"
            ]
        }
    ]
}
```

2. Trust policy for the scenario where ACME Rockets consumes arbitrary blobs signed by Wabbit Networks with timestamp countersignature.

```jsonc
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "wabbit-networks-blobs",
            "signatureVerification": {
              "level" : "strict",
              "verifyTimestamp": "afterCertExpiry"  // Only verify any timestamp countersignature when codesigning certificate chain has expired
            },
            "trustStores": ["ca:acme-rockets", "tsa:trusted-tsa"], // The trust store type `tsa` MUST be configured to enable timestamp verification.
            "trustedIdentities": [
              "x509.subject: C=US, ST=WA, L=Seattle, O=acme-rockets.io, OU=Finance, CN=SecureBuilder"
            ]
        }
    ]
}
```

#### Signature Verification details

- Signature verification is a multi step process performs the following validations
  - integrity (artifact is unaltered, signature is not corrupted)
  - authenticity (the signature is really from the identity that claims to have signed it)
  - authentic timestamp (the signature was generated when the certificate was unexpired)
  - expiry (an optional check if the artifact specifies an expiry time)
  - revocation check (is the signing identity still trusted at the present time).
- Based on the signature verification level, each of these validations is *enforced* or *logged*.
  - If a validation is *enforced*, a failure is treated as critical failure, and causes the overall signature verification to fail.
  - If a validation is *logged*, a failure causes details to be logged, and the next validation is evaluated till all validations succeed or a critical failure is encountered.
- Implementations may change the ordering of these validations based on efficiency, but all validation MUST be performed till the first critical failure is encountered, or all validation succeed, for the overall signature verification process to be considered complete.

 Notary Project defines the following signature verification levels to provide different levels of enforcement for different scenarios.

- `strict` : Signature verification is performed at `strict` level, which enforces all validations. If any of these validations fail, the signature verification fails. This is the recommended level in environments where a signature verification failure does not have high impact to other concerns (like application availability). It is recommended that build and development environments where images are initially created, or for high assurance at deploy time use `strict` level.
- `permissive` : The `permissive` level enforces most validations, but will only logs failures for revocation and expiry. The `permissive` level is recommended to be used if signature verification is done at deploy time or runtime, and the user only needs integrity and authenticity guarantees.
- `audit` : The `audit` level only enforces signature integrity if a signature is present. Failure of all other validations are only logged.
- `skip` : The `skip` level does not perform any signature verification. This is useful when an application uses multiple artifacts, and has a mix of signed and unsigned artifacts. Note that `skip` cannot be used with a global scope (`*`).

The following table shows the resultant validation action, either *enforced* (verification fails), or *logged* for each of the checks, based on signature verification level.

|Signature Verification Level|Recommended Usage|||Validations|||
|----------------------------|-----------------|---------|------------|-----------------|------|----------------|
|||*Integrity*|*Authenticity*|*Authentic timestamp*|*Expiry*|*Revocation check*|
|*strict*    |Use at development, build and deploy time|enforced|enforced|enforced|enforced|enforced|
|*permissive*|Use at deploy time or runtime|enforced|enforced|logged|logged|logged|
|*audit*     |Use when adopting signed images, without breaking existing workflows|enforced|logged|logged|logged|logged|
|*skip*      |Use to exclude verification for unsigned images|skipped|skipped|skipped|skipped|skipped|

**Integrity** : Guarantees that the artifact wasn't altered after it was signed, or the signature isn't corrupted. All signature verification levels always enforce integrity.

**Authenticity** : Guarantees that the artifact was signed by an identity trusted by the verifier. Its definition does not include revocation, which is when a trusted identity is subsequently untrusted because of a compromise.

**Authentic timestamp** : Guarantees that the signature was generated when the certificate was valid. It also allows a verifier to determine if a signature must be treated as valid or invalid based on whether the signature was generated before or after the certificate revocation. In the absence of an Authentic Timestamp, a signature is considered invalid if the signing certificate or chain is either expired or revoked.

**Expiry** : This is an optional feature that guarantees that artifact is within “best by use” date indicated in the signature. Notary Project allows users to include an optional expiry time when they generate a signature. The expiry time is not set by default and requires explicit configuration by users at the time of signature generation. The artifact is considered expired when the current time is greater than or equal to expiry time, users performing verification can either configure their trust policies to fail the verification or even accept the artifact with expiry date in the past using policy. This is an advanced feature that allows implementing controls for user defined semantics like deprecation for older artifacts, or block older artifacts in a production environment. Users should only include an expiry time in the signed artifact after considering the behavior they expect for consumers of the artifact after it expires. Users can choose to consume an artifact even after the expiry time based on their specific needs.

**Revocation check** : Guarantees that the signing identity is still trusted at signature verification time. Events such as key or system compromise can make a signing identity that was previously trusted, to be subsequently untrusted. This guarantee typically requires a verification-time call to an external system, which may not be consistently reliable. The `permissive` verification level only logs failures of revocation check and does not enforce it. If a particular revocation mechanism is reliable, use `strict` verification level instead.

##### Custom Verification Level

Signature verification levels provide defined behavior for each validation e.g. `strict` will always *enforce* authenticity validation. For fine grained control over validations that occur during signature verification, users can define a custom level which overrides the behavior of an existing verification level.

- To use this feature, the `level` property MUST be specified along with an OPTIONAL `override` map.
- Supported values for `level` are - `strict`, `permissive` and `audit`. A `skip` level cannot be customized.
- Supported keys for `override` map and their supported values are as follows:
  - `integrity` validation cannot be overriden, and therefore cannot be specified as a key.
  - `authenticity` - Supported values are `enforce` and `log`.
  - `authenticTimestamp` - Supported values are `enforce` and `log`.
  - `expiry` - Supported values are `enforce` and `log`.
  - `revocation` - Supported values are `enforce`, `log`, and `skip`.

```jsonc
    "signatureVerification": {
      "level" : "strict",
      "override" : {
        "expiry" : "log"
      }
    }
```

#### Timestamp Countersignature Verification details

Sufficient and necessary conditions to trigger timestamp countersignature verification:
  - As the overall switch, `authenticTimestamp` under `signatureVerification` of trust policy is NOT marked as `skip`.
  - The signing scheme is [`notary.x509`](./signing-scheme.md/#notaryx509).
  - `tsa` trust store type is configured under `trustStores` field of the corresponding trust policy statement.
  - `verifyTimestamp` under `signatureVerification` of trust policy is not set or set to `always`; OR it's set to `afterCertExpiry` and at least one certificate in the signing certificate chain has expired at the time of verification.

See the flow chart below for a better understanding:
```mermaid
flowchart TD;
  A{Authentic timestamp};
  B{Signing scheme}; 
  C{`tsa` trust store configured};
  D{`verifyTimestamp`};
  E{Signing cert chain expired};
  R1[Trigger timestamp verification];
  R2[No timestamp verification];

  A --> |Not skip| B
  A --> |Skip| R2
  B --> |notary.x509| C
  B --> |notary.x509.signingAuthority| R2
  C --> |Yes| D
  C --> |No| R2
  D --> |Not set or `always`| R1
  D --> |`afterCertExpiry`| E
  E --> |Yes| R1
  E --> |No| R2
```

Timestamp countersignature verification is a multi step process performs the following validations
  - existence (timestamp token is present in the signature envelope)
  - integrity (timestamp token is unaltered, countersignature is not corrupted)
  - authenticity (the timestamp token is really from the TSA that claims to have signed it)
  - revocation check (is the TSA still trusted at the time of verification)

 **Existence** : Guarantees that the timestamp token exists in the signature envelope. It is always enforced when timestamp countersignature verification is triggered.

 **Integrity** : Guarantees that the timestamp token wasn't altered after it was signed, or the countersignature isn't corrupted. It is always enforced when timestamp countersignature verification is triggered.

 **Authenticity** : Guarantees that the timestamp was issued by a trusted TSA identity. Its definition does not include revocation, which is when a trusted TSA is subsequently untrusted because of a compromise. It is always enforced when timestamp countersignature verification is triggered.

 **Revocation check** : Guarantees that the TSA identity is still trusted at verification time. Events such as key or system compromise can cause a previously trusted TSA identity to become untrusted. It is always enforced when timestamp countersignature verification is triggered.

#### Trust Policy Constraints

##### OCI Trust Policy Constraints

- Each trust policy MUST contain scope property `registryScopes` and the scope collection MUST contain at least one value.
- The scope MUST contain one of the following:
  - List of one or more fully qualified repository URIs.
    The repository URI MUST NOT contain the asterisk character `*`.
  - A single value with one asterisk character `*`.
    The scope with `*` value is called global scope.
    The trust policy with global scope applies to all the artifacts.
    There can only be one trust policy that uses a global scope.

##### Blob Trust Policy Constraints
- There MUST be only one trust policy with `globalPolicy` set to `true`
- `signatureVerification` MUST not be set to `skip` if the policy is marked as a global policy

#### Selecting a trust policy to verify a signed OCI artifact

- The URI of an OCI artifact dictates the trust policy that gets applied to verify the artifact's signature.
- For a given OCI artifact there MUST be only one applicable trust policy, except for trust policy with a global scope.
- For a given OCI artifact, if there is no applicable trust policy then implementations of the [Notary Project verification specification](./signing-and-verification-workflow.md) MUST consider the artifact as untrusted and fail signature verification.
- The scope MUST NOT support reference expansion i.e. URIs must be fully qualified.
  E.g. the scope should be `docker.io/library/registry` rather than `registry`.
- Evaluation order of trust policies:
  1. *Exact match*: If there exists a trust policy whose scope contains the artifact's repository URI then the aforementioned policy MUST be used for signature evaluation.
     Otherwise, continue to the next step.
  1. *Global*: If there exists a trust policy with global scope (`*`) then use that policy for signature evaluation.
     Otherwise, fail the signature verification.

#### Selecting a trust policy to verify a signed blob

- The signature verifier must select the appropriate trust policy for blob signature verification using policy names.
- For a given policy name selected by the verifier, there MUST be only one trust policy with that exact name.
- For a given policy name selected by the verifier, if there is no matching trust policy with that name, then implementations of the [Notary Project verification specification](./signing-and-verification-workflow.md) MUST consider the blob as untrusted and fail signature verification.
- Selecting a trust policy:
  1. *Exact match*: If there exists a trust policy whose name exactly matches the one provided by the signature verifier then the aforementioned policy MUST be used for signature evaluation.
     Otherwise, fail the signature verification.
  1. *Global*: If verifier does not select a policy by its name and if there exists a trust policy marked as a global policy (`globalPolicy:true`) then use that policy for signature evaluation.
     Otherwise, fail the signature verification.

### Trusted Identities Constraints

This section defines how to specify `trustedIdentities` for X.509 certificates using its distinguished name. A distinguished name (usually just shortened to "DN") uniquely identifies the requestor/holder of the certificate.
The DN is comprised of zero or more comma-separated components called relative distinguished names, or RDNs.
For example, the DN `C=US, ST=WA, O=wabbit-network.io, OU=org1` has four RDNs.
The RDN consists of an attribute type name followed by an equal sign and the string representation of the corresponding attribute value.

- The `trustedIdentities` list items MUST support a full and partial list of all the attribute types present in [subject DN](https://www.rfc-editor.org/rfc/rfc5280.html#section-4.1.2.6) of x509 certificate. Alternatively, it supports a single item with value `*`, to indicate that any certificate that chains to the associated trust store (`trustStore`) is allowed.
- If the subject DN of the signing certificate is used in the trust anchor, then it MUST meet the following requirements:
  - The value of each `trustedIdentities` list item, if it begins with `x509.subject:`, MUST be followed by comma-separated one or more RDNs.
    Other types of trusted identities may be supported, by using an alternate prefix, or a different format.
    For example, `x509.subject: C=${country}, ST=${state}, L=${locallity}, O={organization}, OU=${organization-unit}, CN=${common-name}`.
  - Each identity in `identities` list MUST contain country (C), state or province (ST or S), and organization (O) RDNs.
    All other RDNs are optional.
    The minimal possible value is `x509.subject: C=${country}, ST=${state}, O={organization}` or `x509.subject: C=${country}, S=${state}, O={organization}`.
  - `trustedIdentities` list items MUST NOT have overlapping values,
    they are considered overlapping if there exists a certificate for which multiple DNs evaluate true. In such case the policy is considered invalid, and will fail at signature verification time when the policy is validated.
    For example, the following two identity values are overlapping:
    - `x509.subject: C=US, ST=WA, O=wabbit-network.io, OU=org1`
    - `x509.subject:  C=US, ST=WA, O=wabbit-network.io`
  - In some special cases values in `trustedIdentities` list MUST escape one or more characters in an RDN.
    Those cases are:
    - If a value starts or ends with a space, then that space character MUST be escaped as `\`.
    - All occurrences of the comma character (`,`) MUST be escaped as `\,`.
    - All occurrences of the semicolon character (`;`) MUST be escaped as `\;`.
    - All occurrences of the backslash character (`\`) MUST be escaped as `\\`.

### Extended Validation

Notary Project allows user to execute custom validations during verification using plugins. Please refer [plugin-extensibility.md](plugin-extensibility.md#verification-extensibility) for more information.

## Signature Verification

### Prerequisites

- User has configured a valid [trust store](#trust-store) and [trust policy](#trust-policy).
- The artifact's fully qualified repository URI, and associated signature envelope is available.

### Steps

1. **Identify applicable trust policy**
   1. For OCI artifacts, refer to `{NOTATION_CONFIG}/trustpolicy.oci.json` or `{NOTATION_CONFIG}/trustpolicy.json` file and use [the artifact URI as the policy selector](#selecting-a-trust-policy-to-verify-a-signed-oci-artifact) and select the policy with matching scope from `registryScopes`.
   1. For Blob artifacts, refer to `{NOTATION_CONFIG}/trustpolicy.blob.json` and use [the policy name provided by the verifier](#selecting-a-trust-policy-to-verify-a-signed-blob) as the policy selector and select a policy.
   1. If an applicable trust policy cannot be found, fail signature verification.
1. **Proceed based on signature verification level**
   1. If `signatureVerification` level is set to `skip` in the trust policy, return success.
   1. For all other `signatureVerification` levels, `strict`, `permissive` and `audit`, perform each of the validation defined in the next sections - `integrity`, `authenticity`, `trusted timestamp`, `expiry` and `revocation`.
   1. The `signatureVerification` level defines if each validation is `enforced` or `logged`
        1. `enforced` - validation failures are treated as critical, causes the overall signature verification to fail and exit. Subsequent validations are not processed.
        1. `logged` - validation failure is logged and the next validation step is processed.
   1. A signature verification is considered successful when all validation steps are completed without critical failure.
1. **Validate Integrity:**
    1. Validate signature envelope
        1. For OCI artifacts, validate that signature envelope can be parsed successfully based on the signature envelope type specified in the `blobs[0].mediaType` attribute of the signature artifact manifest.
        1. For Blob artifacts, validate that signature envelope can be parsed successfully based on the signature envelope type specified in the detached signature file extension. If no file extension is available, parse the envelope type by trying each of the [supported signature envelopes](./signature-specification.md#supported-signature-envelopes). Fail integrity check if unsuccessful.
    1. Validate that the content type indicated by the `content type` signed attribute in the signature envelope is supported.
    1. Get the signing certificate from the parsed [signature envelope](https://github.com/notaryproject/notaryproject/blob/7b7d283038/signature-specification.md#signature-envelope).
    1. Determine the signing algorithm(hash+encryption) from the signing certificate and validate that the signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements)
    1. Using the public key of the signing certificate and signing algorithm identified in the previous step, validate the integrity of the signature envelope.
    1. Verify signature `payload`
        1. Verify the signature envelope's `payload` matches the source [`payload`](./signature-specification.md#payload) that is getting verified. Make sure the artifact's digest, media type and size match the ones present in the signature envelope.
        1. Additionally for Blob artifacts, calculate the digest of the blob using the hashing algorithm deduced from signing certificate's public key (see [Algorithm Selection](./signature-specification.md#algorithm-selection)) and make sure the digests match.
1. **Validate Authenticity.**
    1. For the applicable trust policy, **validate trust store and identities:**
        1. Validate that the signature envelope contains a complete certificate chain that starts from a code signing certificate and terminates with a root certificate. Also, validate that code signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
        1. For the `trustStore` configured in applicable trust-policy perform the following steps.
            1. Validate that certificate and certificate-chain lead to a trusted certificate present in the named store.
            1. If all the certificates in the named store have been evaluated without a match, then fail this step.
        1. If `trustedIdentities` is `*`, any signing certificate issued by a CA in `trustStore` is allowed, skip to the next validation (Validate Expiry).
        1. Else validate if the X.509 subject (`x509.subject`) in the `trustedIdentities` list matches with the value of corresponding attributes in the signing certificate’s subject, refer [this section](#trusted-identities-constraints) for details. If a match is not found, fail this step.
1. **Validate Expiry:**
    1. If an `expiry time` signed attribute is present in the signature envelope, check if the local machine’s current time(in UTC) is greater than `expiry time`. If yes, fail this step.
1. **Validate Authentic Timestamp:**
    1. If under signing scheme [`notary.x509`](./signing-scheme.md/#notaryx509):
        1. If [`sufficient and necessary conditions to trigger timestamp countersignature verification`](./trust-store-trust-policy.md/#timestamp-countersignature-verification-details) are satisfied, perform timestamp countersignature verification:
            1. Check the unsigned attribute `Timestamp Signature`, if not present or empty, fail this step.
            1. Verify the timestamp countersignature and validate the `TSTInfo` based on [RFC 3161](https://datatracker.ietf.org/doc/html/rfc3161) and [RFC-5816](https://datatracker.ietf.org/doc/html/rfc5816).
            1. Validate that the timestamp hash in `TSTInfo.messageImprint` matches the hash of the signature to which the timestamp was applied.
            1. Validate that the timestamp signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
            1. Validate that the timestamp signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
            1. Validate the `signing-certificate-v2` ([RFC-5126](https://tools.ietf.org/html/rfc5126#section-5.7.3.2)) attribute of timestamp CMS. When missing, fail this step.
            1. Validate that the timestamp certificate chain leads to a trusted root certificate as per setting in `trustStore` with trust store type `tsa` in trust policy.
            1. Validate timestamp certificate chain revocation status using [certificate revocation evaluation](#certificate-revocation-evaluation) section.
            1. Retrieve the timestamp's time from `TSTInfo.genTime`.
            1. Retrieve the timestamp's accuracy.
            If the accuracy is explicitly specified in `TSTInfo.accuracy`, use that value.
            If the accuracy is not explicitly specified and `TSTInfo.policy` is the baseline time-stamp policy([RFC-3628](https://tools.ietf.org/html/rfc3628#section-5.2)), use accuracy of 1 second.
            Otherwise, use an accuracy of 0.
            1. Calculate the timestamp range using the lower and upper limits per [RFC 3161 section 2.4.2](https://tools.ietf.org/html/rfc3161#section-2.4.2) and store the limits as `timeStampLowerLimit` and `timeStampUpperLimit` variables respectively.
            1. Validate that the time range from `timeStampLowerLimit` to `timeStampUpperLimit` is entirely within the signing certificate chain's validity period. If the validation passes, continue to the next validation (Validate Revocation Status). Else fail this step.
        1. If step 6.i.a is not performed. Validate that the local machine's current time (in  UTC) is within the signing certificate chain's validity period. If the validation passes, continue to the next validation (Validate Revocation Status). Else fail this step.

    1. If under signing scheme [`notary.x509.signingAuthority`](./signing-scheme.md/#notaryx509signingauthority):
        1. Check for the `Authentic Signing Time` signed attribute. If it does not exist, fail this step.
        1. Validate that the `Authentic Signing Time` is within the signing certificate chain's validity period. If the validation passes, continue to the next validation (Validate Revocation Status). Else fail this step.

1. **Validate Revocation Status:**
    1. Validate signing identity(certificate and certificate chain) revocation status using [certificate revocation evaluation](#certificate-revocation-evaluation) section as per `revocation` setting in trust policy.
1. Perform extended validation using the applicable (if any) plugin.
1. If present, verify the user provided metadata/custom annotations match the signed attributes present in the signature envelope.
1. If all the steps are completed without critical failures then the signatures is successfully verified.

### Certificate Revocation Evaluation

If the certificate revocation validation is set to `skip` using policy, skip the below steps, otherwise check for revocation status for certificate and certificate chain.

If the revocation status of any of the certificates is revoked or cannot be determined (revocation unavailable), fail this step. This will fail the overall signature verification or log the error, depending on if revocations status check is `enforced` or `logged`, as per the signature verification level.

Starting from Root to leaf certificate, for each certificate in the certificate chain, perform the following steps to check its revocation status:

- If the certificate being validated doesn't include information OCSP or CRLs then no revocation check is performed and the certificate is considered valid(not revoked).
- If the certificate being validated includes either OCSP or CRL information, then the one which is present is used for revocation check.
- If both OCSP URLs and CDP URLs are present, then OCSP is preferred over CRLs.
  If revocation status cannot be determined using OCSP because of any reason such as unavailability then fallback to using CRLs for revocation check.

#### CRLs

There are two types of CRLs (per RFC 3280), Base CRLs and Delta CRls.

- **Base CRLs:** Contains the revocation status of all certificates that have been issued by a given CA.
  BaseCRLs are signed by the certificate issuing CAs.
- **Delta CRLs:** Contains only certificates that have changed status since the last base CRL was published.
  Delta CRLs are signed by the certificate issuing CAs.
- **Indirect CRLs:** Special Case in which BaseCRLs and Delta CRLs are not signed by the certificate issuing CAs, instead they are signed with a completely different certificate.

Implementations of the [Notary Project verification specification](./signing-and-verification-workflow.md) MUST support BaseCRLs and Delta CRLs.
Implementations of the [Notary Project verification specification](./signing-and-verification-workflow.md) MAY support Indirect CRLs.
Implementations of the [Notary Project verification specification](./signing-and-verification-workflow.md) support only HTTP CRL URLs.

Implementations MAY add support for caching CRLs and OCSP response to improve availability, latency and avoid network overhead.

##### CRL Download

CRL download location (URL) can be obtained from the certificate's CRL Distribution Point (CDP) extension.
If the certificate contains multiple CDP locations then each location download is attempted in sequential order, until a 200 response is received for any of the location.
For each CDP location, [Notary Project verification workflow](./signing-and-verification-workflow.md) will try to download the CRL for the default threshold of 5 seconds.
The user may be able to configure this threshold.
If the CRL cannot be downloaded within the timeout threshold the revocation result will be "revocation unavailable".

##### Revocation Checking with CRL

To check the revocation status of a certificate against CRL, the following steps must be performed:

1. Verify the CRL signature.
1. Verify that the CRL is valid (not expired).
   A CRL is considered expired if the current date is after the `NextUpdate` field in the CRL.
1. Look up the certificate’s serial number in the CRL.
    1. If the certificate’s serial number is listed in the CRL, look for `InvalidityDate`.
       If CRL has an invalidity date and artifact signature is timestamped then compare the invalidity date with the timestamping date.
       1. If the invalidity date is before the timestamping date, the certificate is considered revoked.
       1. If the invalidity date is not present in CRL, the certificate is considered revoked.
    1. If the CRL is expired and the certificate is listed in the CRL for any reason other than `certificate hold`, the certificate is considered revoked.
    1. If the certificate is not listed in the CRL or the revocation reason is `certificate hold`, a new CRL is retrieved if the current time is past the time in the `NextPublish` field in the current CRL.
       The new CRL is then checked to determine if the certificate is revoked.
       If the original reason was `certificate hold`, the CRL is checked to determine if the certificate is unrevoked by looking for the `RemoveFromCRL` revocation code.

##### Revocation Checking with Delta CRLs

If a delta CRL exists for a base CRL, the `Freshest CRL` extension in the base CRL provides the URL from where the delta CRL can be downloaded.
The delta CRLs have their identifying number, plus the "Delta CRL Indicator" extension indicating the version number of the base CRL that must be used with the delta CRL.

When delta CRLs are implemented, the following results can occur during revocation checking.

- If the delta CRL cannot be retrieved for some reason, the revocation result will be "revocation unavailable".
- If the delta CRL’s indicator is less than the current base CRL, the revocation result returned will be "revocation unavailable"
- To check revocation of a certificate in Delta CRLs follow steps similar to [Revocation Checking with CRL](#revocation-checking-with-crl).

#### OCSP

##### OCSP Download

OCSP URLs can be obtained from the certificate's authority information access (AIA) extension as defined in [RFC 6960](https://www.rfc-editor.org/rfc/rfc6960).
If the certificate contains multiple OCSP URLs, then each URL is invoked in sequential order, until a 200 response is received for any of the URL.
For each OCSP URL, wait for a default threshold of 2 seconds to receive an OCSP response.
The user may be able to configure this threshold.
If OCSP response is not available within the timeout threshold the revocation result will be "revocation unavailable".

##### Revocation Checking with OCSP

To check the revocation status of a certificate using OCSP, the following steps must be performed.

1. Verify the signature of the OCSP response.
   This step also includes verifying the OSCP response signing certificate is valid and trusted.
   The OCSP signing certificate must be issued by the same CA as the certificate being verified or the OCSP response must be signed by the issuing CA.
1. Verify that the OCSP response is valid (not expired).
   Response is considered expired if the current date is after the `NextUpdate` field in the response.
1. Verify that the OCSP response indicates that the certificate is not revoked i.e `CertStatus` is `good`.
    1. If the certificate is revoked i.e `CertStatus` is `revoked`, look for `InvalidityDate`.
        1. If the invalidity date is present and the authentic signing time is also present then if the invalidity date is before the authentic signing time, the certificate is considered revoked.
        1. If the invalidity date is not present in OCSP response, the certificate is considered revoked.
    1. If the certificate status is `unknown`, the revocation status is considered `revocation unavailable`.
1. If `id-pkix-ocsp-nocheck`(1.3.6.1.5.5.7.48.1.5) extension is not present on the OCSP signing certificate then revocation checking must be performed using CRLs for the OCSP signing certificate.

## FAQ

**Q: Does the Notary Project trust policy supports `n` out of `m` signatures verification requirement?**

**A:** The Notary Project trust policy doesn't support n out m signature requirement verification scheme.
Signature verification workflow succeeds if verification succeeds for at least one signature.

**Q: Does the Notary Project trust policy support overriding of revocation endpoints to support signature verification in disconnected environments?**

**A:** TODO: Update after verification extensibility spec is ready.
Not natively supported but a user can configure `revocationValidations` to `skip` and then use extended validations to check for revocation.

**Q: Why user needs to include a complete certificate chain (leading to root) in the signature?**

**A:** Without a complete certificate chain, the implementation won't be able to perform an exhaustive revocation check, which will lead to security issues, and that's the reason for enforcing a complete certificate chain.

**Q: Why are we validating artifact signature first instead of signing identity?**

**A:** Ideally, we should validate the signing identity first and then use the public key in the signing identity to validate the artifact signature.
However, this will lead to poor performance in the case where the signature is not valid as there are lots of validations against the signing identity including network calls for revocations, and possibly we won't even need to read the trust store/trust policy if the signature validation fails.
Also, by validating artifact signature first we will still fail the validation if the signing identity is not trusted.