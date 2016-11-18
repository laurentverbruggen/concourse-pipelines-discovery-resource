# Pipelines Discovery Resource

Tracks concourse pipelines in a [git](http://git-scm.com/) repository.

It is based on the [concourse git resource](https://github.com/concourse/git-resource) so most configuration will be identical.

## Installing

Use this resource by adding the following to the `resource_types` section of a pipeline config:

```yaml
---
resource_types:
- name: concourse-pipelines-discovery
  type: docker-image
  source:
    repository: laurentverbruggen/concourse-pipelines-discovery-resource
```

See [concourse docs](http://concourse.ci/configuring-resource-types.html) for more details
on adding `resource_types` to a pipeline config.

## Source Configuration

* `uri`: *Required.* The location of the repository.

* `branch`: *Required.* The branch to track.

* `private_key`: *Optional.* Private key to use when pulling/pushing.
    Example:
    ```
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    ```

* `username`: *Optional.* Username for HTTP(S) auth when pulling/pushing.
  This is needed when only HTTP/HTTPS protocol for git is available (which does not support private key auth)
  and auth is required.

* `password`: *Optional.* Password for HTTP(S) auth when pulling/pushing.

* `paths`: *Optional.* If specified (as a list of glob patterns), only changes
  to the specified files will yield new versions from `check`.

* `ignore_paths`: *Optional.* The inverse of `paths`; changes to the specified
  files are ignored.

* `skip_ssl_verification`: *Optional.* Skips git ssl verification by exporting
  `GIT_SSL_NO_VERIFY=true`.

* `tag_filter`: *Optional*. If specified, the resource will only detect commits
  that have a tag matching the specified expression. Patterns are
  [glob(7)](http://man7.org/linux/man-pages/man7/glob.7.html) compatible (as
  in, bash compatible).

* `git_config`: *Optional*. If specified as (list of pairs `name` and `value`)
  it will configure git global options, setting each name with each value.

  This can be useful to set options like `credential.helper` or similar.

  See the [`git-config(1)` manual page](https://www.kernel.org/pub/software/scm/git/docs/git-config.html)
  for more information and documentation of existing git options.

* `disable_ci_skip`: *Optional* Allows for commits that have been labeled with `[ci skip]` or `[skip ci]`
   previously to be discovered by the resource.

* `commit_verification_keys`: *Optional*. Array of GPG public keys that the
  resource will check against to verify the commit (details below).

* `commit_verification_key_ids`: *Optional*. Array of GPG public key ids that
  the resource will check against to verify the commit (details below). The
  corresponding keys will be fetched from the key server specified in
  `gpg_keyserver`. The ids can be short id, long id or fingerprint.

* `gpg_keyserver`: *Optional*. GPG keyserver to download the public keys from.
  Defaults to `hkp:///keys.gnupg.net/`.

* `config`: *Optional.* (default: concourse.json) The name of the JSON file containing pipeline configuration for this source.
  Configuration can specify an array of pipeline objects, where each objects holds the following fields:

    * `name`: *Required.* Name of the pipeline

    * `config`: *Required.* Relative path (from source config file) to configuration file for pipeline

    * `vars`: *Optional.* Variables that can be passed to pipeline creation, see [fly documentation](https://concourse.ci/fly-set-pipeline.html) for more information.

    * `vars_from`: *Optional.* Variable files that can be passed to pipeline creation, see [fly documentation](https://concourse.ci/fly-set-pipeline.html) for more information.


### Example

Resource configuration for a private repo:

``` yaml
resources:
- name: source-code
  type: concourse-pipelines-discovery
  source:
    uri: git@github.com:laurentverbruggen/concourse-pipelines-discovery-resource.git
    branch: master
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    git_config:
    - name: core.bigFileThreshold
      value: 10m
    disable_ci_skip: true
    config: concourse_pipelines.json
```

Fetching a repo and add credentials as variables file:

``` yaml
- get: source-code
  vars_from:
  - source-code/credentials.yml
```

Pipelines configuration file in repository:

``` json
{
  "pipelines": [
    {
      "name": "pipeline",
      "config": "ci/pipeline.yml",
      "vars_from": [
        "credentials.yml"
      ]
    }
  ]
}
```

## Behavior

### `check`: Check for new commits on pipelines resources

The repository is cloned (or pulled if already present), and any commits
from the given version on are returned, but only if the HEAD contains changes (since the given version)
on pipelines files. If no version is given, the ref for `HEAD` is returned.

Any commits that contain the string `[ci skip]` will be ignored. This
allows you to commit to your repository without triggering a new version.

### `in`: Clone the repository, at the given ref, and get the configuration and files of the pipelines

Clones the repository to the destination, and locks it down to a given ref.
It will return the same given ref as version.

The resource folder will only contain the pipeline resources (config file and all files it references).
You can use those files to generate pipelines in a concourse instance. (see [Concourse Pipelines Sync Resource](https://github.com/laurentverbruggen/concourse-pipelines-sync-resource))

As an additional feature the branch name will be added as variable to the resulting pipelines configuration.
The purpose is to allow repositories to generate different configuration for multiple branches.
This is especially useful with [Concourse Bitbucket Pipelines Discovery Resource](https://github.com/laurentverbruggen/concourse-bitbucket-pipelines-discovery-resource)
in combination with [Concourse Bitbucket Pullrequest Resource](https://github.com/laurentverbruggen/concourse-bitbucket-pullrequest-resource).
When discovering multiple branches you will want to restrict scanning for pull requests to branches for which a pullrequest pipeline was created.

Submodules are initialized and updated recursively.

#### Parameters

* `submodules`: *Optional.* If `none`, submodules will not be
  fetched. If specified as a list of paths, only the given paths will be
  fetched. If not specified, or if `all` is explicitly specified, all
  submodules are fetched.

* `disable_git_lfs`: *Optional.* If `true`, will not fetch Git LFS files.

* `vars`: *Optional.* Adds vars to the resulting concourse configuration file.

* `vars_from`: *Optional.* Adds variable files to the resulting concourse configuration file and
  references them in the resource folder with the same relative path as passed for this config.

Note: no depth parameter is defined (like in git resource) since it is useless here.

#### GPG signature verification

If `commit_verification_keys` or `commit_verification_key_ids` is specified in
the source configuration, it will additionally verify that the resulting commit
has been GPG signed by one of the specified keys. It will error if this is not
the case.

### `out`: No-Op
