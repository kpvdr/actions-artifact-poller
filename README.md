# Wheedle

*wheedle* (v): to influence or entice by soft words or flattery

## Introduction
This project independently polls a pair of GitHub repositories, one for for new commits in order to
trigger a GitHub Action on the second, and the second for new GitHub Actions artifacts.

This application will find use primarily in CI/CD projects where some or all of the package
builds are made on GitHub Actions projects created for the purpose of building and testing
one or more packages. As it is inadvisable to leave credentials for the GitHub build project to
notify a private or secure system, this project can be run from a secure location to poll for
the availability of artifacts on a regular basis, and download them when found.

This project contains two pollers:

#### 1. Commit Poller
This poller will check for commits in any GitHub repository. It is typically used to monitor a
repository that contains the source code for which a second GitHub repository exists to build
packages for the source project and test them. As there may be no direct connection between these
GitHub repositories, it may be difficult to trigger a build action on the build repository from
commits made on the source repository.

The commit poller will check for new commits since the last successful build was made by
comparing the commit hashes in the Git commit log. If a new commit is detected since the last
successful build, a build action is triggered on the build repository's actions.

#### 2. Artifact Poller
This poller will check for new GitHub Actions artifacts which may become available. The poller is
run at a regular interval, and keeps track of artifacts it has already seen so as to avoid
downloading duplicate artifacts.

If a new artifact is found, it is downloaded into a temporary location, and then pushed into the
Bodega artifact server. Tagged metadata is then sent to the Stagger artifact tagging and
notification service.

## Requirements
- A **GitHub Personal Token** which will be used for GitHub API calls to the repositories. See
[Creating a personal access token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token)
for instructions on how to do this. Copy the token into a file named `token` which will be located
in the `DATA_DIR` directory. You can rename this, but make sure to change the configuration
(see [Configuration](#configuration) below) to reflect the new name.

## Dependencies
- **Requests** (https://requests.readthedocs.io/en/master/) - This is packaged on some distros (such as
  Fedora) but must be installed using `pip install --user requests` on those where this is not the
  case.
- **Podman** (https://podman.io/) - Packaged on most distros. This is needed if building or using
  containers
- **Python 3**

## Building and installing
```
git clone https://github.com/kpvdr/wheedle.git
```
No build or install (yet). The application runs using make directly from the repository.

## Configuration
In the following tables, there are references to two GitHub repositories:
1. **Source repository:** The repository containing the source code, and which the *commit poller*
   polls for new commits;
2. **Build repository:** The repository which is triggered by the *commit poller*, and which builds
   and tests packages from the source code in the Source Repository. The *artifact poller* will
   check this repository's actions for new artifacts and process them in Bodega and Stagger.

Currently configuration is performed by modifying app.py, where the following constants are
defined:

Constant | Description
---------|------------
`GITHUB_SERVICE_URL` |  GitHub API service URL for all GH API requests.
`BUILD_REPO_OWNER` | GitHub build repository owner.
`BUILD_REPO_NAME` | GitHub build repository name.
`SOURCE_REPO_OWNER` | GitHub source repository owner.
`SOURCE_REPO_NAME` | GitHub source repository name.
`SOURCE_REPO_BRANCH` | Branch for which commits will be polled and from which build artifacts will be built.
`GH_API_AUTH_UID` | Name of token owner which will be used for GitHub API calls.
`GH_API_TOKEN_FILE_NAME` | Text file containing a GitHub personal access token.
`DATA_DIR` | Location of locally saved data and the GitHub token file.
`COMMIT_POLLER_START_DELAY` | Delay in seconds for starting the *commit poller* after the *artifact poller* starts. Allows the *artifact poller* to download the last commit hash.
`ARTIFACT_POLLING_INTERVAL_SECS` | *Artifact poller* interval in seconds.
`ERROR_POLLING_INTERVAL_SECS` | *Artifact poller* interval in seconds when the Bodega and Stagger services are not running.
`LAST_BUILD_CID_ARTIFACT_NAME` | Name of file located in the `DATA_DIR` which contains the last successful build commit hash.
`BUILD_ARTIFACT_NAME_LIST` | Python list of strings containing names of artifacts to be downloaded and processed if found. Wildcards are allowed.
`ARTIFACT_POLLER_DATA_FILE_NAME` | Name of file in the `DATA_DIR` which contains the artifact ids of previously downloaded artifacts.
`TAG` | Tag to be used with Stagger when pushing artifact metadata.
`BODEGA_URL` | URL for the Bodega artifact service.
`STAGGER_URL` | URL for the Stagger artifact tagging service.
`COMMIT_POLLING_INTERVAL_SECS` | *Commit poller* interval in seconds.
`COMMIT_DATA_FILE_NAME` | Name of file in the `DATA_DIR` which contains *commit poller* persistent data.
`DEFAULT_LOG_LEVEL` | Log level (one of `DEBUG`, `INFO`, `WARNING`. `ERROR`, `CRITICAL`).

**TODO:** A file-based configuration will be added soon which will allow multiple instances of the
artifact and commit pollers to run against different repositories.

## Installing, Running and Stopping
#### Running in local environment
Install is performed when first running, and is located at `${HOME}/.local/opt/wheedle`.
```
make run
```
However, install can be performed separately by running `make install` first. An alternative install
location may be specified by adding `INSTALL_DIR=/another/path` after each make statement, ie:
```
make run INSTALL_DIR=/my/new/path
```
To stop, use `ctrl+C` or send a `TERM` signal.

#### Running in a Docker container
The container uses the latest version of Fedora.

1. First build a container image with `make build-image`. This may take a minute or so to complete.
1. Once built, the container can be run and stopped as often as needed with `make run-image` and
   `make stop-image` respectively.
1. An image can be deleted with `make delete-image`. This must be done before a new image can be
   built.

#### Personal Access Token
**NOTE:** A Personal Access Token file should exist named `token` and which should be pointed to in
an environment variable `${TOKEN_FILE}` prior to running `make install` or `make run` (see
[Requirements](#requirements) above). If this variable does not exist, or the token file is not
present, then it will *NOT* be copied to the installation location (there will be a warning), and an
attempt to run the application will produce a `TokenNotFoundError`. The token will need to be
copied manually to `${HOME}/.local/opt/wheedle/data` (or `${INSTALL_DIR}/data` if you specified a
different install directory) before the application can run.

## Persistent Data
Persistent data which is re-loaded each time the application is started, is saved in `DATA_DIR`
(`${HOME}/.local/opt/wheedle` by default).

Persistent data may be cleared by deleting the JSON files (`*.json`) in `DATA_DIR`, or by running
`make clean`. **WARNING:** Do not delete the Personal Access Token file `token` which is also
located in this directory. The application will not run without this file.

## Persistence Files
Name | Type | Description
-----|------|------------
artifact_id.json | JSON | A list of artifact ids previously seen indexed by run number.
commit_hash.json | JSON | The commit hash of the last build. This is uploaded as an artifact.

## Troubleshooting
Error | Possible cause
------|---------------
`ServiceConnectionError` | Either or both the Stagger and Bodega services could not be reached from the pollers. Check the `BODEGA_URL` and `STAGGER_URL` settings, and make sure that these are running and are accessible on the network from the poller machine.
`TokenNotFoundError` | The GitHub token file was not found. See [Requirements](#requirements) above.
