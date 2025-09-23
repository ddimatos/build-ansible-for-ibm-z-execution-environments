# Create Custom Ansible Execution Environments using Ansible Builder for IBM Z

This document will demonstrate using `ansible-builder` to configure and build a custom **Ansible for IBM Z** execution environment that we will then use to run a simple playbook using the execution environment with ``ansible-navigator``. The steps in this guide are performed on an ARM based MacBook, you may need to adjust commands to meet system requirements,the majority of this document will predominately apply to creating execution environments.

When `ansible-builder` begins building the container, it will rely on either Podman or Docker to detect the architecture of the host system and will pull and build images that are compatible with the hosts architecture. If a multi-architecture image is available, it will automatically choose the correct variant.

Red Hat has published base execution environments for these architectures:

- **AMD64** - a 64-bit extension of the x86 architecture primarily used in desktops and servers offering broader backward compatibility.
- **ARM64** - a 64-bit RISC (Reduced Instruction Set Computer) architecture used in mobile devices, embedded devices, servers and laptops offering power efficiency and performance per watt.
- **ppc64le** - a 64-bit architecture capable of running either big-endian or little-endian addressing IBM POWER8 series or later.
- **s390x** - a 64-bit Z architecture for mainframe computing.

## Why use an Execution Environment

Customizing execution environments provides enterprise operations a consistent and tested configuration that can be used during the development cycle and later taken to production without any concern that the production system may behave differently with varying dependencies. Execution environments also provide organizations the mechanics needed to ensure all software in production has been vetted and properly scanned without the concern the container will change.

There are several ways to use containerized execution environments, some of those are listed below.
- Ansible Automation Platform
- Ansible Navigator
- Localized development
- CI/CD pipelines
- AWX




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
python3 -m venv ansible-z-venv
```

Now there is a folder named `ansible-z-venv` for which you can now activate the Python virtual environment with command:

``` sh
source ansible-z-venv/bin/activate
```

## Install ansible-builder into the Python Virtual Environment

Activate the Python virtual environment with command:
``` sh
source ansible-z-venv/bin/activate
```

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

Ansible for IBM Z has several collections, the step by step portion of this guide, will create an execution environment that will include all six collections. If you don't need them all, you can simply remove their entry in the ``requirements.yml``.


## Ansible Execution Environment Files

You will need to create several files that `ansible-builder` will use to build the Ansible execution environment image. In our example, not all files are necessary because the Ansible for IBM Z collections include the necessary needed by ansible builder. For reference the files that will be created are `execution-environment.yml` and `requirements.yml`, the remaining files are described below for reference only. 

* `bindep.txt` - file that specifies system-level dependencies needed for an Ansible Execution Environment (EE)
* `requirements.txt` - The Python dependencies required for the collections identified in `requirements.yaml` and any others needed for proper execution a playbook.
* `requirements.yml` = The Ansible Collections to be installed into the execution environment needed for proper execution of a playbook.
* `execution-environment.yml` - defines the operations that `ansible-builder` will perform when creating the execution environment including the base image, dependencies, build steps, files and configurations.

### Create a directory and files needed for the execution environment.
In this next section, we will set up the directories and files necessary to build an execution environment for the collection.

Create directory with command:

``` shell
mkdir ibm-zos-ee
```

Create files with commands:

``` shell
touch ibm-zos-ee/execution-environment.yml
touch ibm-zos-ee/requirements.yml
```

Populate file  `ibm-zos-ee/execution-environment.yml` , with this content:

``` yml
---
version: 3

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest
    # -----------------------------------------------------------------------------
    # Using registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel9:latest will result
    # in a dependency error for botocore 1.25.6 that depends on urllib3<1.27 and >=1.25.4 ansible ee
    # because zhmclient is requesting `urllib3>=2.5.0; python_version >= '3.9'` and botcore only
    # supports urlib3 2.x.x when Python is >=3.10
    # -----------------------------------------------------------------------------

dependencies:
  galaxy: requirements.yml

options:
  package_manager_path: /usr/bin/microdnf
```


Note, the option `package_manager_path` is set to prevent the container manager for being unable to find the package manager which typically results in the following error.

``` shell
/output/scripts/assemble: line 163: /usr/bin/dnf: No such file or directory
Error: building at STEP "RUN /output/scripts/assemble": while running runtime: exit status 127
```

Populate file `ibm-zos-ee/requirements.yml` with this content:

``` yml
collections:
  - name: ibm.ibm_zos_core
  - name: ibm.ibm_zos_ims
  - name: ibm.ibm_zos_cics
  - name: ibm.ibm_zos_sysauto
  - name: ibm.ibm_zosmf
  - name: ibm.ibm_zhmc
