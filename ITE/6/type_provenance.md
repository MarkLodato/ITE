# Attestation type: Provenance

## Purpose

Describe how an artifact or set of artifacts was produced.

Focus: Fully automated builds with config-as-code and no other parameters.
(We may a more specific name or subtype, since we will need a different
attestation type for builds that don't meet this definition.)

## Schema

See [provenance_curl_maketgz.yaml](examples/provenance_curl_maketgz.yaml).

TODO: Move docs here

## Appendix: Review of CI/CD systems

See [ci_survey.md](ci_survey.md) for a list of well-known CI/CD systems, to make
sure they all map cleanly into this schema.
