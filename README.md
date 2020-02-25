# Simple Deployment Guide for the Reaction Platform on Digital Ocean

### Overview

This deployment guide's purpose is to make it easy for Reaction users to deploy the Reaction platform for evaluation purposes or small deployments. This guide is not meant to generate an enterprise production grade deployment. This deployment guide does not use Kubernetes, instead, Docker Compose is used to manage containers.

### Requirements

- A Linux host with at least 2GB of RAM, this guide uses a DigitalOcean droplet
- A DNS manager that supports Certification Authority Authorization (CCA) records, such as Digital Ocean
- Docker
- Docker Compose
- Git
- Node
- Yarn
- A registered domain
- Some familiarity with [Traefik](https://traefik.io)

### Architecture Diagram

![alt text](docs/img/reaction_deployment_diagram.svg "Reaction Platform Architecture")

## Getting Started

## Exposing Reaction services on the public internet

Reaction services will be exposed to the public using [Traefik](https://traefik.io), which is a cloud native router. Traefik will act as a reverse proxy that will route traffic to Docker containers. As stated above, you will need a registered domain to complete this step, as it will be necessary to manage DNS records for it.

This guide will use the following sub-domains, where `example.com` will need to be substitute it with your domain: 

| subdomain              | description                           |
| ---------------------- | ------------------------------------- |
| api.example.com        | The Reaction  GraphQL API             |
| storefront.example.com | The example storefront                |
| admin.example.com      | The Reaction admin interface          |
| hydra.example.com      | Hydra OAuth 2.0 server                |
| identity.example.com   | The Reaction Identity service         |
| traefik.example.com    | Traefik's admin UI                    |


# Automated Server Configuration

In order to expedite the installation of server dependencies, Ansible will be used to automate most of the server configuration.

### Getting started

**Prepare control node(local computer)**

Ansible requires a control node, which is a computer that manages a remote host. This guide will assumes a Mac laptop/desktop as the control node. 

Install Ansible using [homebrew](https://brew.sh), this guide assumes some familiarity with Ansible, if you need an introduction to basic concepts click [here](https://www.ansibletutorials.com).

`brew install ansible`

Also install python3 to avoid deprecation warnings,

`brew install python3`

**Prepare remote host**

Create a new droplet in you DigitalOcean dashboard with at least 2GB of RAM. When creating the droplet, either select an existing SSH key to login or click on the "New SSH Key" under the authentication section  and copy your public SSH key from your local computer.

Copy the newly created IP address and verify that you can login into the new server by executing:

```
ssh root@XXX.XXX.XXX.XXX
```

**Manage the remote host with Ansible**

On the control node(i.e. a developer's machine) create an inventory file in which `python3` is specified as the interpreter. On your machine, create a new file at named `hosts` at `/etc/ansible`.

Create inventory file
```
touch /etc/ansible/hosts
```

Add the following content to the inventory file:
```
[servers]
reaction.server

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
[web]
```

Edit your hosts file  
```
sudo vim /etc/hosts
```

and add an entry for the DigitalOcean droplet,

```
XXX.XXX.XXX.XXX reaction.server
```

Verify that Ansible can communicate with your remote host by executing:

```
ansible all -m ping -u root
```

Your should see output similar to:

```
reaction.dev | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

**Execute the playbook**

Now it's time to execute the `reaction.yml` playbook, which automates most of the tedious server configuration tasks. Move into the `playbooks` and execute the following command:
```
ansible-playbook reaction.yml -l reaction.server
```

NOTE: the `-l reaction.server` limits the execution of the playbook to the `reaction.server` host.

