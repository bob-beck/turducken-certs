---
title: "Embedded Alternate Issuers"
category: std

docname: draft-davidben-lamps-turducken-certs-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Limited Additional Mechanisms for PKIX and SMIME"
keyword:
venue:
  group: "Limited Additional Mechanisms for PKIX and SMIME"
  type: "Working Group"
  mail: "spasm@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/spasm/"
  github: "davidben/turducken-certs"
  latest: "https://davidben.github.io/turducken-certs/draft-davidben-lamps-turducken-certs.html"

author:
 -
    fullname: "David Benjamin"
    organization: "Google LLC"
    email: "davidben@google.com"

normative:

informative:
  PQ-AUTH-ROADMAP:
    target: https://www.chromium.org/Home/chromium-security/post-quantum-auth-roadmap/
    title: Post-Quantum HTTPS Authentication Roadmap
    author:
    - name: "David Benjamin"
    - name: "Joe DeBlasio"
    date: 2026-02-27

  IETF126-PQ-AUTH:
    target: https://datatracker.ietf.org/meeting/126/materials/slides-126-saag-post-quantum-authentication-beating-the-s-curve-00
    title: "Post-Quantum Authentication: Beating the S-Curve"
    author:
    - name: "David Benjamin"
    date: 2026-07-24

  IETF124-PLANTS:
    target: https://datatracker.ietf.org/meeting/124/materials/slides-124-plants-solution-space-and-dispatched-work-00.pdf
    title: "PLANTS BoF: Solution Space and Dispatched Work"
    author:
    - name: "David Benjamin"
    date: 2025-11-04

  EMUDHRA:
    target: https://emudhra.we-have-a-problem.com/
    title: "eMudhra Intermediate CA Compromise"
    date: 2026-10-20

...

--- abstract

This document defines a mechanism to embed one or more alternate certificates within a single outer certificate. Each alternate certificate is constrained to certify the same subject information, but with a different issuer. Rather than a general purpose mechanism, embedded alternate certificates are a temporary measure for PKI transitions, such as the post-quantum transition, in PKIs with significant legacy deployments.

--- middle

# Introduction

An X.509 certificate {{!RFC5280}} relates two entities in a PKI: a *subject* described by the certificate and an *issuer* who has certified that information. If a relying party trusts the issuer to only certify correct subject information, it can trust the subject information in the certificate.

Security transitions in a PKI must coordinate certificate changes across this system. For example, securing a PKI application against a Cryptographically Relevant Quantum Computer (CRQC) requires issuers to issue some form of post-quantum certificate, services to deploy those certificates, and relying parties to require these certificates.

Until the final step, post-quantum certificates do not prevent attacks. Suppose `example.com` has deployed post-quantum certificates, but `legacy.example` has not. A relying party must trust both post-quantum and traditional CAs to support both services. However, this enables a downgrade attack against `example.com`. A CRQC can simply break the traditional CA's key and forge a certificate for `example.com`.

That is, in order to protect services against CRQC, relying parties must remove support for existing traditional, quantum-vulnerable CAs. This poses a compatibility challenge for PKIs with a large number of existing services.

## Subject and Issuer Transitions

A PKI transition might change the subject information (e.g. new subject key algorithms), the issuer (e.g. new issuer key algorithms or new issuers entirely), or both. When both change, as in the post-quantum transition, the two changes can still complete on separate timelines, with different parties taking action:

* Subject transitions often require changes to a service's software stack and key management. For example, to authenticate with an ML-DSA {{?FIPS204=DOI.10.6028/NIST.FIPS.204}} end-entity key, TLS {{?RFC8446}} serving software must support ML-DSA, and the operator must maintain an ML-DSA private key.

* Issuer transitions impact fields that are generally opaque to the service, such as the certificate signature. In principle, a CA or ACME server {{?RFC8555}} operator could simply reconfigure itself to issue from the new issuer. Deployed services will then automatically apply the issuer transition as they renew certificates.

PKIs typically have many more service operators than CA operators. Issuer transitions that only require CA operator action can complete sooner. In the post-quantum transition, although both issuer and subject changes are necessary to protect a given service, only the issuer transition needs to *complete* to prevent the downgrade attack. Concretely:

1. Relying parties add support for post-quantum subject keys and post-quantum CAs.

2. Some services migrate ahead of time, but many legacy services exist. Relying parties must remain compatible with legacy services, so the downgrade attack is possible.

3. Existing ACME servers are reconfigured to start issuing from post-quantum CAs, instead of traditional CAs.

4. Legacy services automatically switch to post-quantum CAs when renewing certificates. Although their subject keys are still traditional, this is sufficient to unblock step 5 below. These certificates are "post-quantum opt-out certificates" for these legacy services.

5. After certificates are automatically replaced, relying parties remove traditional CAs while retaining support for traditional subject keys. This is enough to prevent the downgrade attack.

6. Legacy services individually upgrade their subject keys and replace their opt-out certificates with fully post-quantum certificates.

In this example, steps 1 through 5 do not require waiting on all service operators. Step 6 does and will thus take time to fully complete, but the downgrade attack has been prevented by step 5. Although opt-out certificates have quantum-vulnerable subject keys, they are issued by post-quantum CAs. Provided relying parties remove all traditional CAs, a CRQC cannot forge one for a post-quantum service. {{PQ-AUTH-ROADMAP}} and {{IETF126-PQ-AUTH}} discuss this dynamic in more detail.

However, these opt-out certificates are not compatible with legacy relying parties. A legacy relying party will not support post-quantum algorithms, much less post-quantum CAs. Steps 3 and 4 are not viable in applications with many relying party instances (e.g. many web browser installs) and require further refinement.

## Combining Multiple Issuers

With sufficient deployment capabilities, a CA or ACME server {{?RFC8555}} operator could issue *both* an older and a newer certificate, relying on an online protocol like TLS to select between them:

* If issuance protocols, such as ACME {{?RFC8555}}, can issue multiple certificates, the ACME server can transparently provision certificates from both the old and new issuer.

* If the TLS software can automatically send each certificate to the right relying party (as in {{Sections 4.4.2.2 and 4.4.4.3 of ?RFC8446}} and {{?I-D.ietf-tls-trust-anchor-ids}}), the TLS software is now ready for new relying parties while remaining compatible with older ones.

However, deployments may be missing these capabilities today. Many Web servers today only deploy one certificate with one key. Waiting for services to upgrade will again delay transition timelines.

This document defines *embedded alternate issuers*, a limited transitionary mechanism for these scenarios. A single X.509 certificate, known as the *outer certificate* and issued by the *outer CA*, embeds one or more *alternate certificates*, each certifying the same subject information but issued by an *alternate CA*. An ACME server can then drive a transition as described above, but embedding certificates for new relying parties inside the certificate for older relying parties.

For a post-quantum transition, this means the opt-out certificate is a combination where:

* The outer certificate certifies a traditional subject key with a traditional CA.
* It has one or more alternate certificates that certify the same traditional subject key, but with a post-quantum CA.

This mitigates the compatibility issue described in {{subject-and-issuer-transitions}}.

Embedded alternate issuers only apply when all certificates describe the same subject information. They are a temporary transitionary tool until services can deploy multiple certificates directly. {{legacy-services}} and {{certificate-mismatch}} discuss this in more detail.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document additionally uses the TLS presentation language defined in {{Section 3 of !RFC8446}}.

## Terminology

Subject:
: The entity being described by a certificate.

Issuer:
: The CA that certified the information in a certificate.

Outer certificate:
: A certificate that embeds one or more alternate certificates using the mechanism defined in this document.

Outer CA:
: The CA that issues the outer certificate.

Alternate certificate:
: One of the certificates embedded in the outer certificate. An alternate certificate certifies the same subject information as the outer certificate, but via a different issuer.

Alternate CA:
: A CA that issues one of the alternate certificates.

# Overview

Outer and alternate certificates are closely related and issued in an integrated process:

* All certificates contain the same subject information, including subjectPublicKeyInfo and subjectAltNames, but they differ in signature and issuer-specific information. The common subject-specific extensions all share an ordering.

* The outer certificate embeds an AlternateIssuerInfo for each alternate certificate. This contains both the signature and issuer-specific information for the alternate certificate. It is sufficient to reconstruct the alternate certificate.

