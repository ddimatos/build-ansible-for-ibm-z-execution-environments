# Create Custom Ansible Execution Environments using Ansible Builder for IBM Z

This document will detail how you can use `ansible-builder` to configure and build custom Ansible for IBM Z execution environments to operate as  Ansible control node, providing operations a versioned, portable and validated control node that can be taken from test to production.

There are many uses for containerized execution environments, they can be used with:
- Ansible Automation Platform
- Ansible Navigator
- AWX
- Localized development
- CI/CD Pipelines

The ability to customize execution environments provides enterprise operations a consistent and proven solution that has been approved for use with production, used in the development phase, possibly designed with approved and validated packages delivering consistent results.


# Setting up the local environment to create the execution environment.

I am using my local system to build the execution environments, I do this in a Python virtual environment to separate any package creep into the host.

## Evaluate Python

The requirements for [ansible-builder](https://ansible.readthedocs.io/projects/builder/en/stable/installation/#requirements) version 3.x is currently are python 3.9 or later.

On your system, check the level of python with:

``` sh
python --version
```

If you don't have python installed or are running an older version, you can install a later version with `brew`.

``` sh
brew install python@3.13
```

If you need to later revert to older version of python you can unlink the recently installed version and link the preferred version.
``` sh
brew unlink python@3.13
brew link python@3.9
```

## Evaluate your containerization manager
To build execution environments, you need either Podman or Docker. You can check if you have one installed with on of the following commands:

``` sh
podman info
docker info
```

I am using Podman Desktop and obtained it through the [download](https://podman-desktop.io/).

## Create Python virtual environment (venv):

Create the Python virtual environment to use to build the execution environment. This will ensure that packages are not installed on your Mac and instead into the venv which can easily be updated or removed later.

``` sh
python3 -m venv ansible-z-builder-venv
```

Now there is a folder named `ansible-z-builder-venv` for which you can now activate the Python virtual environment with command:

``` sh
source ansible-z-builder-venv/bin/activate
```

## Install ansible-builder into the Python Virtual Environment

Install `ansible-builder` into the Python virtual environment so that we can build an execution environment with command:


``` sh
pip3 install ansible-builder
```

You can deactivate the Python virtual environment, you don't need it active any longer.

``` sh
deactivate
```

# Execution Environments

At this point you have a virtual environment and a container manager like Podman that `ansible-builder` depends on to build a containerized solution. Some files are needed to instruct `ansible-builder` which container to use, which packages to install and some other necessary details.

It is beyond the scope of this tutorial to explain a container managers details but some high level points will aid in following along.

Both Podman and Docker use files , `Containerfile` or `Dockerfile` that contain the instructions to create a container image.

A container image is a replica of a container only it is not active and can be used to create additional copies of the container.

While a container is a collection of code and packages bundled such that it ca be run on a variety of platforms.


# Building an Execution Environment

Ansible for IBM Z has several collections, for the step by step portion of this guide, we will use the Ansible Galaxy collection `ibm.ibm_zos_core` then we will repeat the same process focusing on `ibm.ibm_zhmc` that requires a number of third party packages.


## Ansible Execution Environment Files

You will need to several files that `ansible-builder` will use to build the Ansible execution environment image.

* `bindep.txt` - file that specifies system-level dependencies needed for an Ansible Execution Environment (EE)
* `requirements.txt` - The Python dependencies required for the collections identified in `requirements.yaml` and any others needed for proper execution a playbook.
* `requirements.yml` = The Ansible Collections to be installed into the execution environment needed for proper execution of a playbook.
* `execution-environment.yml` - defines the operations that `ansible-builder` will perform when creating the execution environment including the base image, dependencies, build steps, files and configurations.


# Building an Execution Environment for Collection `ibm_zos_core`

In this next section we will set up all the directories and files necessary to build an execution environment for the collection.

## Create a directory with command:

`mkdir ibm-zos-core-ee`

## Create files with commands:

`touch ibm-zos-core-ee/execution-environment.yml`
`touch requirements.yml`

In file `ibm-zos-core-ee/execution-environment.yml` place this content into it:

``` yml
---
version: 3

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel8:latest

dependencies:
  galaxy: requirements.yml
```

In file `ibm-zos-core-ee/execution-environment.yml` place this content into it:

``` yml
collections:
  - name: ibm.ibm_zos_core
```

`execution-environment.yaml` contents

Notice that files `requirements.txt` and `bindep.txt` were not used, this is because at this point of the tutorial, we want to leverage these same files that are already included in the `ibm.ibm_zos_core` collection. If we were installing multiple collections and some collections did not include these files or if we wanted to override them, we would created them at the same time and place them alongside the others in directory `ibm-zos-core-ee`.


## Build the Ansible Execution Environment.

With the directory and files created in the prior step, its time to activate the Python virtual environment where we will be running `ansbile-builder` from to create the image.

### Activate the Python virtual environment with command:

``` shell
source ansible-z-builder-venv/bin/activate
```

### Navigate to the execution environment folder `ibm-zos-core-ee` with command:

``` shell
cd ibm-zos-core-ee
```

### Build the Execution Environment

Run `ansible-builder` with command:

``` shell
ansible-builder build --tag ddimatos/ibm-zos-core-ee
```

If you would like to add additional verbosity to appear in the STDOUT, you can control the verbosity levels 1 - 3 by adding `-v 3` to the command:

``` shell
ansible-builder build --tag ddimatos/ibm-zos-core-ee -v 3
```

The execution environnement is now built, you can deactive the Python virtual environment with command: 

``` shell
deactivate
```

At this point, a new directory will have been created named `context` containing directory `_build` and file `Containerfile` in directory `ibm-zos-core-ee`.

### View the Execution Environment Container

To view the recently crated execution environment container with command:

``` shell
podman images
```

If you want to upload the execution environment to Quay or Docker Hub, you can do so with the command:

Podman:

``` shell
podman push ddimatos/ibm-zos-core-ee
```

Docker Hub
``` shell
quay push ddimatos/ibm-zos-core-ee
```
