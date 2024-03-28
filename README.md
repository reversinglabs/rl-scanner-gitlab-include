# ReversingLabs GitLab CI configuration: rl-scanner-gitlab-include

ReversingLabs provides the officially supported GitLab CI configuration for faster and easier deployment of the `rl-secure` solution in CI/CD workflows.

The `rl-scanner-gitlab-include` repository provides the remote include configuration file called `gl-include-remote-reversinglabs-rl-scanner.yml`.

To add the configuration to existing pipelines, use the following syntax:


        include:
          - remote: <raw url of the yml file>


The configuration uses the official [reversinglabs/rl-scanner](https://hub.docker.com/r/reversinglabs/rl-scanner) Docker image to scan a single build artifact with `rl-secure`, generate the analysis report, and display the analysis status as one of the checks in the GitLab interface.

This configuration is most suitable for experienced users who want to include it into more complex workflows.

By default, the configuration is set up to work with GitLab-hosted runners. It is also possible to use it with locally installed runners to accommodate the [package store use-case](#local-gitlab-runner).


## What is rl-secure?

`rl-secure` is a CLI tool that's part of the [Spectra Assure platform](https://www.reversinglabs.com/products/software-supply-chain-security) - a new ReversingLabs solution for software supply chain protection.

With `rl-secure`, you can:

- Scan your software release packages on-premises and in your CI/CD pipelines to prevent threats from reaching production.
- Compare package versions to ensure no vulnerabilities are introduced in the open source libraries and third-party components you use.
- Prevent private keys, tokens, credentials, and other sensitive information from leaking into production.
- Improve developer experience and ensure compliance with security best practices.
- Generate actionable analysis reports to help you prioritize and remediate issues in collaboration with your DevOps and security teams.


## How the include configuration works

The include configuration is a remote file that needs to be included into the main GitLab pipeline and that acts as its test stage.

This configuration expects that the build artifact is produced before the test stage and uploaded to the job for it to be shared between jobs.
It requires specifying the path of the artifact as the input to the pipeline.
The path must be relative to the root of the GitLab repository.

Users also need to specify the report directory name, as the pipeline saves all supported analysis report formats for the artifact into that directory.
The directory path must be relative to the root of the GitLab repository.
The CycloneDX report is uploaded separately to the same user-provided report directory and can be used for [compliance purposes](https://docs.gitlab.com/ee/user/compliance/license_scanning_of_cyclonedx_files/).

This pipeline has specific checks for [environment variables](#environment-variables) and [input parameters](#inputs) that throw an error if any values are missing.

The scan job runs a set of commands that pull the latest version of the `reversinglabs/rl-scanner` Docker image and scan the provided artifact inside a container.
A specific entrypoint has to be used in order to make the image compatible with GitLab runner.
When the security scan is done, the container automatically shuts down, and the action outputs the analysis result as a status message (PASS, FAIL, ERROR).


## Requirements

1. **A valid rl-secure license with site key**. If you don't already have a site-wide deployment license, follow the instructions in [the official rl-secure documentation](https://docs.secure.software/cli/deployment/rl-deploy-quick-start#prepare-the-license-and-site-key) to get it from ReversingLabs. You don't need to activate the license - just save the license file and the site key for later use. You must convert your license file into a Base64-encoded string to use it with the Docker image.

2. **Your rl-secure license information added as secrets** to your GitLab organization or repository. Use predefined [environment variables](#environment-variables) to provide the secrets to the pipeline.

**Note for GitLab Enterprise users:** The GitLab CI/CD must be appropriately configured for the repository where you want to use the pipeline. If you don't have the [appropriate role](https://docs.gitlab.com/ee/ci/quick_start/index.html), contact your GitLab organization administrators for help.


## Environment variables

This configuration requires the `rl-secure` license data to be passed using the following environment variables.

| Environment variable | Required | Description |
| :--------- | :------ | :------- |
| `RLSECURE_ENCODED_LICENSE` | **Yes** | The `rl-secure` license file as a Base64-encoded string. Users must encode the contents of your license file, and provide the resulting string with this variable. |
| `RLSECURE_SITE_KEY` | **Yes** | The `rl-secure` license site key. The site key is a string generated by ReversingLabs and sent to users with the license file. |

These secrets have to be defined via your GitLab repository `Settings → CI/CD → Variables`.

ReversingLabs strongly recommends following best security practices and [defining these secrets](https://docs.gitlab.com/ee/ci/variables/) on the level of your GitLab organization or repository.


## How to use this include configuration

The most common use-case for this configuration is to run it as the "test" stage of the pipeline, after the build artifact has been created.

To use the `rl-secure` security scanning functionality, a valid site-wide deployment license is required.
This type of license has two parts: the site key and the license file.
ReversingLabs sends both parts of the license to users on request.
Users must then encode the license file with the Base64 algorithm.
The Base64-encoded license string and the site key must be provided to the pipeline using [environment variables](#environment-variables).


### Inputs

| Input parameter         | Required | Description |
| :---------              | :-----   | :------ |
| `MY_ARTIFACT_TO_SCAN`   | **Yes**  | The file name of the artifact to be scanned. The artifact must exist before the test stage is run. |
| `PACKAGE_PATH`          | **Yes**  | The location relative to the checkout of the artifact to be scanned. This artifact is expected to be found at: `${PACKAGE_PATH}/${MY_ARTIFACT_TO_SCAN}`. |
| `REPORT_PATH`           | **Yes**  | The location relative to the checkout where analysis reports will be stored after the scan is finished. The directory specified as `REPORT_PATH` must be empty. It is deleted and re-created before the scan starts.  |
| `RLSECURE_PROXY_SERVER`   | No | Server name for proxy configuration (IP address or DNS name). |
| `RLSECURE_PROXY_PORT`     | No | Network port on the proxy server for proxy configuration. Required if `RLSECURE_PROXY_SERVER` is used. |
| `RLSECURE_PROXY_USER`     | No | User name for proxy authentication. |
| `RLSECURE_PROXY_PASSWORD` | No | Password for proxy authentication. Required if `RLSECURE_PROXY_USER` is used. |
| `RL_STORE`                | No | If using a package store, use this parameter to provide the path to a directory where the self-hosted package store has been initialized. |
| `RL_PACKAGE_URL`          | No | If using a package store, use this parameter to specify the package URL (PURL) for the scanned artifact. The package URL should be in the format `project/package@version`; for example `testing/demo-rl-scanner@v1.0.3`. |
| `RL_DIFF_WITH`            | No | If using a package store, use this parameter to specify a previously scanned package version to compare (diff) against. |
| `RL_VERBOSE`              | No | Set to anything but '' to provide more feedback in the output while running the scan. Disabled by default. |


### Outputs

| Output parameter | Description |
| :--------- | :------ |
| `DESCRIPTION` | The result of the action - a string terminating in FAIL or PASS. |
| `STATUS` | The single-word status representing the result of the action. It can be any of the following: success, failure, error. <br>**Success** indicates that the resulting string contains PASS. <br> **Failure** indicates the resulting string contains FAIL. <br>**Error** indicates that something went wrong during the scan and the action was not able to retrieve the resulting string. |


### Artifacts

The job creates the reports in the directory: `$REPORT_PATH`.
The reports are always created even if the scan job fails.

Users can control the `REPORT_PATH` as an input parameter.
The specified directory must be empty.


## Examples

The following example is a basic pipeline that includes the remote configuration as the test stage.

The workflow scans a build artifact with `rl-secure`, saves the report, and displays the analysis results.
The remote configuration can be included anywhere in the configuration file, but it will run only after the build stage.

```
# REQUIREMENTS:
#   RLSECURE_SITE_KEY: must be declared as a global variable type 'variable'
#   RLSECURE_ENCODED_LICENSE: must be declared as a global variable type 'variable'

variables:
  MY_ARTIFACT_TO_SCAN: bash
  PACKAGE_PATH: packages
  REPORT_PATH: report
 
include:
  - remote: 'https://raw.githubusercontent.com/reversinglabs/rl-scanner-gitlab-include/main/rl-scanner-gitlab-include.yml'
 
job-build:
  stage: build
 
  image:
    # Any image you require
    name: ubuntu:latest
 
  artifacts:
    name: "build_artifact"
    paths:
      - $PACKAGE_PATH/*
 
  script:
    - |
      export HOME=$( pwd ); echo $HOME
 	  # Prepare to build an artifact as the build output
      mkdir $PACKAGE_PATH       
      cp /bin/${MY_ARTIFACT_TO_SCAN} $PACKAGE_PATH/
 
job-deploy:
  stage: deploy
 
  image:
    # Any image you require
    name: ubuntu:latest
 
  script:
    - |
      # Here we have access to all artifacts from the previous jobs
      # $PACKAGE_PATH and $REPORT_PATH should be visible
      export HOME=$( pwd ); echo $HOME
      ls -la .
      ls -la $PACKAGE_PATH
      ls -la $REPORT_PATH
 
      echo "This job deploys the artifact from the $CI_COMMIT_BRANCH branch."
```


### Local GitLab runner

The `rl-scanner-gitlab-include` configuration is based on the way the GitLab-hosted runners work. 
That means a Docker-based [executor](https://docs.gitlab.com/runner/executors/) is expected.

To use a self-hosted [package store](https://docs.secure.software/concepts/basic-concepts#package-store), you will have to install your own runner(s) locally. 
Using a package store opens up the possibility of advanced use cases for the scan process. 
Specifically, you can compare two package versions to generate a diff report.

If you want to use one central package store, you will have to set that up using NFS or CIFS. 
The Docker runners can only use external volumes that are configured in the [config.toml](https://docs.gitlab.com/runner/commands/index.html#configuration-file) file for your `executor`.

The following example `config.toml` file shows how a package store (`rl-store`) is shared between all Docker instances.
In this example, we're using an NFS-based shared package store that can be used by multiple `executors`.

**Example config.toml**

```
    concurrent = 1
    check_interval = 0
    shutdown_timeout = 0

    [session_server]
      session_timeout = 1800

    [[runners]]
      name = "gitlab-runner-***"
      url = "https://gitlab.com"
      id = <some id>
      token = "**********************"
      token_obtained_at = <some date>
      token_expires_at = <some other date>
      executor = "docker"
      [runners.cache]
        MaxUploadedArchiveSize = 0
      [runners.docker]
        tls_verify = false
        image = "<your image configured>"
        privileged = false
        disable_entrypoint_overwrite = false
        oom_kill_disable = false
        disable_cache = false
        volumes = [
          "/cache",
          "/mount/nfs/rl-store:/rl-store:rw",
        ]
        shm_size = 0
```


The following example is a pipeline for a local GitLab runner that includes the `rl-scanner-gitlab-include` configuration.

In this example, we're using a proxy configuration and a package store. 

The package store location is specified with the `RL_STORE` variable. Every scanned artifact is added to the store as a package version assigned to a project and package, and all those elements together make up the package URL (project/package@version). 
The analysis results for every scanned artifact are preserved in the package store. 
When scanning a new package version, you can compare it against previously scanned package versions in the same project with the `RL_DIFF_WITH` variable.


**Example pipeline with proxy configuration and package store**

```
        # REQUIREMENTS:
        #   RLSECURE_SITE_KEY: must be declared as global variables type 'variable'
        #   RLSECURE_ENCODED_LICENSE: must be declared as global variables type 'variable'

        variables:
          MY_ARTIFACT_TO_SCAN: bash
          PACKAGE_PATH: packages
          REPORT_PATH: RlReport
          RLSECURE_PROXY_SERVER: a-proxy.in-a.domain
          RLSECURE_PROXY_PORT: 3128
          RLSECURE_PROXY_USER: proxy_user
          RLSECURE_PROXY_PASSWORD: proxy_password
          RL_STORE: /rl-store
          RL_PACKAGE_URL: testing/demo-rl-scanner@v1.0.3
          # RL_DIFF_WITH: v1.0.1
          RL_VERBOSE: 1

        include:
          - remote: 'https://raw.githubusercontent.com/reversinglabs/rl-scanner-gitlab-include/main/rl-scanner-gitlab-include.yml'

        job-build:
          stage: build

          image:
            name: fedora:latest # or any other image you require

          artifacts:
            name: "build_artifact"
            paths:
              - $PACKAGE_PATH/*

          script:
            - |
              export HOME=$( pwd ); echo $HOME
              mkdir $PACKAGE_PATH # prepare to build an artifact as the build output
              cp /bin/${MY_ARTIFACT_TO_SCAN} $PACKAGE_PATH/

        job-deploy:
          stage: deploy

          image:
            name: ubuntu:latest # or any other image you require

          script:
            - |
              # Here we have access to all artifacts
              # of the previous jobs
              # so $PACKAGE_PATH and $REPORT_PATH should be visible
              export HOME=$( pwd ); echo $HOME
              ls -la .
              ls -la $PACKAGE_PATH
              ls -la $REPORT_PATH
              echo "This job deploys something from the $CI_COMMIT_BRANCH branch."
```


## Useful resources

- The official `reversinglabs/rl-scanner` Docker image [on Docker Hub](https://hub.docker.com/r/reversinglabs/rl-scanner)
- [Supported file formats](https://docs.secure.software/concepts/filetypes) and [language coverage](https://docs.secure.software/concepts/language-coverage) for `rl-secure`
- Introduction to [secure software release processes](https://www.reversinglabs.com/solutions/secure-software-release-processes) with ReversingLabs
- [Software supply chain security for application security teams](https://3375217.fs1.hubspotusercontent-na1.net/hubfs/3375217/Documents/Business-Brief-Software-Supply-Chain-Security-for-Application-Security-Teams.pdf) (link to a PDF document)
- [Use CI/CD configuration from other files](https://docs.gitlab.com/ee/ci/yaml/includes.html)
- [GitLab runner](https://docs.gitlab.com/runner/)
- [GitLab executor](https://docs.gitlab.com/runner/executors/)
- [GitLab Docker executor 'config.toml'](https://docs.gitlab.com/runner/commands/index.html#configuration-file)
