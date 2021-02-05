# Provenance

## Schema

See [provenance_curl_maketgz.yaml](examples/provenance_curl_maketgz.yaml).
(TODO: Move docs here)

## Appendix: Review of CI/CD systems

The intention is to do a comprehensive review of all widely used CI/CD systems
and see if they cleanly fit into the schema.

*   [Github Actions](https://docs.github.com/en/actions)

    *   source: git repo on Github
    *   builder:
        *   orchestrator: hosted
        *   worker: hosted or custom (called a "runner")
    *   build steps:
        *   configuration: source
        *   id: $WORKFLOW:$JOB (`.github/workflows/$WORKFLOW`)
        *   parameters:
            *   `inputs` for
                [workflow_dispatch](https://docs.github.com/en/actions/managing-workflow-runs/manually-running-a-workflow)
    *   isolation: runs in a VM

*   [Google Cloud Build](https://cloud.google.com/cloud-build/docs) - Triggers

    *   source: git repo on Github or Google Source Repositories
    *   [source](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/projects.builds#source):
        *   tarball on Google Cloud Storage
        *   git repo on Google Source Repository or GitHub
            ([RepoSource](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/RepoSource))
            *   NOTE: includes build directory and substitutions!
        *   NOTE: It's not at all clear what happens when you use the `gcloud`
            command. I'm guessing it uploads a tarball to GCS?
    *   builder: single orchestrator, hosted or custom worker
    *   entry point: Dockerfile or cloudbuild.yaml or cloudbuild.json in any
        directory (not sure of precidence). Two types of builds:
        *   Steps are specified in the API. This doesn't really fit with out
            Provenance API and should be relegated to another type of
            attestation, if at all.
            *   [manual build](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/projects.builds/create)
                requires `steps` to be listed
            *   [triggered](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/projects.triggers/create)
                if `steps` is used
        *   Steps are specified in a source file
            *   [triggered](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/projects.triggers/create)
                if `filename` is used, which specifies the path to the config
    *   runs in a Docker container

### Not fully automated

*   [Google Cloud Build](https://cloud.google.com/cloud-build/docs) - Manual
    Builds

    *   fully automated: NO (build steps defined in request)
    *   out of scope
    *   [source](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/projects.builds#source):
        *   tarball on Google Cloud Storage
        *   git repo on Google Source Repository or GitHub
            ([RepoSource](https://cloud.google.com/cloud-build/docs/api/reference/rest/v1/RepoSource))
            *   NOTE: includes build directory and substitutions!
        *   NOTE: It's not at all clear what happens when you use the `gcloud`
            command. I'm guessing it uploads a tarball to GCS?
