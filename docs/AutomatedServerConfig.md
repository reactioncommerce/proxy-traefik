# Automated Server Configuration

In order to expedite the installation of server dependencies, Ansible will be used to automate most of the server configuration.


### Getting started

**Prepare control node(local computer)**

Ansible requireds a control node, which is a computer that manages a remote host. This guide will use a developers Mac laptop/desktop as the control node. 

Install ansible using [homebrew](https://brew.sh), this guide assumes some familiarity with Ansible, if you need an introduction to basic concepts click [here](https://www.ansibletutorials.com).

`brew install ansible`

Also install python3 to avoid deprecation warnings,

`brew install python3`



**Prepare remote host**

Create a new droplet in you DigitalOcean dashboard with at least 2GB of RAM. When creating the droplet, either select an existing SSH key to login or click on the "New SSH Key" under the auhtentication section  and copy your public SSH key from your local computer.

Copy the newly created IP addres and verify thet you can login into the new server by executing:

```
ssh root@XXX.XXX.XXX.XXX
```

**Manage the remote host with Ansible**

Now it's time to create an inventory file in which `python3` is specified as the interpreter. On your machine, create a new file at `/etc/ansible/hosts`, and add the content below.

```
[servers]
reaction.dev

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
[web]
```

Edit your hosts, file  
```
sudo vim /etc/hosts
```

and add an entry for your DigitalOcean droplet,

```
XXX.XXX.XXX.XXX reaction.dev
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