* Each alternate certificate contains an outer certificate binding extension. This contains the issuer-specific fields of the outer certificate, but not the signature. This authenticates the outer certificate's issuer-specific information in relying parties that only trust the alternate CA.

## Issuing Certificates

The certificates are issued in the following process. From the perspective of the outer CA, it is analogous to embedding Signed Certificate Timestamps (SCTs) in Certificate Transparency (CT) {{!RFC6962}}. From the perspective of each alternate CA, it is analogous to issuing a normal certificate, but with an extra extension included.

1. The outer CA validates the subject information and constructs a TBSCertificate with the validated subject information and issuer fields reflecting the outer CA. It sets `notBefore` and `notAfter` such that the certificate validity would be compatible with each alternate CA.

2. For each alternate CA, the outer and alternate CAs run the following process. The processes for each alternate CA MAY run in parallel.

   {:type="a"}
   1. The outer CA requests the alternate CA to issue a corresponding alternate certificate for this TBSCertificate.

   2. The alternate CA validates the subject information. If validation fails, or it is otherwise unable to issue this certificate, including with the same `notBefore` and `notAfter`, it rejects this request.

   3. Otherwise, it constructs a TBSCertificate containing the same subject information. The outer CA's issuer-specific fields are removed and replaced with ones reflecting the alternate CA.

   4. The alternate CA appends an outer certificate binding extension to the TBSCertificate, containing all the removed outer-CA-specific fields. This authenticates the removed fields to a relying party that only trusts the alternate CA.

   5. The alternate CA signs this TBSCertificate to issue an alternate certificate.

   6. The alternate CA extracts the alternate certificate's issuer-specific fields and signature. It returns these to the outer CA as an AlternateIssuerInfo. This contains enough information for the relying party to reconstruct the alternate certificate.

3. If the outer CA participates in Certificate Transparency, it additionally signs a precertificate for this TBSCertificate, including the critical poison extension, as described in {{Section 3.1 of !RFC6962}}. The outer CA then requests SCTs from CT logs. This step MAY run in parallel with step 2.

4. The outer CA collects all AlternateIssuerInfos and SCTs and appends an `id-ce-embeddedSCT-CTv1` extension to its TBSCertificate.

5. The outer CA signs the TBSCertificate to complete the outer certificate.

{{fig-turducken-issuance}} illustrates this process. Although the two issuance processes are now integrated, from the outer CA's perspective, it is analogous to embedding SCTs in CT. From the alternate CA's perspective, this is issuing a certificate with a specific extension added.

~~~ aasvg
 CERTIFICATION AUTHORITY A        CERTIFICATION AUTHORITY B
---------------------------      ---------------------------

TBSCertificate                   TBSCertificate
+-----------------+                +-----------------+
|   SUBJECT INFO  |                |   SUBJECT INFO  |
+-----------------+    ------->    +-----------------+
|     A INFO      |                |     B INFO      |
+-----------------+                +-----------------+
                                           +----------------+
                                       |   |     A INFO     |
                                       |   +----------------+
                                       V

                                 Alternate Certificate
                                 +-----------------+---+
                                 |   SUBJECT INFO  |   |
                                 +-----------------+ B |
                                 |     B INFO      |   |
                                 +-----------------+ S |
                                 | Outer Cert Bind | I |
                                 |  +--------------+ G |
                                 |  |    A INFO    |   |
