# Simple Deployment Guide for the Reaction Platform on Digital Ocean

### Overview

This deployment guide's purpose is to provide a simple and easy guide on how to deploy the Reaction platform for evaluation purposes or small deployments. This guide is not meant to generate an enterprise production grade deployment. This deployment guide does not use Kubernetes, instead, Docker Compose is used to manage containers.

### Requirements

- A Linux host with at least 2GB of RAM, this guide uses a DigitalOcean droplet
- A registered domain
- A DNS manager that supports Certification Authority Authorization (CCA) records, such as Digital Ocean
- Docker
- Docker Compose
- Git
- Node
- Yarn
- Some familiarity with [Traefik](https://traefik.io)

### Reaction Platform Services Overview

###### Reaction GraphQL API
The [Reaction GraphQL API](https://github.com/reactioncommerce/reaction) service provides the interface to the Reaction core functionality.

###### Storefront
The [example storefront](https://github.com/reactioncommerce/example-storefront) service provides the public facing storefront interface that customers will interact with.

###### Reaction Admin
The [Reaction Admin](https://github.com/reactioncommerce/reaction-admin) service is a Meteor(currently being migrated off Meteor) application that provides the admin UI to manage products, orders etc. 

###### Reaction Hydra
The [Reaction Hydra](https://github.com/reactioncommerce/reaction-hydra) is a OAuth2 token server that is integrated with the Reaction development platform.

###### Reaction Identity
The [Reaction Identity](https://github.com/reactioncommerce/reaction-identity) service is a Meteor application that provides authentication and authorization services.

### Architecture Diagram

![alt text](docs/img/reaction_deployment_diagram.svg "Reaction Platform Architecture")

## Getting Started

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

Each of your domains will need an `A` DNS record that resolves to your host's IP. Further, in order to obtain SSL certificates for your sub-domains, you will need a DNS manager tha supports [CAA](https://support.dnsimple.com/articles/caa-record/) records. It's recommend to use DigitalOcean's free [DNS manager](https://www.digitalocean.com/docs/networking/dns/overview/)

Further, you will need a [DigitalOcean Auth token](https://www.digitalocean.com/docs/api/create-personal-access-token/) to generate CAA records for your sub-domains.



# Automated Server Configuration

In order to expedite the installation of server dependencies, Ansible will be used to automate most of the server configuration.

### Getting started

**Prepare the Remote Host**

Create a new droplet in you DigitalOcean dashboard with at least 2GB of RAM. When creating the droplet, either select an existing SSH key to login or click on the "New SSH Key" under the authentication section  and copy your public SSH key from your local computer.

Copy the newly created IP address and verify that you can login into the new server by executing:

```
ssh root@XXX.XXX.XXX.XXX
```

**Generate a secure password for HTTP Basic Auth**

On your remote host install the `htpasswd` utility to generate an encrypted password for the Traefik admin UI.

```
sudo apt-get install apache2-utils
```

Generate a secure password and save it to a secure location. Substitute `my_secure_password` with the password you would like to use for the Traefik admin UI.

```
htpasswd -nb reaction my_secure_password
```

The out put will look like this:

```
reaction:$apr1$pPP6CJbi$5VavZVj7DVbLyAe1TkrCm1
```

NOTE: Sometimes, `htpasswd` will generate passwords that contain `/` or `\` which can be problematic, execute command until a password without slashes is generated.

The credentials will be used in Traefik's configuration to setup HTTP Basic Authentication

**Prepare Control Node(local computer)**

Ansible requires a control node, which is a computer that manages a remote host. This guide will assumes a Mac laptop/desktop as the control node. 

Install Ansible using [homebrew](https://brew.sh), this guide assumes some familiarity with Ansible, if you need an introduction to basic concepts click [here](https://www.ansibletutorials.com).

`brew install ansible`

Also install python3 to avoid deprecation warnings,

`brew install python3`

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

**Set Ansible Environment Variables**

Before executing the Ansible playbook, it's necessary to set variables that are specific to your deployment. Find the
`vars` section in the `reaction.yml` playbook and update as necessary, below is a list of the variable 
that need to be updated and a description of each.

| Variable               | Description                                                                 |
| ---------------------- | ----------------------------------------------------------------------------|
| create_user            | The name of the Linux user that will be created, suggested value `reaction` |
| do_auth_token          | The Authentication token for the Digital Ocean API                          |
| traefik_admin_password | The encrypted password generated with htpasswd                              |
| email                  | An email address to receive SSl certificate notifications                   |
| domain                 | Your registered domain                                                      |

For the rest of the variables, the default values should be used, DO NOT change otherwise, the playbook might fail.

**Execute the playbook**

Now it's time to execute the `reaction.yml` playbook, which automates most of the tedious server configuration tasks. Move into the `playbooks` and execute the following command:
```
cd playbooks
```
Execute playbook
```
ansible-playbook reaction.yml -l reaction.server
```

NOTE: the `-l reaction.server` limits the execution of the playbook to the `reaction.server` host.

**Create Primary Shop**

At this point the Reaction GraphQL API, Example Storefront, Reaction Admin, Reaction Identity and Hydra should be accessible over the internet.

To create the primary shop login into the Reaction Admin at the following URL, first substitute the `example.com` with your actual domain:
```
https://admin.example.com
```

Login with default username/password: `admin@localhost` and password: `r3@cti0n`, and follow instructions to create the primary shop.

Further, the `GraphQL API` explorer will be available at `https://api.example.com/graphql`.


