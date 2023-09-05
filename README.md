# <a href="https://github.com/jupyterhub/repo2docker"><img src="https://raw.githubusercontent.com/jupyterhub/repo2docker/8731ecf0967cc5fde028c456f2b92be651ebbc18/docs/source/_static/images/repo2docker.png" height="48px" /> repo2docker</a>

[![Build Status](https://github.com/jupyterhub/repo2docker/workflows/Test/badge.svg)](https://github.com/jupyterhub/repo2docker/actions)
[![Documentation Status](https://readthedocs.org/projects/repo2docker/badge/?version=latest)](http://repo2docker.readthedocs.io/en/latest/?badge=latest)
[![Contribute](https://img.shields.io/badge/I_want_to_contribute!-grey?logo=jupyter)](https://repo2docker.readthedocs.io/en/latest/contributing/contributing.html)
[![Docker Repository on Quay](https://img.shields.io/badge/quay.io-container-green "Docker Repository on Quay")](https://quay.io/repository/jupyterhub/repo2docker?tab=tags)

`repo2docker` fetches a git repository and builds a container image based on
the configuration files found in the repository.

See the [repo2docker documentation](http://repo2docker.readthedocs.io)
for more information on using repo2docker.

For support questions please search or post to https://discourse.jupyter.org/c/binder.

See the [contributing guide](CONTRIBUTING.md) for information on contributing to
repo2docker.

---

Please note that this repository is participating in a study into sustainability
of open source projects. Data will be gathered about this repository for
approximately the next 12 months, starting from 2021-06-11.

Data collected will include number of contributors, number of PRs, time taken to
close/merge these PRs, and issues closed.

For more information, please visit
[our informational page](https://sustainable-open-science-and-software.github.io/) or download our [participant information sheet](https://sustainable-open-science-and-software.github.io/assets/PIS_sustainable_software.pdf).

---

## Using the "enterprise" additions

The intention of this [repo2docker](http://repo2docker.readthedocs.io) fork is to remain fully backward-compatible.  The following
CLI arguments have been _added_ to enable using the tool in an enterprise context.

### Authenticated `pip`

Your choice of `pip` "index URL" can be supplied by specifying `--pip-index-url <INDEX_URL>`.  This will be injected into the ephemeral `Dockerfile` interally created by repo2docker as a build argument and exposed to `pip` via the (build-time) environment.  You can see the effect of this by specifying `--pip-index-url` and `--no-build` together.

If your `pip` repository requires authentication, the following additional arguments can also be supplied:

   - `--pip-auth <AUTH_TYPE>`<br>
     Valid authentication types are `basic` (HTTP basic auth), `azure-sp-key` and `azure-sp-certificate` (for Azure Service Principals).
   - `--pip-identity <IDENTITY>`<br>
     The "identity" for authenticating against the `pip` repository.  For HTTP basic auth, this is the username.  For Azure Service Principals, specify the Tenant and App Registration's Client ID in the form `<TENANT_ID>/<CLIENT_ID>` (e.g.: `aaaa-bbbb-cccc-dddd-eeee/tttt-uuuu-vvvv-wwww-xxxx`).
   - `--pip-secret <SECRET>`<br>
     The "secret" for authenticating against the `pip` repository.  For HTTP basic auth, this is the password.  For Azure Service Principals, this is the "secret" (password) for the associated App Registration, or a PEM formatted certificate + private key.

## Combining the "enterprise" additions with BinderHub

If you're using BinderHub and wish to use the aforementioned "enterprise" additions, [KubeMod](https://github.com/kubemod/kubemod) can fulfil your requirements.

### Authenticating against git repositories

In an enterprise context, it's likely your notebooks reside in git repositories mandating authentication.  Assuming you've deployed BinderHub atop `microk8s`, you can use the following instructions to have KubeMod inject the required SSH configuration into repo2docker immediate before build-time.

You will need to use a repo2docker container image with the SSH client installed (the official image presently does not).  These instructions assume you have built the Dockerfile in this repository and and tagged it `repo2docker-enterprise`.

These should be adaptable for other flavours of Kubernetes.

```
# On the microk8s host:
sudo mkdir -p /srv/hostdata/pvc-ssh-config
cd /srv/hostdata/pvc-ssh-config/
# Replace bitbucket.com with your git provider
ssh-keyscan -t rsa bitbucket.com | sudo tee -a known_hosts
sudo ssh-keygen  # don't supply a password
```

The SSH configuration can be made available to pods running on Kubernetes as a persistent volume.  The following Kubernetes manifest does this:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-ssh-config
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 100Mi
  storageClassName: local-storage-dir
  volumeMode: Filesystem
  local:
    fsType: ""
    path: /srv/hostdata/pvc-ssh-config
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-ssh-config
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  volumeName: pvc-ssh-config
```

BinderHub itself also needs to have this SSH configuration injected into it, and needs to be instructed to use the `repo2docker-enterprise` container image for builds.  This can be done via the following BinderHub Helm chart values:

```
config:
  BinderHub:
    build_image: repo2docker-enterprise:latest
  KubernetesBuildExecutor:
    build_image: repo2docker-enterprise:latest

extraVolumes:
  - name: pvc-ssh-config
    persistentVolumeClaim:
      claimName: pvc-ssh-config
extraVolumeMounts:
  - name: pvc-ssh-config
    mountPath: /root/.ssh
```

Finally for repo2docker itself, we use KubeMod to apply a real-time "JIT" patch as required:

```
apiVersion: api.kubemod.io/v1beta1
kind: ModRule
metadata:
  name: repo2docker-enterprise-patch
spec:
  type: Patch

  match:
    - select: '$.kind'
      matchValue: 'Pod'

    - select: '$.metadata.labels.component'
      matchValue: 'binderhub-build'

  patch:
    - op: add
      path: /spec/volumes/-1
      value: |-
        name: pvc-ssh-config
        persistentVolumeClaim:
          claimName: pvc-ssh-config

    # if you are using a private repository for hosting repo2docker-enterprise ...
    - op: add
      path: /spec/imagePullSecrets/-1
      value: |-
        name: image-pull-secret

    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/volumeMounts/-1
      value: |-
        mountPath: /root/.ssh
        name: pvc-ssh-config
        readOnly: true
```

This KubeMod patch identifies the repo2docker pods when they are started by BinderHub and adds the SSH configuration volume.

### Authenticating against `pip` repositories

Follow the instructions above for adding authenticated git support, skipping the SSH volumes etc if they're not needed.  In the KubeMod manifest, append the following additional directives which will append `--pip-auth ...` CLI arguments to repo2docker:

```
    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "--pip-auth"
    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "azure-sp-key"

    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "--pip-index-url"
    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "https://pkgs.dev.azure.com/<ORGANISATION>/<PROJECT_GUID>/_packaging/<REPOSITORY>/pypi/simple/"

    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "--pip-identity"
    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "<TENANT_ID>/<CLIENT_ID>"

    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "--pip-secret"
    - op: add
      select: '$.spec.containers[? @.name == "builder" ]'
      path: /spec/containers/#0/args/-2
      value: "<APP_REG_SECRET>"
```

Configure the argument values as appropriate.

## Using repo2docker

### Prerequisites

1. Docker to build & run the repositories. The [community edition](https://store.docker.com/search?type=edition&offering=community)
   is recommended.
2. Python 3.6+.

Supported on Linux and macOS. [See documentation note about Windows support.](http://repo2docker.readthedocs.io/en/latest/install.html#note-about-windows-support)

### Installation

This a quick guide to installing `repo2docker`, see our documentation for [a full guide](https://repo2docker.readthedocs.io/en/latest/install.html).

To install from PyPI:

```bash
pip install jupyter-repo2docker
```

To install from source:

```bash
git clone https://github.com/jupyterhub/repo2docker.git
cd repo2docker
pip install -e .
```

### Usage

The core feature of repo2docker is to fetch a git repository (from GitHub or locally),
build a container image based on the specifications found in the repository &
optionally launch the container that you can use to explore the repository.

**Note that Docker needs to be running on your machine for this to work.**

Example:

```bash
jupyter-repo2docker https://github.com/norvig/pytudes
```

After building (it might take a while!), it should output in your terminal
something like:

```bash
    Copy/paste this URL into your browser when you connect for the first time,
    to login with a token:
        http://0.0.0.0:36511/?token=f94f8fabb92e22f5bfab116c382b4707fc2cade56ad1ace0
```

If you copy paste that URL into your browser you will see a Jupyter Notebook
with the contents of the repository you had just built!

For more information on how to use `repo2docker`, see the
[usage guide](http://repo2docker.readthedocs.io/en/latest/usage.html).

## Repository specifications

Repo2Docker looks for configuration files in the source repository to
determine how the Docker image should be built. For a list of the configuration
files that `repo2docker` can use, see the
[complete list of configuration files](https://repo2docker.readthedocs.io/en/latest/config_files.html).

The philosophy of repo2docker is inspired by
[Heroku Build Packs](https://devcenter.heroku.com/articles/buildpacks).

## Docker Image

Repo2Docker can be run inside a Docker container if access to the Docker Daemon is provided, for example see [BinderHub](https://github.com/jupyterhub/binderhub). Docker images are [published to quay.io](https://quay.io/repository/jupyterhub/repo2docker?tab=tags). The old [Docker Hub image](https://hub.docker.com/r/jupyter/repo2docker) is no longer supported.
