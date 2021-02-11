# ITE-6: Generalized link format

Author: lodato@google.com

Contributors/reviewers: dedic@google.com, nitinjain@google.com,
patricklawson@google.com, tomhennen@google.com, wietse@google.com

Date: September 2020

# Abstract

This ITE defines a new schema for in-toto link files, which are now generally
called "attestations." An attestation has three distinct layers, mapping to the
three distinct steps in verification. The innermost layer is user-defined to
allow customers to define their own schemas; "link" is now one such user-defined
schema.

This specification is developed jointly with
[Binary Authorization](https://cloud.google.com/binary-authorization), who will
support the agreed-upon format once finalized. This way a single link file will
be usable by either system.

# Specification

## Introduction

An **attestation** is the generalization of an in-toto link. It is a statement
about an artifact, signed by an attester. Each attestation has a type indicating
what the statement means, plus type-dependent details. In this new model, a
"link" is just one type of attestation, but others are possible.

Examples of attestations:

*   Provenance: GitHub Actions attests to the fact that it built a container
    image with digest "sha256:87f7fe…" from git commit "f0c93d…" in the "master"
    branch of "https://github.com/example/foo".
*   Code review: GitHub attests to the fact that Alice uploaded and Bob approved
    git commit "f0c93d…" in the "master" branch of
    "https://github.com/example/foo".
*   Test result: Google Container Analysis attests to the fact that no
    vulnerabilities were found in container image "sha256:87f7fe…" at a
    particular time.
*   Policy decision: Binary Authorization attests to the fact that container
    image "sha256:87f7fe…" is allowed to run under GKE project "example-project"
    within the next 4 hours, and that it used the three attestations above and
    as well as the policy with sha256 hash "79e572".

The benefit of this ITE is to express these attestations more natually than was
possible with the old in-toto link schema.

## Authentication and serialization

The authentication and serialization is handled by
[ITE-5](https://github.com/MarkLodato/ITE/blob/ite-5/ITE/5/README.md). The
payload is always JSON encoded with a `payloadType` of
`"https://in-toto.io/Attestation/v1-json"`. (In the future other encodings, such
as CBOR, may be defined.)

Prior to ITE-5, the existing in-toto signature wrapper may be used. In this
case, the payload is always JSON with a `_type` of
`"https://in-toto.io/Attestation/v1-json"`.

## General attestation schema

Every attestation contains the following fields:

<a id="attestation_type"></a>
`attestation_type` _string ([TypeURI]), required_

> URI representing the meaning of this attestation and how to interpret the rest
> of the fields. Example:
>
> ```json
> "attestation_type": "https://in-toto.io/Provenance/v1"
> ```
>
> (Somewhat similar to `name` in [in-toto 0.9].)

<a id="subject"></a>
`subject` _object ([ArtifactCollection]), required_

> The collection of software artifacts this attestation is about. Example:
>
> ```json
> "subject": {
>   "curl-7.72.0.tar.bz2": { "sha256": "ad9197…" },
>   "curl-7.72.0.tar.gz": { "sha256": "d4d589…" }
> }
> ```
>
> When there is a single artifact whose name is not meaningful, is RECOMMENDED
> to use `"_"` as the name.
>
> (Roughly equivalent to `products` in [in-toto 0.9].)

<a id="materials"></a>
`materials` _object ([ArtifactCollection]), optional_

> The collection of software artifacts that influenced the attestation, aside
> from the `subject` itself. Example:
>
> ```json
> "materials": {
>   "git+https://github.com/curl/curl@curl-7_72_0": { "git_commit": "9d954e4…" },
>   "pkg:deb/debian/stunnel4@5.50-3?arch=amd64": { "sha256": "e1731ae…" },
>   "pkg:deb/debian/python-impacket@0.9.15-5?arch=all": { "sha256": "71fa2e6…" },
>   "pkg:deb/debian/libzstd-dev@1.3.8+dfsg-3?arch=amd64": { "sha256": "91442b0…" },
>   "pkg:deb/debian/libbrotli-dev@1.0.7-2+deb10u1?arch=amd64": { "sha256": "05b6e46…" }
> }
> ```
>
> (Unchanged from [in-toto 0.9].)

## Field type definitions

<a id="TypeURI"></a>
_TypeURI (string)_

> Uniform Resource Identifier as specified in [RFC 3986], used as a
> collision-resistant type identifier. Case sensitive and MUST be case
> normalized as per section 6.2.2.1 of RFC 3986, meaning that the scheme and
> authority MUST be in lowercase. SHOULD resolve to a human-readable
> description, but MAY be unresolvable. SHOULD include a version number to allow
> for revisions. Example: `"https://in-toto.io/Attestation/v1"`.

<a id="ArtifactCollection"></a>
_ArtifactCollection (object)_

> A collection of software artifacts. Each key/value pair represents a single
> software artifact, such as a file, a container image, or a git commit.
>
> The key identifies the artifact relative to this attestation. It MUST be a URI
> or path-noscheme ([RFC 3986]). If it is a URI, it MUST be case normalized and
> SHOULD resolve to the artifact, but MAY be unresolvable. It is RECOMMENDED to
> use [Package URL][] (`pkg:`) or [SPDX Download Location][] (e.g.
> `git+https:`). If path-noscheme, it SHOULD represent a relative filesystem
> path.
>
> The value contains cryptographic digests of the artifact's content. It is a
> map from digest type to digest value (string), encoded as lowercase hex. The
> digest type unambiguously identifies the hash algorithm and how it is applied
> to the artifact. An artifact MUST be considered matching if *any* of its
> digests match. Verifiers MUST choose which digest types they accept and MUST
> ignore digest types they do not accept or recognize. The value MAY be empty or
> null, meaning that no digest is available.
>
> The following digests types are RECOMMENDED:
>
> *   Regular File: `sha256`
> *   Git repository: `git_commit`
> *   Mercurial repository: `hg_changeset`
> *   Container image: [`oci_image_id`][oci_image_id] and `oci_repo_digest`. It
>     is best to list both when available. The former is registry independent
>     (over the uncompressed manifest) while the latter depends on the registry
>     (computed over the compressed manifest). Example for
>     [docker.io/curlimages/curl:7.72.0 amd64](https://hub.docker.com/layers/curlimages/curl/7.72.0/images/sha256-3c3ff0c379abb1150bb586c7d55848ed4dcde4a6486b6f37d6815aed569332fe):
>     `{"oci_image_id": "sha256:d2d63f4...", "oci_repo_digest":
>     "sha256:3c3ff0c..."}`
>
> Additional pre-defined digest types: `sha224`, `sha384`, `sha512`,
> `sha512_224`, `sha512_256`, `sha3_224`, `sha3_256`, `sha3_384`, `sha3_512`,
> `shake128`, `shake256`, `blake2b`, `blake2s`, `md5` (DISCOURAGED), `sha1`
> (DISCOURAGED).
>
> Custom digest types MAY be used if none of the recommended or pre-defined
> digest types work.

<a id="Timestamp"></a>
_Timestamp (string)_

> A point in time, represented as a string in [RFC 3339] format in the UTC time
> zone ("Z"). Example: `"1985-04-12T23:20:50.52Z"`.

## Provenance schema

```json
"attestation_type": "https://in-toto.io/Provenance/v1"
```

A provenance-type attestation explains the process that produced the `subject`.
The `materials` SHOULD include all artifacts that influenced the build,
including sources, dependencies, build tools, base images, and so on.

Provenance attestations have the following type-specific fields:

<a id="builder"></a>
`builder` _object, required_

> Idenfifies the entity that executed the build steps. Example:
>
> ```json
> "builder": {
>   "id": "https://github.com/Attestations/GitHubHostedActions@v1"
> }
> ```
>
> This is distinct from the signer because one signer may generate attestations
> for more than one builder. For example, a signle GitHubActions signer may
> produce attestations for both "github-hosted runner" and various "self-hosted
> runner" builders.
>
> Even though it may be implicit from the signer, it is required to aid
> readability and debugging.
>
> Verifiers MUST only accept specific builders from specific signers.

<a id="builder.id"></a>
`builder.id` _string ([TypeURI]), required_

> URI indicating the builder's identity.

<a id="recipe"></a>
`recipe` _object, optional_

> Describes the actions that the builder performed. Example:
>
> ```json
> "recipe": {
>   "type": "https://github.com/Attestations/GitHubActionsWorkflow@v1",
>   "material": "git+https://github.com/curl/curl@curl-7_72_0",
>   "entry_point": "build.yaml:maketgz",
>   "arguments": null
> }
> ```
>
> MAY be omitted if unknown or implicit from the builder.

<a id="recipe.type"></a>
`recipe.type` _string ([TypeURI]), required_

> URI indicating what type of recipe was performed. It determines the meaning
> of `recipe.entry_point`, `recipe.arguments`, `materials`, and
> `reproducibility`.

<a id="recipe.material"></a>
`recipe.material` _string, optional_

> Key in `materials` containing the recipe steps that are not implied by
> `recipe.type`. For example, if the recipe type were "make", then this would
> point to the source containing the Makefile, not the `make` program itself.
>
> Omit this field (or use null) if the recipe doesn't come from a material.
>
> TODO: What if there is more than one material?

<a id="recipe.entry_point"></a>
`recipe.entry_point` _string, optional_

> String identifying the entry point. The meaning is defined by `recipe.type`.
> For example, if the recipe type were "make", then this would reference the
> directory in which to run `make` as well as which target to use.
>
> MAY be omitted if the recipe type specifies a default value.

<a id="recipe.arguments"></a>
`recipe.arguments` _object, optional_

> Collection of input arguments that influenced the build on top of
> `recipe.material` and `recipe.entry_point`. The schema is defined by
> `recipe.type`.
>
> Omit this field (or use null) to indicate "no arguments."

<a id="reproducibility"></a>
`reproducibility` _object, optional_

> Other information that is needed to reproduce the build but that cannot be
> controlled by users. The schema is determined by `recipe.type`.

<a id="metadata"></a>
`metadata` _object, optional_

> Other properties of the build.

<a id="metadata.build_timestamp"></a>
`metadata.build_timestamp` _string ([Timestamp]), optional_

> The timestamp of when the build occurred.

<a id="metadata.materials_complete"></a>
`metadata.materials_complete` _boolean, optional_

> If true, `materials` is claimed to be complete, usually through some
> controls to prevent network access.

## Link type

Going forward, it is expected that most applications will use a more specific
`attestation_type` with its own schema rather than relying on a generic Link
schema.

That said, we define a backwards compatible Link format that is isomorphic with
the old format and supported by existing layouts:

Old Field Name  | New Field Name
--------------- | -----------------------------------------------
`_type: "link"` | `attestation_type: "https://in-toto.io/Link/v1"`
`products`      | `subject`
`materials`     | `materials`
`name`          | `name`
`command`       | `command`
`byproducts`    | `byproducts`
`environment`   | `environment`

## Notes

Attestations SHOULD be designed to encourage policies to be "monotonic," meaning
that deleting an attestation will never turn a DENY decision into an ALLOW. One
reason for this is because verifiers MUST ignore unrecognized subject digest
types; if no subject is recognized, the attestation is effectively deleted.
Example: instead of "deny if a 'has vulnerabilities' attestation exists", prefer
"deny unless a 'no vulnerabilities' attestation exists".

Attestation designers are free to limit what subject types are valid for a given
attestation type. For example, suppose a "Gerrit code review" attestation only
applies to `git_commit` subjects. In that case, a producer of such attestations
should never use a subject other than `git_commit`.

# Examples

## Custom-type attestations

In many cases, custom-type attestations would be a more natural fit, as shown
below. Such custom attestations are not yet supported by in-toto because the
layout format has no way to reference such attestations. Still, we show the
examples to explain the benefits for the new link format.

The initial step is often to write code. This has no materials and no real
command. The existing Link schema has little benefit. Instead, a custom
`attestation_type` would avoid all of the meaningless boilerplate fields. This
example also shows the use of a `git_commit` artifact type.

```json
{
  "attestation_type": "https://example.com/WriteCode/v1",
  "subject": {
    "git+https://github.com/example/my-project@master": {
      "git_commit": "859b387b985ea0f414e4e8099c9f874acb217b94"
    }
  }
}
```

Test results are also an awkward fit for the Link schema, since the subject is
really the materials, not the products. Again, a custom `attestation_type` is a
better fit:

```json
{
  "attestation_type": "https://example.com/TestResult/v1",
  "subject": {
    "_": {
      "git_commit": "859b387b985ea0f414e4e8099c9f874acb217b94"
    }
  },
  "passed": true
}
```

# Motivating use case

**TODO: This section has not yet been updated for latest schema.**

MyCompany wants to centrally enforce the following rules of its production
Kubernetes environments:

*   All containers must undergo a source-level vulnerability scan showing zero
    known high severity vulnerabilities.
*   All first-party code must be peer reviewed, reside in MyCompany's GitHub
    org, and be sufficiently recent.
*   All third-party code and build tools must be verified via Reproducible
    Builds. (Let's pretend such an attestation service exists.)
*   All build steps must be performed by GitHub Actions, Google Cloud Build, or
    AWS CodeBuild in (a hypothetical) "hermetic" mode.
*   The intermediate products in the supply chain have not been tampered with.

It is both too costly and too insecure to have every team write their own
layout. There are several hundred different Kuberenetes environments
administered by many different product teams, none of whom employ security
experts. Instead, we need a solution that allows the central security team to
write a policy that automatically applies to every environment across the
company.

The current in-toto link and layout formats are impractical for this
application:

*   It is awkward to express these concepts in the current link format. One
    would need to either record the exact command lines used, which is too
    brittle, or ignore all of the standard fields and jam everything in
    `environment`, which is hard to use.
*   It is impossible to express this policy in the current layout format.
    *   There is no support for verifying any details. The closest option,
        `expected_command`, is just a warning but not an error.
    *   There is no support for performing generic traversals of the build
        graph, such as "allow any number of verifiable build steps."
*   There is no practical way to analyze a layout to determine if it meets the
    requirements above.

The proposed attestation format, along with a future policy engine, allows us to
craft such a policy. This ITE does not cover the policy engine piece, but we
show the ideas via pseudocode.

## Policy pseudocode

The following pseudocode implements the policy above. Assume that memoization
takes care of cycles. This policy would be written by a security expert at the
company and used for all Kubernetes environments.

```python
policy(artifact):
  lookup attestations for artifact
  allow if (any attestation meets vulnerability_scan and
            any attestation meets first_party_code_review)
  for each attestation meeting verifiable_build:
    allow if (every 'top_level_source' relation meets good_top_level_source and
              every 'dependent_sources' relation meets good_dependent_source and
              every 'tool' relation meets good_tool)
  deny otherwise

good_top_level_source(relation):
  return policy(relation.artifact)

good_dependent_source(relation):
  lookup attestations for relation.artifact
  allow if any attestation meets first_party_code_review
  deny otherwise

good_tool(relation):
  lookup attestations for relation.artifact
  allow if any attestation (meets reproducible_build and
                            attestation.details.name == relation.name)
  deny otherwise

vulnerability_scan(attestation):
  attestation is signed by 'MyCompanyScanner'
  attestation.attestation_type == 'https://example.com/VulnerabilityScan/v1'
  attestation.details.vulnerability_counts.high == 0
  attestation.details.timestamp is within 14 days of today

first_party_code_review(attestation):
  attestation is signed by 'GitHub'
  attestation.attestation_type == 'https://example.com/CodeReview/v1'
  attestation.details.repo_url starts with 'https://github.com/my-company/'
  attestation.details.code_reviewed == true
  attestation.details.timestamp is within 30 days of today

reproducible_build(attestation):
  attestation is signed by 'ReproducibleBuilds'
  attestation.attestation_type == 'https://example.com/ReproducibleBuild/v1'

verifiable_build(attestation):
  return (hermetic_github_action(attestation) or
          hermetic_cloud_build(attestation) or
          hermetic_codebuild(attestation))

hermetic_github_action(attestation):
  attestation is signed by 'GitHubActions'
  attestation.attestation_type == 'https://example.com/GitHubActionProduct/v1'
  attestation.details.hermetic == true

hermetic_cloud_build(attestation):
  attestation is signed by 'GoogleCloudBuild'
  attestation.attestation_type == 'https://example.com/GoogleCloudBuildProduct/v1'
  attestation.details.no_network == true

hermetic_cloud_build(attestation):
  attestation is signed by 'AwsCodeBuild'
  attestation.attestation_type == 'https://example.com/AwsCodeBuildProduct/v1'
  attestation.details.no_network == true

# Types of artifact IDs considered by `lookup attestations for <X>`.
allowed_artifact_id_types = [
  'sha256', 'sha512', 'container_image_digest', 'git_commit',
]
```

## Attestations

Let's take a look at one example team's software supply chain.

![drawing](attestation_supply_chain.png)

*   Top-level code repository is "https://github.com/my-company/my-product".
    *   This defines submodules and the GitHub Actions workflows.
*   Vulnerability scan is provided by an in-house scanner.
*   Docker image is produced by the GitHub Actions "Build" workflow.
    *   In the hypothetical "hermetic" mode, this records all dependent
        submodules and build tools.

This corresponds to the following attestations. Assume each is signed by the
appropriate party; we only show the claim here.

```json
{
  "attestation_type": "https://example.com/CodeReview/v1",
  "subject": { "git_commit": "859b387b985ea0f414e4e8099c9f874acb217b94" },
  "details": {
    "timestamp": "2020-04-12T13:50:00Z",
    "repo_type": "git",
    "repo_url": "https://github.com/my-company/my-product",
    "repo_branch": "master",
    "code_reviewed": true
  }
}
```

```json
{
  "attestation_type": "https://example.com/CodeReview/v1",
  "subject": { "git_commit": "2f02c094e6a9afe8e889c3f1d3cb66b437797af4" },
  "details": {
    "timestamp": "2020-04-12T13:50:00Z",
    "repo_type": "git",
    "repo_url": "https://github.com/my-company/submodule1",
    "repo_branch": "master",
    "code_reviewed": true
  }
}
```

```json
{
  "attestation_type": "https://example.com/CodeReview/v1",
  "subject": { "git_commit": "5215a97a7978d8ee0de859ccac1bbfd2475bfe92" },
  "details": {
    "timestamp": "2020-04-12T13:50:00Z",
    "repo_type": "git",
    "repo_url": "https://github.com/my-company/submodule2",
    "repo_branch": "master",
    "code_reviewed": true
  }
}
```

```json
{
  "attestation_type": "https://example.com/VulnerabilityScan/v1",
  "subject": { "git_commit": "859b387b985ea0f414e4e8099c9f874acb217b94" },
  "details": {
    "timestamp": "2020-04-12T13:55:02Z",
    "vulnerability_counts": {
      "high": 0,
      "medium": 1,
      "low": 17
    }
  }
}
```

```json
{
  "attestation_type": "https://example.com/GitHubActionProduct/v1",
  "subject": { "container_image_digest": "sha256:c201c331d6142766c866..." },
  "relations": {
    "top_level_source": [{
      "artifact": { "git_commit": "859b387b985ea0f414e4e8099c9f874acb217b94" },
      "git_repo": "https://github.com/example/repo"
    }],
    "dependent_sources": [{
      "artifact": { "git_commit": "2f02c094e6a9afe8e889c3f1d3cb66b437797af4" },
      "git_repo": "https://github.com/example/submodule1"
      }, {
      "artifact": { "git_commit": "5215a97a7978d8ee0de859ccac1bbfd2475bfe92" },
      "git_repo": "https://github.com/example/submodule2"
    }],
    "tools": [{
      "artifact": { "sha256": "411c1dfb3c8f3bea29da934d61a884baad341af8..." },
      "name": "clang"
      }, {
      "artifact": { "sha256": "9f5068311eb98e6dd9bb554d4b7b9ee126b13693..." },
      "name": "bazel"
    }]
  },
  "details": {
    "workflow_name": "Build",
    "hermetic": true
  }
}
```

```json
{
  "attestation_type": "https://example.com/ReproducibleBuild/v1",
  "subject": { "sha256": "411c1dfb3c8f3bea29da934d61a884baad341af8..." },
  "details": {
    "name": "clang"
  }
}
```

```json
{
  "attestation_type": "https://example.com/ReproducibleBuild/v1",
  "subject": { "sha256": "9f5068311eb98e6dd9bb554d4b7b9ee126b13693..." },
  "details": {
    "name": "bazel"
  }
}
```

## Policy result attestations

It may not be practical to perform attestation chaining at Kubernetes deployment
time due to latency limitations, since the chain of attestations could be
unbounded in length. To work around this limitation, the full policy evaluation
can happen as a step earlier in the software supply chain. That policy
evaluation returns its own attestation proving that the artifact passed the
policy. Then the Kubernetes policy only requires one such attestation.

```python
kubernetes_policy(artifact):
  lookup attestations for artifact
  allow if any attestation meets passed_policy_evaluation
  deny otherwise

passed_policy_evaluation(attestation):
  attestation is signed by 'BinaryAuthorization'
  attestation.attestation_type == 'https://example.com/BinAuthzDecision/v1'
  attestation.details.decision == 'allow'
  attestation.details.timestamp is within 24 hours of now
  attestation.details.environment matches this Kubernetes environment

allowed_artifact_id_types = ['container_image_digest']
```

```json
{
  "attestation_type": "https://example.com/BinAuthzDecision/v1",
  "subject": { "container_image_digest": "sha256:c201c331d6142766c866..." },
  "details": {
    "timestamp": "2020-04-12T18:04:10Z",
    "decision": "allow",
    "environment": {
      "gcp_project": "example-project",
      "cluster": "us-east1-a.prod-cluster"
    }
  }
}
```

# Motivation

This ITE has two main goals:

1.  Support [use cases](#motivating-use-case) where the existing link schema is
    a poor fit. For example, test steps and vulnerability scans are not about
    "producing" a new artifact so they are awkward to represent in the current
    format.
2.  Support interoperability with
    [Binary Authorization](https://cloud.google.com/binary-authorization), which
    will support the agreed-upon format once finalized. This way we have a
    single ecosystem of software supply chain security.

Functional requirements:

*   Must support user-defined types and schemas, for two reasons:
    *   To allow in-toto users to more naturally express attestations, as
        explained above.
    *   Because Binary Authorization does not want to require its users to use
        the existing in-toto link schema, which is overly specific.
*   Should allow indexing of attestations by artifact ID, without having to
    understand the user-defined schema.
    *   Primary reason: To support generic attestation indexing/storage/fetching
        without requiring user configuration for every type of attestation.
    *   Secondary reason: To simplify the programming model of policies. The
        binding between artifact and attestation can be done in a framework
        without requiring type-dependent configuration.
    *   Implication: the association between attestation and primary artifact ID
        must be standardized.
*   Should allow identification of related artifacts IDs given an attestation,
    without having to understand the user-defined schema.
    *   Reason: To support "inline attestations," where the client fetches and
        sends all required attestations to the server for verification. The
        client does not know the policy ahead of time or understand all
        attestation types.
    *   Example: Given a provenance attestation for a docker image, it should be
        possible to identify all the materials generically.
    *   Implication: the association between attestation and related artifact
        IDs must be standardized.

Nonfunctional requirements:

*   Must support backwards compatible links that can be consumed by existing
    layout files.
*   Must differentiate between different types of related artifacts (only if
    related artifacts are standardized.) Examples: materials vs products,
    sources vs build tools.
    *   Should be type-dependent, rather than mandating "materials" and
        "products."

# Reasoning

## Reason for `subject` and `materials` to be standard Attestation fields

There are two main reasons for standardizing `subject` and `materials` within
Attestation schema.

First, doing so allows policy engines to make decisions without requiring
`attestation_type`-specific logic or configuration. Binary Authorization
policies today are purely about "does an attestation exist that is signed by X
with subject Y", and similarly in-toto layouts are about "does an attestation
exist that is signed by X with materials/products Z?"[1] These relatively simple
policies are quite powerful. With this proposal, such policies become more
expressive without any additional configuration: "does an attestation exist that
is signed by X having type T, with subject Y and/or materials Z?"

Second, it enables lookup of attestations by `subject`, again without
`attestation_type`-specific logic or configuration. Consider the policy
described in the [motivating use case](#motivating-use-case). There, the
instruction is "fetch attestations for artifact X". The lookup could be from a
set of attestations provided by the caller, or it could be from an external
database keyed by subject.[2] Without a standardized `subject` field, this would
be significantly harder.

The alternative is to make `subject` and/or `materials` specific to each
`attestation_type`. Doing so would require users to configure the system for
every possible `attestation_type` they wanted to support, in order to instruct
the system how to find subject and relations. Furthermore, because there would
be no standardization, concepts and models may not necessarily translate between
attestation types. For example, one attestation type might require an "or"
between artifact IDs, while another requires an "and." This difference would add
complexity and confusion.

# Backwards Compatibility

Once the policy engine is updated to support new-style attestations, any
attestation of type "https://in-toto.io/Link/v1" will be supported by existing
layouts.

# Security

# Infrastructure Requirements

# Testing

# Prototype Implementation

# References

# Open Questions

# Footnotes

\[1]: The `expected_command` is only a warning, and `inspections` require
running external commands which is infeasible in many situations.

\[2]: That said, we strongly recommend against keying a database purely by
content hash. The reason is that such databases quickly run into scaling issues,
as explained in
[Building Secure and Reliable Systems](https://static.googleusercontent.com/media/landing.google.com/en//sre/static/pdf/Building_Secure_and_Reliable_Systems.pdf#page=364),
Chapter 14, page 328, "Ensure Unambiguous Provenance." Instead, we recommend
keying primarily by resource name, in addition to content hash.

[ArtifactCollection]: #ArtifactCollection
[Package URL]: https://github.com/package-url/purl-spec/
[RFC 3339]: https://tools.ietf.org/html/rfc3339
[RFC 3986]: https://tools.ietf.org/html/rfc3986
[SPDX Download Location]: https://spdx.github.io/spdx-spec/3-package-information/#37-package-download-location
[Timestamp]: #Timestamp
[TypeURI]: #TypeURI
[in-toto 0.9]: https://github.com/in-toto/docs/blob/v0.9/in-toto-spec.md
[oci_image_id]: https://github.com/opencontainers/image-spec/blob/master/config.md#imageid
