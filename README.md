# Create Custom Ansible Execution Environments using Ansible Builder for IBM Z

This document demonstrates using `ansible-builder` to configure and build a custom **Ansible for IBM Z** execution environment and using it to run a playbook with ``ansible-navigator``. The steps in this guide are performed on an ARM based MacBook, you may need to adjust commands to meet system requirements,the majority of this document will apply to creating execution environments for all platforms.

When `ansible-builder` begins building the container, it will rely on either Podman or Docker to detect the architecture of the host system and will pull and build images that are compatible with the systems architecture. If a multi-architecture image is available, it will automatically choose the correct variant.

Red Hat has published base execution environments for these architectures:

- **AMD64** - a 64-bit extension of the x86 architecture primarily used in desktops and servers offering broader backward compatibility.
- **ARM64** - a 64-bit RISC (Reduced Instruction Set Computer) architecture used in mobile devices, embedded devices, servers and laptops offering power efficiency and performance per watt.
- **ppc64le** - a 64-bit architecture capable of running either big-endian or little-endian for IBM POWER8 series or later.
- **s390x** - a 64-bit Z architecture for mainframe computing.

## Why use an execution environment

Customizing execution environments provides enterprise operations a consistent and tested configuration that can be used during the development cycle and later taken to production without any concern that the production system may behave differently with varying dependencies. Execution environments also provide organizations the mechanics needed to ensure all software in production has been vetted and properly scanned without the concern the container will change.

There are several ways to use containerized execution environments, some of those are listed below.
- Ansible Automation Platform
- Ansible Navigator
- Localized development
- CI/CD pipelines
- AWX

# Setting up the local environment to create the execution environment.

To build the execution environments, create a Python virtual environment that provide a dedicated space for project-specific dependencies, preventing conflicts with other projects or the systems Python installation.

## Python

