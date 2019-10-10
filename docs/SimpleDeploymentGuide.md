# Simple Deployment Guide

### Overview

This deployment guide's purpose is to make it easy for Reaction users to deploy the Reaction platform for evaluation purposes or small deployments. This guide is not meant to generate an enterprise production grade deployment. This deployment guide does not use Kubernetes, instead, Docker Composed is used to manage containers.

### Requirements

- A Linux host with at least 2GB of RAM, this guide uses a DigitalOcean droplet
- Docker
- Docker Compose
- Git
- Node
- Yarn
- Portainer
- Registered domain
- Some familiarity with [Traefik](https://traefik.io)

### Architecture

![alt text](img/reaction_deployment_diagram.svg "Reaction Platform Architecture")

### Getting Started

**Preparing the Linux host**

In this guide a DigitalOcean node will be used to host the Reaction Platform. If you don't yet have an account, create one at [digitalocean.com](https://digitalocean.com). Once you are signed into your account, create a new droplet using the Ubuntu 18.4 image with at least 1GB of RAM and then follow the instruction [here](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04) to prepare the host. Enable DigitalOceans [free firewall](https://www.digitalocean.com/docs/networking/firewalls/) and add inbound rules for SSH, HTTP, HTTPS and add your droplet to the firewall.

Next, install Make, Docker and Docker Compose.

Install make by executing: `sudo apt install build-essential`

Enable swap file by following instructions [here](https://www.cyberciti.biz/faq/linux-add-a-swap-file-howto/) use at least 4GB.

Install Docker by following instructions [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

Install Docker Compose by following instructions [here](https://docs.docker.com/compose/install/).

Install Node and NPM using NVM by following instructions [here](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-18-04#installing-using-nvm)

Install Yarn without recommended packages by executing, `sudo apt-get update && sudo apt-get install --no-install-recommends yarn`

Generate a new SSH key by following instructions [here](https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) and add it key to your GitHub account [settings](https://github.com/settings/keys).

OPTIONAL: add the following aliases to your `~/.bashrc` to execute Docker and Docker Compose commands efficiently: [Aliases](https://gist.github.com/willopez/3c8bb568a861cff29006eca4a4899601)

**Clone the Reaction Platform**

Clone the [Reaction platform](https://github.com/reactioncommerce/reaction-platform) repository to a directory on your host and execute the `make`. The reaction platform code bases will be downloaded, Docker images built and Docker networks created, this can take a few minutes.

```
git clone git@github.com:reactioncommerce/reaction-platform.git
```

Confirm that all containers were created and started successfully by executing: `docker ps -a | less -S`, the output should look similar to the output below:

```
CONTAINER ID        IMAGE                             COMMAND                  CREATED              STATUS
02b014978752        example-storefront_web            "tini -- ./.reaction…"   About a minute ago   Up About a minute
5a07bf085e4b        reaction_reaction                 "bash -c 'time meteo…"   2 minutes ago        Exited (1) 21 second
4bc8e09d8fcb        mongo:3.6.3                       "docker-entrypoint.s…"   3 minutes ago        Up 2 minutes
5b438a1bb711        oryd/hydra:v1.0.0-beta.9-alpine   "hydra serve all --d…"   8 minutes ago        Up 8 minutes
311659cec603        oryd/hydra:v1.0.0-beta.9-alpine   "hydra migrate sql -e"   8 minutes ago        Exited (0) 8 minutes
d6e932995efa        postgres:10.3                     "docker-entrypoint.s…"   8 minutes ago;
```

Further confirm reaction is running correctly by changing to the `reaction` directory(`cd reaction`) and executing `docker-compose logs -f reaction` the output should have a line at the end that reads similar to this: `18:43:02.831Z INFO Reaction: Reaction initialization finished: 9910ms`

**Expose Reaction to the outside world**

In this section Reaction and the example storefront will be exposed to the public using [Traefik](https://traefik.io), which is a cloud native router. Traefik will act as a reverse proxy that will route traffic to Docker containers. As stated above, you will need a registered domain to complete this step, as it will be necessary to manage DNS records for it.

This guide will use the following sub-domains:

| subdomain                 | description                           |
| ------------------------- | ------------------------------------- |
| reaction.yourdomain.com   | The Reaction core API and Operator UI |
| storefront.yourdomain.com | The example storefront                |
| hydra.yourdomain.com      | Hydra OAuth 2.0 server                |
| traefik.yourdomain.com    | Traefik's admin UI                    |

Each of your domains will need an `A` DNS record that resolves to your host's IP. Further, in order to obtain SSl certificates for your sub-domains, you will need a DNS manager tha supports [CAA](https://support.dnsimple.com/articles/caa-record/) records. I recommend using DigitalOcean's free [DNS manager](https://www.digitalocean.com/docs/networking/dns/overview/)

Further, you will need a [DigitalOcean Auth token](https://www.digitalocean.com/docs/api/create-personal-access-token/) to generate CAA records for your sub-domains.

Next, we will setup Traefik.

**Generate a secure password for HTTP Basic Auth**

Install the `htpasswd` utility to generate an encrypted password for the Traefik admin UI.

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

**Configure Traefik**

Clone the [proxy](git@github.com:reactioncommerce/proxy.git) repository to a directory on you host. This repo contains the necessary files to correctly configure Traefik as a reverse proxy for the Reaction Platform.

```
git clone git@github.com:reactioncommerce/proxy.git
```

`cd` into the `traefik` directory and open the `docker-compose.yml` file. Under the `environment` section, substitute `YOUR_DIGITALOCEAN_AUTH_TOKEN` with your actual DO Auth token, save and close. Further, substitute `REPLACE_WITH_PATH_TO_TRAEFIK_FOLDER` with the path to the traefik folder, i.e. `/home/your_user/proxy/traefik/`

Next, open the `traefik.toml` file and find the `entryPoints` section and substitute `YOUR_USER:YOUR_ENCRYPTED_PASSWORD` with actual value generated previously. Further, under the `acme` section substitute `YOUR_EMAIL` and `YOUR_DOMAIN` with your values. Use your TLD domain not your subdomain, as the CAA record will a wildcard record, meaning that it will generate SSL certificates for subdomains, i.e. `yourdomain.com`.

A wildcard CAA record must be generated in your
DNS provider with the following settings:

```
authority granted for: letsencrypt.org
tag: issuewild
flags: 0
```

Create the `web` Docker network, it will be the Docker network that Traefik will be on. And also create an `internal` Docker network for internal service communication.

```
docker create network web
docker create network internal
```

Within the `traefik` folder execute `chmod 600 traefik.toml` to set the correct permissions and `docker-compose up` to confirm the service starts successfully, the logs should display `Attaching to Traefik` and nothing else.

Now its time to use your browser and navigate to `traefik.yourdomain.com` and you should be presented with a browser prompt for credentials, use the user name and password you used in the section above. Note: use the un-encrypted version of your password. Upon successful entry, you should be presented with Traefik's admin UI.

**Expose Reaction Platform services**

In this section the Reaction Platform services will be exposed to the public web. Thi will be achieve by adding labels to each of the services that contain their corresponding subdomain.

###### Expose the Reaction API

1. Within the `reaction` directory that is within the `proxy` directory copy the `docker-compose.override.yml` to the reaction directory within the `reaction-platform` directory.

```
cp /home/your_username/proxy/reaction/docker-compose.override.yml /home/your_username/reaction-platform/reaction
```

2. Change the working directory to the reaction directory within the Reaction Platform.

3. Edit the `docker-compose.override.yml` file by substituting `YOUR_DOMAIN.COM` under the labels section with your actual domain.

4. Restart the Reaction service for the updates to take effect.

```
docker-compose down && docker-compose up -d && docker-compose logs -f
```

5. In the Traefik admin there should be a new frontend for `reaction.YOUDOMAIN.COM`

###### Expose the example storefront, and hydra services

Repeat steps 1-5 for the `example-storefront` and `reaction-hydra` directories.

In the Traefik admin UI verify a frontend exists for the `example-storefront` and `hydra`