Outer Certificate                +--+--------------+---+
+-----------------+---+
|   SUBJECT INFO  |   |                   |
+-----------------+ A |                   |
|     A INFO      |   |                   V
+-----------------+ S |
| SCTs            | I |            "SCT"
|  +--------------+ G |            +--------------+
|  |    B INFO    |   |            |    B INFO    |
+  +--------------+   |   <-----   +--------------+
|  |    B SIG     |   |            |    B SIG     |
+--+--------------+---+            +--------------+
~~~
{: #fig-turducken-issuance title="A diagram of how the alternate and outer certificates are issued. A is the outer CA, and B is an alternate CA."}

{{certification-authorities}} discusses this in more detail.

## Verifying Certificates

When the certificate subject presents the outer certificate to a relying party, relying parties will attempt to validate one of several certificates:

* Older relying parties that are not aware of this protocol will ignore the unrecognized SCT and validate the outer certificate as usual.

* Newer relying parties will reconstruct the alternate certificates and use them as candidate target certificates when building certification paths {{!RFC4158}}.

This allows the same certificate to serve both older relying parties that expect the outer CA, and newer relying parties that expect one of the alternate CAs.

{{relying-parties}} discusses this in more detail.

# Certification Authorities

Outer and alternate certificates are constrained to share subject information. While the outer CA and each alternate CA MAY have different operators, they are expected to coordinate their certificate profiles and issuance processes. Concretely:

The outer and alternate certificates MUST have a `version` of v3 and MUST omit `issuerUniqueID` and `subjectUniqueID`. These are preexisting requirements from {{Sections 4.1.2.1 and 4.1.2.8 of !RFC5280}}.

The outer and alternate certificates MUST have the same `validity`, `subject`, and `subjectPublicKeyInfo` values.

The outer and alternate certificates' `extensions` lists MUST satisfy the following:

* Each extension list MUST consist of, in order, some number of *subject extensions*, some number of *issuer extensions*, and a single final extension. See {{extension-types}} for details.

* The subject extensions across all certificates MUST be equal and appear in the same order.

* The outer certificate's final extension MUST have type `id-ce-embeddedSCT-CTv1`, as described in {{alternate-certificate-information}}.

* Each alternate certificate's final extension MUST have type `id-pe-outerCertificateBinding`, as described in {{outer-certificate-binding}}.

{{certificate-mismatch}} discusses why these constraints are necessary.

## Extension Types

As discussed in {{certificate-mismatch}}, this document requires outer and alternate certificates to certify the same subject information. X.509 does not distinguish between certificate extensions that relate to the subject and those that relate to the issuer.

The registry defined in {{iana-extension-types}} classifies extensions into *subject extensions* and *issuer extensions*.

Broadly, subject extensions are those that define properties of the certificate subject. These are expected to match no matter which CA certifies this information. Subject extensions include:

* `id-ce-subjectKeyIdentifier`, {{Section 4.2.1.2 of !RFC5280}}
* `id-ce-keyUsage`, {{Section 4.2.1.3 of !RFC5280}}
* `id-ce-certificatePolicies`, {{Section 4.2.1.4 of !RFC5280}}
* `id-ce-policyMappings`, {{Section 4.2.1.5 of !RFC5280}}
* `id-ce-subjectAltName`, {{Section 4.2.1.6 of !RFC5280}}
* `id-ce-subjectDirectoryAttributes`, {{Section 4.2.1.8 of !RFC5280}}
* `id-ce-basicConstraints`, {{Section 4.2.1.9 of !RFC5280}}
* `id-ce-nameConstraints`, {{Section 4.2.1.10 of !RFC5280}}
* `id-ce-policyConstraints`, {{Section 4.2.1.11 of !RFC5280}}
* `id-ce-extKeyUsage`, {{Section 4.2.1.12 of !RFC5280}}
* `id-ce-inhibitAnyPolicy`, {{Section 4.2.1.14 of !RFC5280}}
* `id-pe-subjectInfoAccess`, {{Section 4.2.2.2 of !RFC5280}}

Issuer extensions are specific to the certificate issuer. These include:

* `id-ce-authorityKeyIdentifier`, {{Section 4.2.1.1 of !RFC5280}}
* `id-ce-issuerAltName`, {{Section 4.2.1.7 of !RFC5280}}
* `id-ce-cRLDistributionPoints`, {{Section 4.2.1.13 of !RFC5280}}
* `id-ce-freshestCRL`, {{Section 4.2.1.15 of !RFC5280}}
* `id-pe-authorityInfoAccess`, {{Section 4.2.2.1 of !RFC5280}}
* `id-ce-embeddedSCT-CTv1`, {{Appendix B of !RFC9162}}
* `id-ce-transparencyInfo`, {{Appendix B of !RFC9162}}
* `id-pe-outerCertificateBinding`, {{outer-certificate-binding}}

Concretely, an extension is a subject extension if its extension ID appears in the registry with an "Extension Type" column of "Subject". It is an issuer extension if the "Extension Type" column is "Issuer". X.509 is extensible, so the registry cannot be exhaustive. If the extension ID does not appear in the registry, its classification is not yet defined.

Existing CAs and relying parties will not be aware of classifications that postdate them. While a correctly-operated CA will not issue certificates with unrecognized extensions, relying parties process unrecognized non-critical extensions. {{outer-certificate-binding}} discusses the checks performed by a CA, while {{known-subject-extensions}} discusses the checks performed by a relying party.

## Alternate Certificate Information

Each alternate certificate is described as differences from the outer certificate in an AlternateIssuerInfo structure. An AlternateIssuerInfo is defined as follows:

~~~ asn.1
AlternateIssuerInfo ::= SEQUENCE {
    serialNumber          CertificateSerialNumber,
    signatureAlgorithm    AlgorithmIdentifier{SIGNATURE-ALGORITHM,
                              {SignatureAlgorithms}},
    issuer                Name,
    subjectExtensionCount INTEGER (0..MAX),
    issuerExtensions      SEQUENCE OF Extension{{IssuerExtensions}},
    signatureValue        BIT STRING
}

AlternateIssuerInfoList ::= SEQUENCE OF AlternateIssuerInfo

IssuerExtensions EXTENSION ::= {
    ext-AuthorityKeyIdentifier | ext-IssuerAltName |
    ext-CRLDistributionPoints | ext-FreshestCRL |
    ext-AuthorityInfoAccess |
    ext-embeddedSCT-CTv1 | ext-transparencyInfo |
    ext-outerCertificateBinding,
    ...
}
~~~

An AlternateIssuerInfo's fields are defined as follows:

* `serialNumber` is the alternate certificate's serial number.
* `signatureAlgorithm` is the alternate certificate's signature algorithm. This value appears in both the TBSCertificate and Certificate structures.
* `issuer` is the alternate certificate's issuer name.
* `subjectExtensionCount` is the number of common subject extensions in the two certificates. It MUST be strictly less than the number of extensions in the outer certificate, to exclude the outer certificate's final extension.
* `issuerExtensions` is the list of issuer extensions in the alternate certificate, excluding the final `id-pe-outerCertificateBinding` extension. Each extension in this list MUST be an issuer extension.
* `signatureValue` is the alternate certificate's signature value. This is a signature over the reconstructed TBSCertificate, not the AlternateIssuerInfo.

AlternateIssuerInfos are embedded within the outer certificate's final extension. This extension MUST be of type `id-ce-embeddedSCT-CTv1` (OID 1.3.6.1.4.1.11129.2.4.2). The format of the contents is defined in {{Section 3.3 of !RFC6962}}. ({{!RFC6962}} was obsoleted by {{?RFC9162}}, but the two protocols are incompatible. This document uses the definitions from {{!RFC6962}} for compatibility with existing deployments.)

The extension MUST contain a Signed Certificate Timestamp (SCT) ({{Section 3.2 of !RFC6962}}) of the following form:

* As in {{?RFC6962}}, the `sct_version` field MUST be `v1` (zero).
* The `id` field MUST be the 32-byte string represented in hexadecimal as `abdcf1b6ca6af3c91f37f54fbafdbd17e711780d5197d15d0e1e71106d8cb6c5`. This is the SHA-256 {{!SHS=DOI.10.6028/NIST.FIPS.180-4}} hash of the ASCII string "Embedded Alternate Issuers".
* The `timestamp` field MUST be zero.
* The `extensions` field MUST be the empty string.
* The `algorithm` field of the digitally-signed element ({{Section 4.7 of !RFC5246}}) MUST have a `hash` of `none` (zero) and a `signature` of `anonymous` (zero). ({{!RFC5246}} was obsoleted by {{?RFC8446}} but the SignedCertificateTimestamp structure relies on definitions from {{!RFC5246}}.)
* The `signature` field of the digitally-signed element MUST contain a DER-encoded AlternateIssuerInfoList structure, with one AlternateIssuerInfo per alternate certificate.

The `id-ce-embeddedSCT-CTv1` extension MAY contain other SCTs, for example, if the outer certificate itself participates in Certificate Transparency.

## Outer Certificate Binding

Each alternate certificate contains an outer certificate binding extension, defined below:

~~~ asn.1
OuterCertificateBinding ::= SEQUENCE {
    serialNumber          CertificateSerialNumber,
    signatureAlgorithm    AlgorithmIdentifier{SIGNATURE-ALGORITHM,
                              {SignatureAlgorithms}},
    issuer                Name,
    subjectExtensionCount INTEGER (0..MAX),
    issuerExtensions      SEQUENCE OF Extension{{IssuerExtensions}}
}

id-pe-outerCertificateBinding OBJECT IDENTIFIER ::= {
    iso(1) identified-organization(3) dod(6) internet(1) security(5)
    mechanisms(5) pkix(7) pe(1) TBD2 }

ext-outerCertificateBinding EXTENSION ::= {
    SYNTAX OuterCertificateBinding
    IDENTIFIED BY id-pe-outerCertificateBinding
    CRITICALITY FALSE
}
~~~

The OuterCertificateBinding structure is similar to the AlternateIssuerInfo, except that it does not contain a `signatureValue` field. It describes the outer certificate, relative to the alternate certificate.

* `serialNumber` is the outer certificate's serial number.
* `signatureAlgorithm` is the outer certificate's signature algorithm. This value appears in both the TBSCertificate and Certificate structures.
* `issuer` is the outer certificate's issuer name.
* `subjectExtensionCount` is the number of common subject extensions in the two certificates.
* `issuerExtensions` is the list of issuer extensions in the outer certificate, excluding the final `id-ce-embeddedSCT-CTv1` extension.

The outer certificate binding extension ensures that, even if a relying party only trusts the alternate CA, the outer certificate's contents are still authenticated by a CA trusted by the relying party. The alternate CA MUST check the OuterCertificateBinding is well-formed before issuing the certificate. In particular:

* The outer certificate binding extension MUST be the final extension in the alternate certificate.

* `subjectExtensionCount` MUST be strictly less than the number of extensions in the alternate certificate, to exclude the alternate certificate's outer certificate binding extension.

* Each extension in the alternate certificate, after the first `subjectExtensionCount` extensions, MUST be an issuer extension, as described in {{extension-types}}. That is, the alternate certificate can only add issuer extensions.

* Each extension in `issuerExtensions`, critical or not, MUST also be an issuer extension. That is, the alternate certificate can only remove issuer extensions.

If the alternate certificate does not satisfy these requirements, the alternate certificate is misissued. The above requirements apply even if the extension is not in the registry or is not known to the CA.

The relying party performs a different version of this check in {{known-subject-extensions}}, due to the possibility of unrecognized extensions.

# Relying Parties

When verifying a certificate, a relying party checks if the outer certificate's final extension is `id-ce-embeddedSCT-CTv1`. If so, it looks for an SCT formatted as in {{alternate-certificate-information}}. If found, it decodes the SCT's `signature` value as a DER-encoded AlternateIssuerInfoList structure.

If any AlternateIssuerInfo has a recognized `issuer`, the relying party reconstructs the alternate certificate as described in {{reconstructing-the-alternate-certificate}}. If successful, the alternate certificate is used as a candidate target certificate when building certification paths {{!RFC4158}}.

If certification path validation ({{Section 6 of !RFC5280}}) succeeds for any path ending at an alternate certificate, the relying party accepts the certificate, even if it would otherwise not accept the outer certificate.

## Reconstructing the Alternate Certificate

The following procedure reconstructs an alternate certificate from the outer certificate and an AlternateIssuerInfo.

1. Check each of the following. If any check fails, abort this procedure.

   * The outer certificate's `version` is `v3`.
   * The outer certificate's `signatureAlgorithm` and `signature` fields are equal.
   * The outer certificate has no `subjectUniqueID` or `issuerUniqueID` field.
   * The outer certificate has at least `subjectExtensionCount + 1` extensions and the final extension is `id-ce-embeddedSCT-CTv1`.

2. Make a copy of the outer certificate as a Certificate structure.

3. Replace the TBSCertificate's `serialNumber` field with the AlternateIssuerInfo's `serialNumber`. Let `outerSerialNumber` be the replaced value.

4. Replace the Certificate's `signatureAlgorithm` field and the TBSCertificate's `signature` field with the AlternateIssuerInfo's `signatureAlgorithm`. Let `outerSignatureAlgorithm` be the replaced value.

5. Replace the TBSCertificate's `issuer` field with the AlternateIssuerInfo's `issuer`. Let `outerIssuer` be the replaced value.

6. Remove the TBSCertificate's final extension, which will be `id-ce-embeddedSCT-CTv1`.

7. Remove all but the first `subjectExtensionCount` extensions in the TBSCertificate. Let `outerIssuerExtensions` be the removed extensions.

8. Check if `outerIssuerExtensions` and the AlternateIssuerInfo's `issuerExtensions` contain known subject extensions, as described in {{known-subject-extensions}}. If any are found, abort this procedure.

9. Append the AlternateIssuerInfo's `issuerExtensions` to the TBSCertificate.

10. Append a non-critical outer certificate binding extension ({{outer-certificate-binding}}) to the TBSCertificate such that:

    * `serialNumber` is `outerSerialNumber`.
    * `signatureAlgorithm` is `outerSignatureAlgorithm`.
    * `issuer` is `outerIssuer`.
    * `subjectExtensionCount` is the AlternateIssuerInfo's `subjectExtensionCount` value.
    * `issuerExtensions` is `outerIssuerExtensions`.

11. Replace the Certificate's `signatureValue` field with the AlternateIssuerInfo's `signatureValue`.

## Known Subject Extensions

{{outer-certificate-binding}} places requirements on alternate CAs that certify an OuterCertificateBinding. Relying parties are expected to ignore unrecognized non-critical extensions, so a relying party cannot check these requirements in full.

Instead, relying parties SHOULD check that `outerIssuerExtensions`, as computed in {{reconstructing-the-alternate-certificate}}, and the AlternateIssuerInfo's `issuerExtensions` contain no known subject extensions. Relying parties MUST allow unrecognized extensions for purposes of this check. (If the unrecognized extension is critical, it will likely be rejected later during path validation.)

Relying parties are not required to replicate the entire registry in {{iana-extension-types}}. If the relying party does not implement some subject extension, it MAY ignore it in `outerIssuerExtensions` and `issuerExtensions`. If the relying party implements some extension which does not appear in the registry, it MAY ignore it in this check if the extension does not interact with any issuer-specific constraint during path validation.

See {{certificate-mismatch}} for further discussion.

# Deployment Considerations

## Legacy Services

Embedded alternate issuers are a temporary transitionary mechanism for deployments where some legacy services do not support multiple certificates. Services that support multiple certificates directly SHOULD NOT use this mechanism. Deployments are RECOMMENDED to enforce this in CAs and relying parties.

For example, in a post-quantum transition for TLS, embedded alternate issuers are only applicable to unmodified legacy services that only have traditional end-entity keys. Once a service has a post-quantum end-entity key, it can be assumed to be relatively up-to-date and necessarily maintains separate end-entity keys for both legacy and post-quantum relying parties. Such a service would have no need for embedded alternate issuers. Accordingly, in a TLS deployment using embedded alternate issuers for a post-quantum transition:

* CAs SHOULD NOT issue embedded alternate issuers if the end-entity key is post-quantum.
* Relying parties SHOULD ignore embedded alternate issuers if the end-entity key is post-quantum.

This ensures that, when the post-quantum transition is completed, there will be no embedded alternate issuers remaining and implementations can remove the supporting logic.

A given deployment SHOULD only deploy embedded alternate issuers once, for only one transition. After that transition has completed, the deployment SHOULD have transitioned to a state where certificate holders support multiple certificates.

## Certificate Transparency Log Sizes

In deployments that use Certificate Transparency (CT) {{!RFC6962}}, a direct post-quantum transition risks significantly increasing the sizes of CT logs. ({{!RFC6962}} was obsoleted by {{?RFC9162}}, but the two protocols are incompatible. This document uses the definitions from {{!RFC6962}} for compatibility with existing deployments.) {{IETF124-PLANTS}} estimated an 8.3x size increase from migrating to ML-DSA-44. While {{?I-D.ietf-plants-merkle-tree-certs}} mitigates this, this mechanism risks logging the large AlternateIssuerInfo structures in CT logs when the outer certificate is logged.

{{alternate-certificate-information}} partially mitigates this by embedding AlternateIssuerInfo structures in SCTs. The outer certificate's precertificate omits all SCTs and thus the added post-quantum structures. Logged outer certificate precertificates thus do not increase in size. Alternate certificates, if using {{?I-D.ietf-plants-merkle-tree-certs}}, will be unaffected by this size increase. However, when the final certificate with embedded SCTs is logged, the AlternateIssuerInfo will be logged.

To mitigate this, deployments using embedded alternate issuers to transition away from a CT-based PKI thus SHOULD permit CT logs to reject logging of any certificates with an embedded AlternateIssuerInfoList. This will not impact other embedded SCTs, which refer to logged precertificates rather than logged final certificates.

# Security Considerations

## Validation

Although each alternate certificate is ultimately encoded relative to the outer certificate, the alternate CA's signature is on the reconstructed alternate TBSCertificate, so it is still considered to have issued the reconstructed alternate certificate. The alternate CA MUST perform all checks it would otherwise have performed in the course of issuing a certificate, such as domain validation. Misissuance of an alternate certificate remains misissuance.

## Certificate Mismatch

Relying parties process embedded alternate issuers during path building and path validation and potentially validate an alternate certificate instead of the outer certificate. The service presenting the certificate or, potentially, some other components of the relying party might be unaware that a different target certificate was ultimately validated. This presents a security risk if the component assumes the relying party validated the outer certificate.

If a relying party's path validation component accepted an alternate certificate, subsequent access checks in the relying party SHOULD act on the alternate certificate instead of the outer certificate. However, this might not be possible in some cases:

* An application's path validation component might be updated independently from the rest of the application.
* Path validation implementations MAY limit alternate certificate support to applications that opt in, but this might slow down PKI transitions that use this mechanism.
* A large application might also opt-in, but forget to update some component.

This document further mitigates this risk in two ways:

1. Limiting alternate certificates to encoding the same subject information as the outer certificate.
2. Requiring the alternate CA to sign over removed or replaced outer certificate fields, via the outer certificate binding extension.

This contrasts with {{?I-D.bonnell-lamps-chameleon-certs}} and {{?I-D.ietf-lamps-certdiscovery}}, which allow the second certificate to differ significantly. There, the application is expected to be fully aware of the second certificate.

If a more general protocol were instead used in this document, path validation might, for example, validate an alternate certificate's public key, while the rest of the application acts on a different, unchecked public key in the outer certificate. This would allow an attacker to bypass certificate-based authentication. This is not possible here because alternate certificates cannot replace the `subjectPublicKeyInfo` field.

X.509 extensions complicate this mitigation. X.509 does not distinguish between subject and issuer extensions, so this document introduces a classification in {{extension-types}} and {{iana-extension-types}}. However, extensibility introduces the risk that not all components of a system are aware of the classification of every extension.

This is primarily mitigated by the outer certificate binding extension. Except for `id-ce-embeddedSCT-CTv1`, extensions added or removed from the outer certificate are still signed by the alternate CA, so relying parties can hold the alternate CA responsible for their contents. In particular, {{outer-certificate-binding}} makes the alternate CA responsible for ensuring all such extensions are issuer extensions. CAs are expected to not sign unknown extensions, so there is no conflict with extensibility.

However, there are some cases where the relying party constrains the CA. If the alternate CA were under, say, a name constraint ({{Section 4.2.1.10 of !RFC5280}}), the relying party could not rely on the alternate CA alone to mitigate subjectAltName mismatches. If the alternate certificate contained a permitted subjectAltName, while the outer certificate contained an excluded one, a mismatch could bypass the name constraint. {{known-subject-extensions}} recommends that the relying party also check extension classification. Though the relying party might not recognize all extensions, its path validator necessarily recognizes all extensions that it incorporates into path validation.

# IANA Considerations

## PKIX Module Identifier

IANA is requested to add the following entry in the "SMI Security for PKIX Module Identifier" registry {{!RFC7299}}:

| Decimal | Description                          | References |
|---------|--------------------------------------|------------|
| TBD1    | id-mod-embeddedAlternateIssuers-2026 | [this-RFC] |

## PKIX Certificate Extension

IANA is requested to add the following entry in the "SMI Security for PKIX Certificate Extension" registry {{!RFC7299}}:

| Decimal | Description                   | References |
|---------|-------------------------------|------------|
| TBD2    | id-pe-outerCertificateBinding | [this-RFC] |

## Extension Types for Embedded Alternate Issuers {#iana-extension-types}

IANA is requested to add a new registry, "Extension Types for Embedded Alternate Issuers", under the top-level registry "Public Key Infrastructure using X.509 (PKIX) Parameters" at <https://www.iana.org/assignments/pkix-parameters>. The registration policy is Specification Required {{!RFC8126}}.

The registry contains the following note:

> See {{extension-types}} and {{certificate-mismatch}} in [this-RFC] for guidance on how to classify extensions for this registry.

The registry initially consists of:

| OID                     | Description                      | Extension Type | References              |
|-------------------------|----------------------------------|----------------|-------------------------|
| 1.3.6.1.4.1.11129.2.4.2 | id-ce-embeddedSCT-CTv1           | Issuer         | [this-RFC], {{RFC9162}} |
| 1.3.6.1.5.5.7.1.1       | id-pe-authorityInfoAccess        | Issuer         | [this-RFC], {{RFC5280}} |
| 1.3.6.1.5.5.7.1.11      | id-pe-subjectInfoAccess          | Subject        | [this-RFC], {{RFC5280}} |
| 1.3.6.1.5.5.7.1.TBD2    | id-pe-outerCertificateBinding    | Issuer         | [this-RFC]              |
| 1.3.101.75              | id-ce-transparencyInfo           | Issuer         | [this-RFC], {{RFC9162}} |
| 2.5.29.9                | id-ce-subjectDirectoryAttributes | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.14               | id-ce-subjectKeyIdentifier       | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.15               | id-ce-keyUsage                   | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.17               | id-ce-subjectAltName             | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.18               | id-ce-issuerAltName              | Issuer         | [this-RFC], {{RFC5280}} |
| 2.5.29.19               | id-ce-basicConstraints           | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.30               | id-ce-nameConstraints            | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.31               | id-ce-cRLDistributionPoints      | Issuer         | [this-RFC], {{RFC5280}} |
| 2.5.29.32               | id-ce-certificatePolicies        | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.33               | id-ce-policyMappings             | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.35               | id-ce-authorityKeyIdentifier     | Issuer         | [this-RFC], {{RFC5280}} |
| 2.5.29.36               | id-ce-policyConstraints          | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.37               | id-ce-extKeyUsage                | Subject        | [this-RFC], {{RFC5280}} |
| 2.5.29.46               | id-ce-freshestCRL                | Issuer         | [this-RFC], {{RFC5280}} |
| 2.5.29.54               | id-ce-inhibitAnyPolicy           | Subject        | [this-RFC], {{RFC5280}} |

--- back

# ASN.1 Module

This module uses structures defined in {{!RFC5912}} and {{!RFC9162}}.

~~~ asn.1
EmbeddedAlternateIssuers-2026
  { iso(1) identified-organization(3) dod(6) internet(1)
    security(5) mechanisms(5) pkix(7) id-mod(0)
    id-mod-embeddedAlternateIssuers-2026(TBD1) }

DEFINITIONS IMPLICIT TAGS ::=
BEGIN

IMPORTS
  SIGNATURE-ALGORITHM, AlgorithmIdentifier{}
  FROM AlgorithmInformation-2009 -- in [RFC5912]
    { iso(1) identified-organization(3) dod(6) internet(1)
      security(5) mechanisms(5) pkix(7) id-mod(0)
      id-mod-algorithmInformation-02(58) }
  Extension{}, EXTENSION
  FROM PKIX-CommonTypes-2009 -- in [RFC5912]
    { iso(1) identified-organization(3) dod(6) internet(1)
      security(5) mechanisms(5) pkix(7) id-mod(0)
      id-mod-pkixCommon-02(57) }
  ext-AuthorityKeyIdentifier, ext-IssuerAltName,
  ext-CRLDistributionPoints, ext-FreshestCRL,
  ext-AuthorityInfoAccess
  FROM PKIX1Implicit-2009 -- in [RFC5912]
    { iso(1) identified-organization(3) dod(6) internet(1)
      security(5) mechanisms(5) pkix(7) id-mod(0)
      id-mod-pkix1-implicit-02(59) }
  Name, CertificateSerialNumber, SignatureAlgorithms
  FROM PKIX1Explicit-2009 -- in [RFC5912]
    { iso(1) identified-organization(3) dod(6) internet(1)
      security(5) mechanisms(5) pkix(7) id-mod(0)
      id-mod-pkix1-explicit-02(51) }
  ext-embeddedSCT-CTv1, ext-transparencyInfo
  FROM CertificateTransparencyV2Module-2021 -- in [RFC9162]
    { iso(1) identified-organization(3) dod(6) internet(1)
      security(5) mechanisms(5) pkix(7) id-mod(0)
      id-mod-public-notary-v2(102) }
;

AlternateIssuerInfo ::= SEQUENCE {
    serialNumber          CertificateSerialNumber,
    signatureAlgorithm    AlgorithmIdentifier{SIGNATURE-ALGORITHM,
                              {SignatureAlgorithms}},
    issuer                Name,
    subjectExtensionCount INTEGER (0..MAX),
    issuerExtensions      SEQUENCE OF Extension{{IssuerExtensions}},
    signatureValue        BIT STRING
}

AlternateIssuerInfoList ::= SEQUENCE OF AlternateIssuerInfo

IssuerExtensions EXTENSION ::= {
    ext-AuthorityKeyIdentifier | ext-IssuerAltName |
    ext-CRLDistributionPoints | ext-FreshestCRL |
    ext-AuthorityInfoAccess |
    ext-embeddedSCT-CTv1 | ext-transparencyInfo |
    ext-outerCertificateBinding,
    ...
}

OuterCertificateBinding ::= SEQUENCE {
    serialNumber          CertificateSerialNumber,
    signatureAlgorithm    AlgorithmIdentifier{SIGNATURE-ALGORITHM,
                              {SignatureAlgorithms}},
    issuer                Name,
    subjectExtensionCount INTEGER (0..MAX),
    issuerExtensions      SEQUENCE OF Extension{{IssuerExtensions}}
}

id-pe-outerCertificateBinding OBJECT IDENTIFIER ::= {
    iso(1) identified-organization(3) dod(6) internet(1) security(5)
    mechanisms(5) pkix(7) pe(1) TBD2 }

ext-outerCertificateBinding EXTENSION ::= {
    SYNTAX OuterCertificateBinding
    IDENTIFIED BY id-pe-outerCertificateBinding
    CRITICALITY FALSE
}

END
~~~

# Uncoordinated Issuance

{{issuing-certificates}} assumes the outer CA and each alternate CA can coordinate. This appendix considers an alternate CA that wishes to produce an embedded alternate issuer certificate without a cooperating outer CA.

Recall the two relying parties the construction serves. A legacy relying party has no knowledge of this mechanism. It treats the embedded alternate information as an unrecognized non-critical extension and validates the outer certificate in the usual way ({{verifying-certificates}}), against the classical CAs in its trust store. An updated relying party instead reconstructs the alternate certificate ({{reconstructing-the-alternate-certificate}}) and validates it against the alternate CA, relying on the outer certificate's issuer for nothing.

The alternate CA can produce the alternate certificate unaided: it certifies the subject with its own key and appends the outer certificate binding. What it cannot produce unaided is the outer certificate, which a legacy relying party accepts only if it is signed by a classical CA that relying party already trusts.

An uncoordinated alternate CA therefore needs the private key of such a classical CA. This requirement is weaker than it appears, because an un-updatable classical relying party goes on trusting classical CAs whose private keys have been exposed. An updated relying party does not rely on the outer CA at all, having validated the alternate certificate instead. The outer CA's sole function is to be trusted by the parties that cannot tell the difference. A classical CA whose private key has leaked, and which legacy relying parties still trust and cannot exclude, is thus not merely adequate for the role but ideal: trusted exactly where it must be, and irrelevant everywhere else.

Generically, any key compromise affecting a CA that remains in an un-updatable classical relying party's trust store can be used in this way. Because such a relying party can neither remove a compromised CA from its store nor receive an exclusion blocking it -- the inability to update being the premise of this document -- every unremedied compromise it still trusts is a candidate outer CA. The construction does not depend on any particular incident; the eMudhra disclosure {{EMUDHRA}} is merely a convenient recent source of keys, used for the examples below.

## Example Outer Certificate Authorities

The certificates below are the eight eMudhra intermediate CA certificates whose private keys were disclosed in {{EMUDHRA}}, reproduced as issued. Each chains to emSign ECC Root CA - G3 through emSign Root TLS CA - G3 or emSign Root CS CA - G3. The stated scalar is the private key for each. An uncoordinated alternate CA can use any of them as the outer CA of {{uncoordinated-issuance}}.

C = NL, O = eMudhra BV, CN = EM DV TLS CA - G3B (private key scalar: 156):

~~~
-----BEGIN CERTIFICATE-----
MIIDaDCCAu6gAwIBAgIQAKSu5njyM2K0wORgt3LyFDAKBggqhkjOPQQDAzBWMQsw
CQYDVQQGEwJJTjElMCMGA1UECgwcZU11ZGhyYSBUZWNobm9sb2dpZXMgTGltaXRl
ZDEgMB4GA1UEAwwXZW1TaWduIFJvb3QgVExTIENBIC0gRzMwHhcNMjUwNDMwMTU1
NDA4WhcNNDAwNDI4MTgzMDAwWjA/MQswCQYDVQQGEwJOTDETMBEGA1UECgwKZU11
ZGhyYSBCVjEbMBkGA1UEAwwSRU0gRFYgVExTIENBIC0gRzNCMHYwEAYHKoZIzj0C
AQYFK4EEACIDYgAE557ZwEqXc8RzXlERHahawr1eZKs3S4DhLLwgOOweTw47gS/J
hZ4eZ6+wTi+LyX90hbplqpEe6KTkW2LVzAPVa+Z4s8OMtNyTfKJGDiOEU8iSk10p
1E+R4VYGtjzO2rRoo4IBljCCAZIwHwYDVR0jBBgwFoAU4TqrIXzx401UrtRyt/2L
WUIgkFQwHQYDVR0OBBYEFOIeL+ccGTkOVUtiwSAkVzUkiiUUMB0GA1UdJQQWMBQG
CCsGAQUFBwMBBggrBgEFBQcDAjBdBgNVHSAEVjBUMDkGCysGAQQBg44hAQABMCow
KAYIKwYBBQUHAgEWHGh0dHA6Ly9yZXBvc2l0b3J5LmVtc2lnbi5jb20wDQYLKwYB
BAGDjiEBAmQwCAYGZ4EMAQIBMHUGCCsGAQUFBwEBBGkwZzAkBggrBgEFBQcwAYYY
aHR0cDovL29jc3AtYi5lbXNpZ24uY29tMD8GCCsGAQUFBzAChjNodHRwOi8vcmVw
b3NpdG9yeS5lbXNpZ24uY29tL2NlcnRzL0VNRFZUTFNDQUczQi5wN2MwNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5lbXNpZ24uY29tLz9yb290dGxzY2FnMy5j
cmwwDgYDVR0PAQH/BAQDAgEGMBIGA1UdEwEB/wQIMAYBAf8CAQAwCgYIKoZIzj0E
AwMDaAAwZQIwLmsGXi6oFMJ2klkcEMXMug6gvxQWZWkewiDWVzQyl2L6z6DuQfPO
Ibh1grvdULDCAjEAxZvLlxUavG69J6wnSjC8XBbqNIQsJ6CE2QdNOe84Ptxb6VG8
gt2TlfJbHii+KNvK
-----END CERTIFICATE-----
~~~

C = US, O = eMudhra Inc, CN = EM DV TLS CA - G3C (private key scalar: 230):

~~~
-----BEGIN CERTIFICATE-----
MIIDaDCCAu6gAwIBAgIPWIsuPCu4zaE+GhIJJUMhMAoGCCqGSM49BAMDMFYxCzAJ
BgNVBAYTAklOMSUwIwYDVQQKDBxlTXVkaHJhIFRlY2hub2xvZ2llcyBMaW1pdGVk
MSAwHgYDVQQDDBdlbVNpZ24gUm9vdCBUTFMgQ0EgLSBHMzAeFw0yNTA0MzAxNTM1
MDlaFw00MDA0MjgxODMwMDBaMEAxCzAJBgNVBAYTAlVTMRQwEgYDVQQKDAtlTXVk
aHJhIEluYzEbMBkGA1UEAwwSRU0gRFYgVExTIENBIC0gRzNDMHYwEAYHKoZIzj0C
AQYFK4EEACIDYgAEOnp4YsjETTJN3WkDOmeCwO/u/kBV5ooSP6A3G6O9kZRRTuFy
pDPjYw8CvRiAN9nlFJ+bd9uUooHfmHsv6oiLdL3+j9pMEJnubKGVZv4+8OX2sJpV
bMyLpQVh3xsqRB0Bo4IBljCCAZIwHwYDVR0jBBgwFoAU4TqrIXzx401UrtRyt/2L
WUIgkFQwHQYDVR0OBBYEFK5+EFw8KZbBqEcnBVnx4D3U5QrzMB0GA1UdJQQWMBQG
CCsGAQUFBwMBBggrBgEFBQcDAjBdBgNVHSAEVjBUMDkGCysGAQQBg44hAQABMCow
KAYIKwYBBQUHAgEWHGh0dHA6Ly9yZXBvc2l0b3J5LmVtc2lnbi5jb20wDQYLKwYB
BAGDjiEBAmQwCAYGZ4EMAQIBMHUGCCsGAQUFBwEBBGkwZzAkBggrBgEFBQcwAYYY
aHR0cDovL29jc3AtYy5lbXNpZ24uY29tMD8GCCsGAQUFBzAChjNodHRwOi8vcmVw
b3NpdG9yeS5lbXNpZ24uY29tL2NlcnRzL0VNRFZUTFNDQUczQy5wN2MwNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5lbXNpZ24uY29tLz9yb290dGxzY2FnMy5j
cmwwDgYDVR0PAQH/BAQDAgEGMBIGA1UdEwEB/wQIMAYBAf8CAQAwCgYIKoZIzj0E
AwMDaAAwZQIxAMEN2HWMocbuWdy1d33pH6JpMvP/V/DIS5PtZbv1Q7TXEbxrcyxg
yuddOJscYDJJlAIwB4sH1BpbScDzkdD4tJBSwrG/1Mx82vb8YaySehNuu1+1oXNd
AEKwaJ0mVzScy2+6
-----END CERTIFICATE-----
~~~

C = NL, O = eMudhra BV, CN = EM EV TLS CA - G3B (private key scalar: 115):

~~~
-----BEGIN CERTIFICATE-----
MIIDaDCCAu2gAwIBAgIQAMfEHv6S2CXwXq6EGkqCwTAKBggqhkjOPQQDAzBWMQsw
CQYDVQQGEwJJTjElMCMGA1UECgwcZU11ZGhyYSBUZWNobm9sb2dpZXMgTGltaXRl
ZDEgMB4GA1UEAwwXZW1TaWduIFJvb3QgVExTIENBIC0gRzMwHhcNMjUwNDMwMTYw
MzMxWhcNNDAwNDI4MTgzMDAwWjA/MQswCQYDVQQGEwJOTDETMBEGA1UECgwKZU11
ZGhyYSBCVjEbMBkGA1UEAwwSRU0gRVYgVExTIENBIC0gRzNCMHYwEAYHKoZIzj0C
AQYFK4EEACIDYgAEJduoWmTTl9oLEFLp6lRuSHvv6pydnWigxerc/u91ppu11vwB
9Cv7ym60C81kwHZ+Nv/QXZeoE8WqDGyVMOec9fCGZGjxOAZCCqXIUYQLHiUlQOeQ
5LZUw4Uu1krx6GV9o4IBlTCCAZEwHwYDVR0jBBgwFoAU4TqrIXzx401UrtRyt/2L
WUIgkFQwHQYDVR0OBBYEFD7cINmJRpiHe4c/2IFHslwV2EcGMB0GA1UdJQQWMBQG
CCsGAQUFBwMBBggrBgEFBQcDAjBcBgNVHSAEVTBTMDkGCysGAQQBg44hAQABMCow
KAYIKwYBBQUHAgEWHGh0dHA6Ly9yZXBvc2l0b3J5LmVtc2lnbi5jb20wDQYLKwYB
BAGDjiEBAngwBwYFZ4EMAQEwdQYIKwYBBQUHAQEEaTBnMCQGCCsGAQUFBzABhhho
dHRwOi8vb2NzcC1iLmVtc2lnbi5jb20wPwYIKwYBBQUHMAKGM2h0dHA6Ly9yZXBv
c2l0b3J5LmVtc2lnbi5jb20vY2VydHMvRU1FVlRMU0NBRzNCLnA3YzA3BgNVHR8E
MDAuMCygKqAohiZodHRwOi8vY3JsLmVtc2lnbi5jb20vP3Jvb3R0bHNjYWczLmNy
bDAOBgNVHQ8BAf8EBAMCAQYwEgYDVR0TAQH/BAgwBgEB/wIBADAKBggqhkjOPQQD
AwNpADBmAjEA9Jq999oaOUKhFwopw33iCmt0OHIn3gjT43GfuOHR/YckXdwoNdDA
lIEhz0NgRO3/AjEAvY/a3OMvktIZLYjn+WLKmp0QhrIMV+e6EpS7j4GVeOKouNp6
MN/P+ByGeiRMghzB
-----END CERTIFICATE-----
~~~

C = US, O = eMudhra Inc, CN = EM EV TLS CA - G3C (private key scalar: 56):

~~~
-----BEGIN CERTIFICATE-----
MIIDaDCCAu6gAwIBAgIQAIAJkopkEHYBrwX6LxnxCTAKBggqhkjOPQQDAzBWMQsw
CQYDVQQGEwJJTjElMCMGA1UECgwcZU11ZGhyYSBUZWNobm9sb2dpZXMgTGltaXRl
ZDEgMB4GA1UEAwwXZW1TaWduIFJvb3QgVExTIENBIC0gRzMwHhcNMjUwNDMwMTUz
OTMxWhcNNDAwNDI4MTgzMDAwWjBAMQswCQYDVQQGEwJVUzEUMBIGA1UECgwLZU11
ZGhyYSBJbmMxGzAZBgNVBAMMEkVNIEVWIFRMUyBDQSAtIEczQzB2MBAGByqGSM49
AgEGBSuBBAAiA2IABC1W6Cmm+4LGSkaX2WCvlR6QZ66R+SbTQwbaSKiq63hptqZ+
bSdCbgRCqgOzbhmI3/Y2UU7hzwf1uks/8qBmRa59MdWGatMao9JG6wsxd+6Zy3Vm
xRbOdICKZo+R8MR/yqOCAZUwggGRMB8GA1UdIwQYMBaAFOE6qyF88eNNVK7Ucrf9
i1lCIJBUMB0GA1UdDgQWBBSmJV6AqU2385Z8G8eR35K87kvrXzAdBgNVHSUEFjAU
BggrBgEFBQcDAQYIKwYBBQUHAwIwXAYDVR0gBFUwUzA5BgsrBgEEAYOOIQEAATAq
MCgGCCsGAQUFBwIBFhxodHRwOi8vcmVwb3NpdG9yeS5lbXNpZ24uY29tMA0GCysG
AQQBg44hAQJ4MAcGBWeBDAEBMHUGCCsGAQUFBwEBBGkwZzAkBggrBgEFBQcwAYYY
aHR0cDovL29jc3AtYy5lbXNpZ24uY29tMD8GCCsGAQUFBzAChjNodHRwOi8vcmVw
b3NpdG9yeS5lbXNpZ24uY29tL2NlcnRzL0VNRVZUTFNDQUczQy5wN2MwNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5lbXNpZ24uY29tLz9yb290dGxzY2FnMy5j
cmwwDgYDVR0PAQH/BAQDAgEGMBIGA1UdEwEB/wQIMAYBAf8CAQAwCgYIKoZIzj0E
AwMDaAAwZQIwBy1wTnzH4lXDTy2PD91ysLjvnlVN3JbJmc1BIMEU8TOLkYVHibMc
9HeyonikFSfmAjEA6uE3QRxlrZTz/1/lXPTEMUhoauAtycagtlNFp79kjhvNa2/f
Aal1GNxsOutLnr0Y
-----END CERTIFICATE-----
~~~

C = NL, O = eMudhra BV, CN = EM OV CS CA - G3B (private key scalar: 66):

~~~
-----BEGIN CERTIFICATE-----
MIIDWzCCAuCgAwIBAgIPeBiTP4+FtGGaKzX7qD0KMAoGCCqGSM49BAMDMFUxCzAJ
BgNVBAYTAklOMSUwIwYDVQQKDBxlTXVkaHJhIFRlY2hub2xvZ2llcyBMaW1pdGVk
MR8wHQYDVQQDDBZlbVNpZ24gUm9vdCBDUyBDQSAtIEczMB4XDTI1MDQzMDE2MDgy
N1oXDTQwMDQyODE4MzAwMFowPjELMAkGA1UEBhMCTkwxEzARBgNVBAoMCmVNdWRo
cmEgQlYxGjAYBgNVBAMMEUVNIE9WIENTIENBIC0gRzNCMHYwEAYHKoZIzj0CAQYF
K4EEACIDYgAEqrsPsGZqz88B+tUr/P1MK/LLPDSX3nOVDlgZth/mlQmHQcjEkUOb
NKBDHK/+W/m0wcjorgAB+fTcitgsWfNjnpbNbD1NsM7+wmw9YlF7N3YTcnQ/jord
d7EAZPKr5eBQo4IBizCCAYcwHwYDVR0jBBgwFoAUaHQD+IXKjVRQVx0W4MjeRQ2r
zS0wHQYDVR0OBBYEFHn66OzP2w11jaRY1p+27dx8Sd7mMBMGA1UdJQQMMAoGCCsG
AQUFBwMDMF4GA1UdIARXMFUwOQYLKwYBBAGDjiEBAAEwKjAoBggrBgEFBQcCARYc
aHR0cDovL3JlcG9zaXRvcnkuZW1zaWduLmNvbTAOBgwrBgEEAYOOIQECgUgwCAYG
Z4EMAQQBMHQGCCsGAQUFBwEBBGgwZjAkBggrBgEFBQcwAYYYaHR0cDovL29jc3At
Yi5lbXNpZ24uY29tMD4GCCsGAQUFBzAChjJodHRwOi8vcmVwb3NpdG9yeS5lbXNp
Z24uY29tL2NlcnRzL0VNT1ZDU0NBRzNCLnA3YzA2BgNVHR8ELzAtMCugKaAnhiVo
dHRwOi8vY3JsLmVtc2lnbi5jb20vP3Jvb3Rjc2NhZzMuY3JsMA4GA1UdDwEB/wQE
AwIBBjASBgNVHRMBAf8ECDAGAQH/AgEAMAoGCCqGSM49BAMDA2kAMGYCMQC/zr03
YEELsNKqzUtH2dVFi4BaKFdyrr/6G1dIWp8KAVgsw2/u4lyyboUDaC9jDvICMQC9
HTxku84tNX9dmHOtH7aC2Nu/JM0jTf6KjZAvovCJzzv1IFWSZpVLZibynMLXOrM=
-----END CERTIFICATE-----
~~~

C = US, O = eMudhra Inc, CN = EM OV CS CA - G3C (private key scalar: 237):

~~~
-----BEGIN CERTIFICATE-----
MIIDWzCCAuGgAwIBAgIPVEt/kiwxriIKSpIpFvbmMAoGCCqGSM49BAMDMFUxCzAJ
BgNVBAYTAklOMSUwIwYDVQQKDBxlTXVkaHJhIFRlY2hub2xvZ2llcyBMaW1pdGVk
MR8wHQYDVQQDDBZlbVNpZ24gUm9vdCBDUyBDQSAtIEczMB4XDTI1MDQzMDE2Mzg0
MFoXDTQwMDQyODE4MzAwMFowPzELMAkGA1UEBhMCVVMxFDASBgNVBAoMC2VNdWRo
cmEgSW5jMRowGAYDVQQDDBFFTSBPViBDUyBDQSAtIEczQzB2MBAGByqGSM49AgEG
BSuBBAAiA2IABLnw4GNHAqacpqrdRDz9YsQbm6SyCSRpfXl052kaEm+5avs+NML9
O8X0gwYfJl3CpYQ2NHMjUQQ2so8E84YjYwZ5gOJQPgJTLWVGI1pxZaJZoe9Jb3Qb
X97neHII5Lj676OCAYswggGHMB8GA1UdIwQYMBaAFGh0A/iFyo1UUFcdFuDI3kUN
q80tMB0GA1UdDgQWBBTub3MpjW+/p7Mbw5U09hVWRHhmtzATBgNVHSUEDDAKBggr
BgEFBQcDAzBeBgNVHSAEVzBVMDkGCysGAQQBg44hAQABMCowKAYIKwYBBQUHAgEW
HGh0dHA6Ly9yZXBvc2l0b3J5LmVtc2lnbi5jb20wDgYMKwYBBAGDjiEBAoFIMAgG
BmeBDAEEATB0BggrBgEFBQcBAQRoMGYwJAYIKwYBBQUHMAGGGGh0dHA6Ly9vY3Nw
LWMuZW1zaWduLmNvbTA+BggrBgEFBQcwAoYyaHR0cDovL3JlcG9zaXRvcnkuZW1z
aWduLmNvbS9jZXJ0cy9FTU9WQ1NDQUczQy5wN2MwNgYDVR0fBC8wLTAroCmgJ4Yl
aHR0cDovL2NybC5lbXNpZ24uY29tLz9yb290Y3NjYWczLmNybDAOBgNVHQ8BAf8E
BAMCAQYwEgYDVR0TAQH/BAgwBgEB/wIBADAKBggqhkjOPQQDAwNoADBlAjEA8yPg
8sntOhLwV2QXJ5LEaKSpo/fnE3uOgTadK73fDhyjiqr0Io9GVuQTYqBPbQ2TAjBg
euApmBS486+iei03lRrp139dPw6k28jjGNvWsAU7A1N86J1zPJVDEyVH+nMAWC0=
-----END CERTIFICATE-----
~~~

C = NL, O = eMudhra BV, CN = EM OV TLS CA - G3B (private key scalar: 155):

~~~
-----BEGIN CERTIFICATE-----
MIIDZjCCAu2gAwIBAgIPbK2jZ/scEACIEg6iAtVoMAoGCCqGSM49BAMDMFYxCzAJ
BgNVBAYTAklOMSUwIwYDVQQKDBxlTXVkaHJhIFRlY2hub2xvZ2llcyBMaW1pdGVk
MSAwHgYDVQQDDBdlbVNpZ24gUm9vdCBUTFMgQ0EgLSBHMzAeFw0yNTA0MzAxNjAw
MTJaFw00MDA0MjgxODMwMDBaMD8xCzAJBgNVBAYTAk5MMRMwEQYDVQQKDAplTXVk
aHJhIEJWMRswGQYDVQQDDBJFTSBPViBUTFMgQ0EgLSBHM0IwdjAQBgcqhkjOPQIB
BgUrgQQAIgNiAARziXqg6zz9yosSQC/68KvE4SErr43ud4rJ7i1tuD+sKAxeBWOK
56Ks1xwyKo42HN659pxDArndjXPRzGyqsswK7k4hbOl53VGKSvS2JuVoTN1lgDft
DIVOTMEvTPhkbQ2jggGWMIIBkjAfBgNVHSMEGDAWgBThOqshfPHjTVSu1HK3/YtZ
QiCQVDAdBgNVHQ4EFgQUm9hlUa/n5yw3rQjuaRnZvzeErgowHQYDVR0lBBYwFAYI
KwYBBQUHAwEGCCsGAQUFBwMCMF0GA1UdIARWMFQwOQYLKwYBBAGDjiEBAAEwKjAo
BggrBgEFBQcCARYcaHR0cDovL3JlcG9zaXRvcnkuZW1zaWduLmNvbTANBgsrBgEE
AYOOIQECbjAIBgZngQwBAgIwdQYIKwYBBQUHAQEEaTBnMCQGCCsGAQUFBzABhhho
dHRwOi8vb2NzcC1iLmVtc2lnbi5jb20wPwYIKwYBBQUHMAKGM2h0dHA6Ly9yZXBv
c2l0b3J5LmVtc2lnbi5jb20vY2VydHMvRU1PVlRMU0NBRzNCLnA3YzA3BgNVHR8E
MDAuMCygKqAohiZodHRwOi8vY3JsLmVtc2lnbi5jb20vP3Jvb3R0bHNjYWczLmNy
bDAOBgNVHQ8BAf8EBAMCAQYwEgYDVR0TAQH/BAgwBgEB/wIBADAKBggqhkjOPQQD
AwNnADBkAjBAk+5BizdJhrED3wb7F2Gm7bWI60dlNd075nghT2TrR4NWKZtJZcbc
28xxIY/NONICMF6mPdsjosaqmvEMWqKcBh4vsIvXSKnQj5o8zrFB4oKS+accjYTe
8Dz3bQjauvy9uw==
-----END CERTIFICATE-----
~~~

C = US, O = eMudhra Inc, CN = EM OV TLS CA - G3C (private key scalar: 4):

~~~
-----BEGIN CERTIFICATE-----
MIIDaDCCAu6gAwIBAgIPKWkmwiejoVJjkrGUgqUEMAoGCCqGSM49BAMDMFYxCzAJ
BgNVBAYTAklOMSUwIwYDVQQKDBxlTXVkaHJhIFRlY2hub2xvZ2llcyBMaW1pdGVk
MSAwHgYDVQQDDBdlbVNpZ24gUm9vdCBUTFMgQ0EgLSBHMzAeFw0yNTA0MzAxNTM3
NDFaFw00MDA0MjgxODMwMDBaMEAxCzAJBgNVBAYTAlVTMRQwEgYDVQQKDAtlTXVk
aHJhIEluYzEbMBkGA1UEAwwSRU0gT1YgVExTIENBIC0gRzNDMHYwEAYHKoZIzj0C
AQYFK4EEACIDYgAEE4JRzVKskpjByKrZdzId65fnCb0LTKCspV3IrVHc/J0ViaFZ
fjpRIOHv1jHGPhg1ysrimGmmLhYx6KKBgatWYW3EXZGKvAnzqw5jz3kqpNztc4e+
N7ulaVSfHAKycO1no4IBljCCAZIwHwYDVR0jBBgwFoAU4TqrIXzx401UrtRyt/2L
WUIgkFQwHQYDVR0OBBYEFDTq6gzryQra+dDF4u2beMmHnhtsMB0GA1UdJQQWMBQG
CCsGAQUFBwMBBggrBgEFBQcDAjBdBgNVHSAEVjBUMDkGCysGAQQBg44hAQABMCow
KAYIKwYBBQUHAgEWHGh0dHA6Ly9yZXBvc2l0b3J5LmVtc2lnbi5jb20wDQYLKwYB
BAGDjiEBAm4wCAYGZ4EMAQICMHUGCCsGAQUFBwEBBGkwZzAkBggrBgEFBQcwAYYY
aHR0cDovL29jc3AtYy5lbXNpZ24uY29tMD8GCCsGAQUFBzAChjNodHRwOi8vcmVw
b3NpdG9yeS5lbXNpZ24uY29tL2NlcnRzL0VNT1ZUTFNDQUczQy5wN2MwNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5lbXNpZ24uY29tLz9yb290dGxzY2FnMy5j
cmwwDgYDVR0PAQH/BAQDAgEGMBIGA1UdEwEB/wQIMAYBAf8CAQAwCgYIKoZIzj0E
AwMDaAAwZQIxAMSZt2xsPeumRdeSMSQ3CbzIZeQ/q7GDCWfeE1Pq+mhgA6gRM44c
NVF3xJ+wsxJk+wIwFJbaDMokD8Ei6eNOiA8yUMYvfzQyngmrhZQMdPCK1SewrR79
4rSb9vGqxZzelQth
-----END CERTIFICATE-----
~~~

# Acknowledgments
{:numbered="false"}

Thanks to the authors of {{?I-D.bonnell-lamps-chameleon-certs}} and {{?I-D.ietf-lamps-certdiscovery}} for exploring the space of how to represent related certificates. While this document is intentionally more specialized, it has benefited significantly from their work.