```

 When `ansible-navigator` builds the container, it will pull these collections from Galaxy, if you want to pull from Ansible Automation Platform (AAP) you would need to update ansible.cfg to point AAP. Another option is to build from source, supposed you wanted to include the latest changes from the zhmc collection in Github, you can do so by using this entry in `requirements.yml`:

``` yml
collections:
  - name: ibm.ibm_zos_core
  - name: ibm.ibm_zos_ims
  - name: ibm.ibm_zos_cics
  - name: ibm.ibm_zos_sysauto
  - name: ibm.ibm_zosmf
  - name: https://github.com/zhmcclient/zhmc-ansible-modules.git
    type: git
    version: master
```

## Create the Ansible Execution Environment

With the directory and files created in the prior step, its time to activate the Python virtual environment where we will be running `ansbile-builder` from to create the image.

Activate the Python virtual environment with command:

``` shell
source ansible-z-venv/bin/activate
```

Navigate to the execution environment folder `ibm-zos-ee` with command:

``` shell
cd ibm-zos-ee
```

Run `ansible-builder` with command:

``` shell
ansible-builder build --tag ddimatos/ibm-zos-ee
```

If you would like to add additional verbosity to appear in the STDOUT, you can control the verbosity levels 1 - 3 by adding `-v 3` to the command:

``` shell
ansible-builder build --tag ddimatos/ibm-zos-ee -v 3
```

You should see the build complete with the message below, this can take some time, hence its best to run with verbosity so you can see the build process. 

```
Complete! The build context can be found at: /private/tmp/ibm-zos-core-ee/context
```

The execution environnement is now built, you can deactivate the Python virtual environment with command: 

``` shell
deactivate
```

At this point, a new directory will have been created named `context` containing directory `_build` and file `Containerfile` in directory `ibm-zos-ee`.

## View the Execution Environment Container

To view the recently crated execution environment container with command:

``` shell
podman images
```

If you have access to upload the execution environment to Quay or Docker Hub, you can do so with the appropriate following commands. If you do upload your image, you will need to update file `ansible-navigator.yml` with the appropriate values to point to the remote repository. 

Podman:

``` shell
podman push ddimatos/ibm-zos-ee
```

Docker Hub
``` shell
quay push ddimatos/ibm-zos-ee
```

Output:
```
Name                     Version        Shadowed   Type        Path
0│ansible.builtin          2.16.14        False      contained   /usr/lib/python3.11/site-packages/ansible
1│ibm.ibm_zhmc             1.10.0-dev1    False      contained   /usr/share/ansible/collections/ansible_collections/ibm/ibm_zhmc
2│ibm.ibm_zos_cics         2.2.1          False      contained   /usr/share/ansible/collections/ansible_collections/ibm/ibm_zos_cics
3│ibm.ibm_zos_core         1.14.1         False      contained   /usr/share/ansible/collections/ansible_collections/ibm/ibm_zos_core
4│ibm.ibm_zos_ims          1.3.1          False      contained   /usr/share/ansible/collections/ansible_collections/ibm/ibm_zos_ims
5│ibm.ibm_zos_sysauto      1.1.0          False      contained   /usr/share/ansible/collections/ansible_collections/ibm/ibm_zos_sysauto
6│ibm.ibm_zosmf            1.5.0          False      contained   /usr/share/ansible/collections/ansible_collections/ibm/ibm_zosmf
```

# Run the Execution Environment

Activate the Python virtual environment with command:
``` sh
source ansible-z-venv/bin/activate
```

Install ansible-navigator, the command-line tool and a text-based user interface for creating, reviewing, running and troubleshooting Ansible execution environments.

``` sh
pip3 install ansible-navigator
```

Create files with commands:

``` shell
touch ibm-zos-ee/ansible-navigator.yml
touch ibm-zos-ee/inventory.yml
touch ibm-zos-ee/test.yml
```

Populate file  `ibm-zos-ee/ansible-navigator.yml` , with this content which informs `ansible-navigator` to use the local container at `localhost/ddimatos/ibm-zos-core-ee:latest` when running a playbook with the execution environment. 

``` yml
---
ansible-navigator:
  ansible:
    inventory:
      entries:
      - inventory.yml
  execution-environment:
    container-engine: podman
    enabled: true
    image: localhost/ddimatos/ibm-zos-core-ee:latest
    pull:
      policy: 'never'
  logging:
    level: debug