The requirements for [ansible-builder](https://ansible.readthedocs.io/projects/builder/en/stable/installation/#requirements) version 3.x currently are python 3.9 or later.

On your system, check the level of python with:

``` sh
python --version
```

If you don't have Python installed or are running an older version, you can install a later version with the  `brew` command:

``` sh
brew install python@3.13
```

If you need to later revert to older version of python you can `unlink` the recently installed version and `link` the preferred version with commands:
``` sh
brew unlink python@3.13
brew link python@3.9
```

## The Container manager

To build execution environments, you need either [Podman](https://podman.io/) or [Docker](https://www.docker.com/). You can check if you have either installed with on of the following commands:

``` sh
podman info
docker info
```

This guide uses [Podman Desktop](https://podman-desktop.io).

## Create a Python virtual environment (venv):

Create the Python virtual environment serving as a sandbox to build the execution environment.

``` sh
python3 -m venv ansible-z-venv
```

Now there is a folder named `ansible-z-venv` containing the virtual environment that will activate with command:

``` sh
source ansible-z-venv/bin/activate
```

## Install ansible-builder into the Python virtual environment

With the virtual environment activated from the previous step, install `ansible-builder` into the Python virtual environment with command:

``` sh
pip3 install ansible-builder
```

Now that you have installed `ansible-builder` you can deactivate the Python virtual environment with command:

``` sh
deactivate
```

# Execution Environments

If you have been following along, you now have a virtual environment (ansible-z-venv) with `ansible-builder` installed and a container manager, e.g., Podman. `ansible-builder` will create the execution environment with the assistance of the the container manager and configuration files.

If you are unfamiliar with container managers, it will help to understand how they aid in creating an execution environment. Both Podman and Docker use files , `Containerfile` or `Dockerfile` that contain the instructions to create a container image. In our guide, we will use similar configuration files tailored for `ansible-builder`. A container image is a replica of a container only it is not active and can be used to create additional copies of the container. A container is a collection of code and packages bundled such that it ca be run on a variety of platforms, which is what an Ansible execution environment is. 

## Building an Execution Environment

**Ansible for IBM Z** offers several certified collections, the step by step portion of this guide, will create an execution environment that will include all the collections. If you don't need them all, you can simply remove their entry in the ``requirements.yml``.

## Ansible Execution Environment Files

Create several files that `ansible-builder` will use to build the Ansible execution environment image. In our example, not all files are necessary because the **Ansible for IBM Z** collections include several of them. For reference, the required files are  that will be created are `execution-environment.yml` and `requirements.yml`, the remaining files are for reference only.

* `execution-environment.yml` - specifies the operations `ansible-builder` will perform to create the execution environment including the base image, dependencies, build steps, files and configurations.
* `requirements.yml` - specifies the list of Ansible Collections to be installed into the execution environment.
* `bindep.txt` - specifies the system-level dependencies needed for the Ansible execution environment.
* `requirements.txt` - specifies the Python dependencies required by various collections listed in `requirements.yaml`.

## Create the required directory and files

In this section, create the directories and files necessary to build an execution environment.

Create directory with command:

``` shell
mkdir ibm-zos-ee
```

Create files with commands:

``` shell
touch ibm-zos-ee/execution-environment.yml
touch ibm-zos-ee/requirements.yml
```

Populate file  `ibm-zos-ee/execution-environment.yml` , with content:

``` yml
---
version: 3

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest

dependencies:
  galaxy: requirements.yml

options:
  package_manager_path: /usr/bin/microdnf
```

Populate file `ibm-zos-ee/requirements.yml` with content:
``` yml
collections:
  - name: ibm.ibm_zos_core
  - name: ibm.ibm_zos_ims
  - name: ibm.ibm_zos_cics
  - name: ibm.ibm_zos_sysauto
  - name: ibm.ibm_zosmf
  - name: ibm.ibm_zhmc
```

Notes:
* Avoid using ee-supported 'registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel9:latest'
  because it will result in a dependency error between zhmclient and botocore. Where botocore 1.25.6
  depends on urllib3<1.27 and >=1.25.4 and zhmclient raises it to urllib3>=2.5.0 when python_version >=3.9.
  Package botocore only supports urlib3 2.x when Python is >=3.10.
* The option `package_manager_path` is defined to prevent the container manager from being unable to find
  it which typically results in the following error:
    ```
    /output/scripts/assemble: line 163: /usr/bin/dnf: No such file or directory
    Error: building at STEP "RUN /output/scripts/assemble": while running runtime: exit status 127
    ```

 When `ansible-navigator` builds the container, it will pull collections from Galaxy, if you want to pull from Ansible Automation Platform (AAP), create and then [update ansible.cfg](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/latest/html/creating_and_using_execution_environments/assembly-using-builder) to pull from AAP. This exercise will be left to the user to explore.

If you are interested installing collections from source, you can substitute the public repository in the `name` field, specify the `type` and identify the branch with the `version`. This is helpful in cases where the latest source in a branch has a change you want to try but the collection has not been released to Galaxy or AAP yet, thus providing you with an execution environment with the latest source. Below is an example of pulling the latest source from the zHMC collection. 

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

## Create the Ansible execution environment

With the directory and files created in the prior step, now activate the Python virtual environment to run `ansbile-builder` to create the execution environment.

Activate the Python virtual environment with command:

``` shell
source ansible-z-venv/bin/activate
```

Navigate to the execution environment folder `ibm-zos-ee` with command:

``` shell
cd ibm-zos-ee
```

Run `ansible-builder` with added verbosity level 3. Verbosity is not required and levels are between 1 - 3, but I highly recommend it so that you followi along and watch the process. Some images can take considerable time to build and verbosity ensures the build is continuing. Use command:

``` shell
ansible-builder build --tag ddimatos/ibm-zos-ee -v 3
```

Note: You may run into the below error when ansible-builder connects to the Red Hat Container Registry and tries to authenticate.
```
Error: creating build container: initializing source docker://registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest: unable to retrieve auth token: invalid username/password: unauthorized: Please login to the Red Hat Registry using your Customer Portal credentials. Further instructions can be found here: https://access.redhat.com/RegistryAuthentication
```
For this error, authenticate first:
```
podman login registry.redhat.io
Username: <user>
Password: <password>
Login Succeeded!
```
You should see the build complete with the message below.

``` sh
Complete! The build context can be found at: /private/tmp/ibm-zos-core-ee/context
```

The execution environnement is now built, you can deactivate the Python virtual environment with command:

``` shell
deactivate
```

At this point, a new directory will have been created named `context` containing directory `_build` and file `Containerfile` under `ibm-zos-ee`. Earlier we discussed the purpose of `Containerfile`, feel free to browse its contents. 

## View the execution environment within the container manager

View the recently crated execution environment container with command:

``` shell
podman images
```

This is optional, if you have access to upload the execution environment to Quay or Docker Hub, you can do so with the following command:

Podman
``` shell
podman push ddimatos/ibm-zos-ee
```

Docker Hub
``` shell
quay push ddimatos/ibm-zos-ee
```

Note, if upload your image to a repository such as Quay.io, you will need to update file `ansible-navigator.yml` with the appropriate [pull policy](https://ansible.readthedocs.io/projects/navigator/settings/#pull-policy) to pull from repository. Current options are 'always', 'missing', 'never' or 'tag', in our example we use 'never' because our images is local.

# Run the execution environment

To run the execution environment we need to install `ansible-navigator`, begin by activating the Python virtual environment with command:
``` sh
source ansible-z-venv/bin/activate
```

Install `ansible-navigator`, the command-line tool with a text-based user interface for running and reviewing Ansible playbooks with an execution environment.
``` sh
pip3 install ansible-navigator
```

## Create ansible-navigator files

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

## Create Ansible playbook artifacts

Populate file  `ibm-zos-ee/inventory.yml` , with this content which is used by Ansible to connect to the managed node.
``` yml
zsystem:
  hosts:
    zvm:
      ansible_password: change-me
      ansible_user: change-me
      ansible_host: change-me
      ansible_python_interpreter: "/allpython/3.11-3/usr/lpp/IBM/cyp/v3r11/pyz/bin/python3"
```

Inventory should be tailored to suite your needs. You can skip configuring `ansible_password` if you [share your SSH keys](https://ansible.readthedocs.io/projects/navigator/faq/#ssh-keys) with the execution environment. The default behavior is keys using default names (id_rsa, id_dsa, id_ecdsa, id_ed25519, and id_xmss) are automatically mounted the execution environments default user home directory.

Populate file  `ibm-zos-ee/test.yml` , with this sample playbook content:

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

The playbook will issue two pings, one from `ansible.builtin` and one from the `ibm_zos_core` collection (`zos_ping`). This playbook is all inclusive in that I wanted to keep it simple and included all the environment variables, you may need to adjust the various environment variables to match your deployments.

## Run the playbook with the execution environment
command:
``` shell
ansible-navigator run test.yml -i inventory.yml -m stdout
```

Optionally if you would like verbose:
command:
``` shell
ansible-navigator run test.yml -i inventory.yml -m stdout -vvvv
```

Note, in the event you encounter a similar error 'hostkeys_find: found ssh-ed25519 key under different name/addr at /root/.ssh/known_hosts', it means the SSH client has found an SSH host key for a server (specifically an ssh-ed25519 key) in your known_hosts file, but this key is associated with a different hostname or IP address than the one you are currently trying to connect to. You can add your SHH key to the agent with the following commands.

Error:
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

# Helpful ansible-navigator commands:

The ansible-navigator commands are helpful when diagnosing or viewing the content contained in the execution environment.

* ``ansible-navigator`` - launches ``ansible-navigator`` in interactive mode with a textual interface. To run a playbook with the command, use the following syntax ``ansible-navigator run <playbook>`` , e.g., ``ansible-navigator run test.yml``
* ``ansible-navigator run test.yml -m stdout`` - runs the playbook in non-interactive (stdout) mode, displaying the output directly to the terminal, similar to when running a playbook ad-hoc with command ``ansible-playbook``.
* ``ansible-navigator inventory -i inventory`` -  runs the playbook for the specified inventory, without this, ``ansible-navigator`` will look locally to where the execution environment is located, e.g., ``Unable to parse /private/tmp/inventory.yml as an inventory source``.
* ``ansible-navigator config`` - displays the 'default', 'source',  and 'current' Ansible configurations.
* ``ansible-navigator doc ibm.ibm_zos_core.zos_ping`` - displays documentation for the specified module.
* ``ansible-navigator images`` - lists execution environment containers along with details and status.
* ``cat ansible-navigator.log`` - displays the ansible-navigator log containing information about the tool’s execution, including any errors or warnings.
* ``ansible-navigator replay test-artifact-2025-09-21T04:39:58.605761+00:00.json`` - replays a previous ansible-navigator “run” using syntax: ``{playbook_dir}/{playbook_name}-artifact-{time_stamp}.json``
* ``ansible-navigator run test.yml -i inventory --eei ibm-zos-core-ee`` - specifies that the playbook, inventory and specific execution environment be used to run.  
* ``ansible-navigator collections`` - lists all installed Ansible collections, including name, version, and installation path, example output below:

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
