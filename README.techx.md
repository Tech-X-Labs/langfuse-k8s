# Tech X Labs fork of Langfuse Helm Charts

## Overview
A set of unfortunate circumstances requires us to fork the Langfuse Helm charts in order to properly support secret management within our charts.

## Issue Details
- Langfuse requires a `NEXTAUTH_SECRET` environment variable which is used for session encryption.
- Langfuse provides a value of `changeme` in their defalut `values.yaml` file.
- In the Langfuse Helm charts, this is accomplished by setting `langfuse.nextauth.secret` in the Helm values file.
  - If set...
    - The Langfuse charts will create a [Kubernetes Secret](/charts/langfuse/templates/nextauth-secret.yaml).
    - The secret value will be populated with the value from `langfuse.nextauth.secret`in the values file.
    - The `NEXTAUTH_SECRET` environment will be populated using a `secretRef` into the secret that the Langfuse charts create.
  - If not set...
    - The Langfuse charts will not create a Kubernetes Secret
    - The Langfuse charts will not provide a `NEXTAUTH_SECRET` environment variable to the Deployment.
- We cannot just provide an a new value for `NEXTAUTH_SECRET` in our values file. This creates two environment variables of the same name which cause Helm issues.
- The ideal solution is for us to specify `langfuse.nextauth.secret = null` in our values file, which _should_ cause the Langfuse secret to not be created. Then we would vault the value for `NEXTAUTH_SECRET` and provide a `secretRef` to it in our values files.
- But, in Helm, there is a bug only when using charts as a dependency where if you specify a value to be null, it does **not** override the value in the parent chart values file.
  
  See bug reports [#12469](https://github.com/helm/helm/issues/12469) and [#12488](https://github.com/helm/helm/issues/12488), tracked by open PR [#12879](https://github.com/helm/helm/pull/12879).
- This means that we are unable to keep the Langfuse charts from creating and referencing their secret which can only take its value from a plaintext value in a values file.

## Solution
There are two solutions. First, clone the Langfuse Helm charts into our repositories and modify them. This is simpler, but keeps us from being able to benefit from upstream changes. The second is to fork the Langfuse Helm charts, set the default `langfuse.nextauth.secret` value to `null`, package, and use as our dependency instead of directly from Langfuse.

## Updating packages and Helm repository index
Distributing a new package happens in two parts: packaging the release and making it available in a Helm repository index.

### Packaging the release
1. Create a branch from a Langfuse tag
1. Make changes as necessary
1. Update [`Chart.yaml`](/Chart.yaml) for the new version number (if needed, ideally should match the corresponding upstream).
1. Create a Github release
1. Create a Helm package using `helm package .`
1. Attach the package tarball to the Github release (e.g. `langfuse-0.x.0.tgz`). This name will correspond to the `<name>-<version>.tgz` from `Chart.yaml`.

### Updating the Helm repository index
When using a Helm dependency, the repository must point to an `index.yaml` file that contains information about the releases and chart packages. That file is served on a Github page within this project located at [`/docs`](/docs).

The [`index.yaml`](/docs/index.yaml) file contains references to _all_ releases, so when you're adding a new release, make sure you're just adding the new information, not replacing previous references.

1. Generate a new index for the branch by running `helm repo index <dir> --url=https://github.com/Tech-X-Labs/langfuse-k8s/releases/download/<release name>`.
2. Verify that you can download the file at `https://github.com/Tech-X-Labs/langfuse-k8s/releases/download/<release name>/<name>-<version>.tgz`
3. Add the new release to [`/index.yaml`](/docs/index.yaml). This will be a new [`- apiVersion: vN`](/docs/index.yaml#L4) section and its child nodes under `entries.langfuse`.
4. Review, commit, and push this index. 
5. Ensure that [Github Pages is configured](https://github.com/Tech-X-Labs/langfuse-k8s/settings/pages) to point to the branch where these changes are made, or main if so configured.

## Reference Chart as Dependency
Reference the forked chart instead of Langfuse directly. For example:
```yaml
apiVersion: v2
name: langfuse
description: My Langfuse deployment
type: application
version: 0.0.1
dependencies:
  - name: langfuse
    version: 0.x.0
    repository: https://langfuse.github.io/langfuse-k8s
```
