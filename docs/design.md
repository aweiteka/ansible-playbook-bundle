# Design

## Overview
An [Ansible Playbook Bundle (APB)](https://github.com/fusor/ansible-playbook-bundle)
borrows several concepts from the [Nulecule](https://github.com/projectatomic/nulecule)
and the [Atomicapp](http://www.projectatomic.io/docs/atomicapp/) project, namely the concept of a short
lived container with the sole purpose of orchestrating the deployment of the intended application. For the case
of APB, this short lived container is the APB; a container with an Ansible runtime environment
plus any files required to assist in orchestration such as playbooks, roles, and extra dependencies.
Specification of an APB is intended to be lightweight, consisting of several named playbooks and a
metadata file to capture information such as parameters to pass into the application.

## Workflow
APB is broken up into the following steps.

  1. [Preparation](#preparation)
     * [APB init](#apb-initialization)
     * [Spec File](#spec-file)
     * [Actions](#actions) (provision, deprovision, bind, unbind)
  1. [Build](#build)
  1. [Deploy](#deploy)

### Preparation

#### APB Initialization
![Prepare](images/apb-prepare.png)
The first step to creating an APB is to run the `apb init` command, which will create the required skeleton directory structure, and a few required files (e.g. `apb.yml` spec file) for the APB.

##### Directory Structure
The following shows an example directory structure of an APB.
```bash
example-apb/
├── Dockerfile
├── apb.yml
└── roles/
│   └── example-apb-openshift
│       ├── defaults
│       │   └── main.yml
│       └── tasks
│           └── main.yml
└── playbooks/
    └── provision.yml
    └── deprovision.yml
    └── bind.yml
    └── unbind.yml
```

#### Spec File

The `apb init` will create an example APB Spec `apb.yml` File as shown below:
```yml
name: my-apb
image: <docker-org>/my-apb
description: "My New APB"
bindable: false
async: optional
parameters: []
dependencies: []
```
The spec file will need to be edited for your specific application.

For example, the `etherpad-apb` spec file looks as follows:
```yml
name: etherpad-apb
image: ansibleplaybookbundle/etherpad-apb
description: Note taking web application
bindable: true
async: optional
metadata:
  displayName: "Etherpad (APB)"
  longDescription: "An apb that deploys Etherpad Lite"
  imageUrl: "https://translatewiki.net/images/thumb/6/6f/Etherpad_lite.svg/200px-Etherpad_lite.svg.png"
  documentationUrl: "https://github.com/ether/etherpad-lite/wiki"
parameters:
  - mariadb_name:
      title: MariaDB Database Name
      type: string
      default: etherpad
  - mariadb_user:
      title: MariaDB User
      type: string
      default: etherpad
      maxlength: 63
  - mariadb_password:
      title: MariaDB Password
      description: A random alphanumeric string if left blank
      type: string
      default: admin
  - mariadb_root_password:
      title: Root Password
      description: root password for mariadb 
      type: string
      default: admin
required:
  - mariadb_name
  - mariadb_user
dependencies:
  - docker.io/mariadb:latest
  - docker.io/tvelocity/etherpad-lite:latest
```

The `metadata` field is optional and used when integrating with the origin service catalog.
For an APB that does not have any parameters, `parameters` field would look like:
```yml
parameters: []
```

#### Actions
The following are the actions for an APB. At a minimum, an APB must implement the `provision` and `deprovision` actions.
 * provision.yml
   * Playbook called to handle installing application to the cluster
 * deprovision.yml
   * Playbook called to handle uninstalling
 * bind.yml
   * Playbook to grant access to another service to use this service, i.e. generates credentials
 * unbind.yml
   * Playbook to revoke access to this service

The required named playbooks correspond to methods defined by the [Open Service Broker API](https://github.com/openservicebrokerapi/servicebroker). For example, when the
Ansible Service Broker needs to `provision` an APB it will execute the `provision.yml`.

After the required named playbooks have been generated, the files can be used directly to test management of the
application. A developer may want to work with this directory of files, make tweaks, run, repeat until they are
happy with the behavior. They can test the playbooks by invoking Ansible directly with the playbook and any
required variables.

### Build
The build step is responsible for building a container image from the named playbooks for distribution.
Packaging combines a base image containing an Ansible runtime with Ansible artifacts and any dependencies required
to run the playbooks. The result is a container image with an ENTRYPOINT set to take in several arguments, one of
which is the method to execute, such as provision, deprovision, etc.

![Package](images/apb-package.png)

### Deploy
Deploying an APB means invoking the container and passing in the name of the playbook to execute along with any
required variables. It’s possible to invoke the APB directly without going through the Ansible Service Broker.
Each APB is packaged so it’s ENTRYPOINT will invoke Ansible when run. The container is intended to be short-lived,
coming up to execute the Ansible playbook for managing the application then exiting.

In a typical APB deploy, the APB container will provision an application by running the `provision.yml` playbook
 which executes a deployment role. The deployment role is responsible for creating the OpenShift resources,
 perhaps through calling `oc create` commands or leveraging Ansible modules. The end result is that the APB runs
 Ansible to talk to OpenShift to orchestrate the provisioning of the intended application.

![Deploy](images/apb-deploy.png)