```

Populate file  `ibm-zos-ee/inventory.yml` , with this content which is used by Ansible to connect to the managed node. Inventory should be tailored to suite your needs.
``` yml
zsystem:
  hosts:
    zvm:
      ansible_password: change-me
      ansible_user: change-me
      ansible_host: change-me
      ansible_python_interpreter: "/allpython/3.11-3/usr/lpp/IBM/cyp/v3r11/pyz/bin/python3"
```

Populate file  `ibm-zos-ee/test.yml` , with this sample playbook content. The playbook will issue two pings, one from `ansible.builtin` and one from the `ibm_zos_core` collection (`zos_ping`). This playbook is all inclusive in that I wanted to keep it simple and included all the environment variables, you may need to adjust the various environment variables to match your deployments.

``` yml
---
- name: Test ping playbook
  hosts: all
  collections:
    - ibm.ibm_zos_core
  vars:
    ZOAU: "/zoau/v1.3.5.1"
    PYZ: "/allpython/3.11-3/usr/lpp/IBM/cyp/v3r11/pyz"
    PYZ_VERSION: "3.12"
    ansible_pipelining: true
    ansible_python_interpreter: "{{ PYZ }}/bin/python3"
    ansible_scp_extra_args: "-O"
  environment:
    _BPXK_AUTOCVT: "ON"
    ZOAU_HOME: "{{ ZOAU }}"
    PYTHONPATH: "{{ ZOAU }}/lib/{{ PYZ_VERSION}}"
    LIBPATH: "{{ ZOAU }}/lib:{{ PYZ }}/lib:/lib:/usr/lib:."
    PATH: "{{ ZOAU }}/bin:{{ PYZ }}/bin:/bin:/var/bin"
    _CEE_RUNOPTS: "FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)"
    _TAG_REDIR_ERR: "txt"
    _TAG_REDIR_IN: "txt"
    _TAG_REDIR_OUT: "txt"
    LANG: "C"
    PYTHONSTDINENCODING: "cp1047"


  tasks:
    - name: Test ansible.builtin ping.
      ansible.builtin.ping:
      register: result

    - name: Print ansible.builtin ping result.
      ansible.builtin.debug:
        msg: "{{ result }}"

    - name: Test ibm.ibm_zos_core zos_ping.
      zos_ping:
      register: result

    - name: Print ibm.ibm_zos_core zos_ping result.
      ansible.builtin.debug:
        msg: "{{ result }}"
```

command:
``` shell
ansible-navigator run test.yml -i inventory -m stdout
```

Common Ansible-navigator commands:

* ``ansible-navigator`` - launches ``ansible-navigator`` in interactive mode with a textual interface. To run a playbook with the command, use the following syntax ``ansible-navigator run <playbook>`` , e.g., ``ansible-navigator run test.yml``
* ``ansible-navigator run test.yml -m stdout`` - runs the playbook in non-interactive (stdout) mode, displaying the output directly to the terminal, similar to when running a playbook ad-hoc with command ``ansible-playbook``.
* ``ansible-navigator inventory -i inventory`` -  runs the playbook for the specified inventory, without this, ``ansible-navigator`` will look locally to where the execution environment is located, e.g., ``Unable to parse /private/tmp/inventory.yml as an inventory source``.
* ``ansible-navigator config`` - displays the 'default', 'source',  and 'current' Ansible configurations.
* ``ansible-navigator doc ibm.ibm_zos_core.zos_ping`` - displays documentation for the specified module.
* 
* ``ansible-navigator images`` - Lists the available container images for execution environments, along with their details and status.
* ``cat ansible-navigator.log`` - Displays the Ansible Navigator log, which contains information about the tool’s operation, including any errors or warnings.
* ``ansible-navigator replay test-artifact-2025-09-21T04:39:58.605761+00:00.json`` - “Replays” a previous ansible-navigator “run” ``{playbook_dir}/{playbook_name}-artifact-{time_stamp}.json``
* ``ansible-navigator run test.yml -i inventory --eei ansible-navigator-demo-ee`` - Specifies both an inventory and execution environment in a “run”.
* ``ansible-navigator collections`` Lists all installed Ansible collections, including their name, version, and installation path (while some recent versions appear to have a bug, the older stable 1.1.0 is still working OK and we have no reason to believe the functionality will not be retained going forward, so we are including it here).

