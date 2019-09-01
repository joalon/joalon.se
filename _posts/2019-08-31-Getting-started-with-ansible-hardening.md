---
layout: post
title: "Getting started with ansible-hardening"
author: "Joakim Lönnegren"
categories: blog
tags: [Ansible, Ansible-Hardening, AWX, Security, Automation, 2019]
image:
  feature: ansible-hardening/feature.png
  teaser: ansible-hardening/teaser.png
---
The Openstack project has developed a hardening role for applying best practices according to the DISA, the Defense Information System Agency which is part of the U.S. Department of Defense. The particular implementation is based on their unclassified Security Technical Implementation Guide, STIG. In this post we'll apply the role in 'check mode' against localhost, a development laptop running Fedora 30.

The ansible-hardening source code can be found on [Github](https://github.com/openstack/ansible-hardening/). The project page can be found as a part of the [Openstack documentation](https://docs.openstack.org/ansible-hardening/latest/).

## Getting started
Ansible works with some abstractions. The main one being a 'playbook' which defines a set of plays to be carried out on the 'inventory', a list of computers. An ansible 'role' is itself a collection of plays which is parameterized and can be applied to groups or individual servers in an inventory.

## The Role
Here's an overview of a temporary directory `tmp-dir` where I'll run the hardening playbook.

```bash
tmp-dir/
  hardening.yml
  roles/
    ansible-hardening/
```

Under tmp-dir run the following command to clone the hardening role into the `roles` directory, which is a default ansible directory where ansible roles can be stored so that the runtime can find them.

```bash
git clone https://opendev.org/openstack/ansible-hardening roles/ansible-hardening
```

The ansible-hardening role is written to use both tags and variables to customize the functionality. For example, the security_ntp_servers variable takes a list of ntp servers and applies them to your system. Tags can then be used to customize which tasks to be run, for example running the example with `--skip-tags ntpd`, which would skip all controls relating to the ntp daemon. The full updated list of controls can be found in the documentation [](). 

## The Playbook
When applying an ansible role to a server you'll need a playbook. Here follows a basic `hardening.yml` playbook which will apply the ansible-hardening role to all servers in the inventory.

```yaml
---
- name: Harden all systems
  hosts: all
  become: yes

  vars: 
    ansible_python_interpreter: /usr/bin/python3

  roles:
    - ansible-hardening
```

Ansible is still migrating from Python 2 to Python 3. When running this playbook without specifying the ansible_python_interpreter I got the error "No match for argument: python2-dnf". Since I couldn't install that package I went digging and found the following issue [54855](https://github.com/ansible/ansible/issues/54855). The fix mentioned is to specify the python3 interpreter for the time being.

When running a playbook the default command is `ansible-playbook` which is often run with `ansible-playbook (flags) <playbook>.yml`. For running this playbook I used the following command:

```bash
ansible-playbook -CkKi  'localhost,'  hardening.yml
```

Some different flags used. Here's a short recap:

```bash
-C     Check mode, used to run a playbook and report which changes would be made without actually doing any of them. 
-k     SSH connection password. Leave this out if you have copied your keys to the remote server.
-K      Your sudo/admin password. Defaults to the same as the SSH connection password.
-i      The inventory to use. Note that the inventory string has a comma on the end to specify that it is a list only containing localhost. You could also use an inventory file or leave it out to use the default /etc/ansible/hosts
```

Running this took around 10 minutes and the results look something like this:

![Results](/images/ansible-hardening/ansible-playbook-result.png)

Any green 'ok' is acceptably hardened. The yellow 'changed' are items you'll want to take a look at.

## Bonus: AWX
I've also setup [AWX](https://github.com/ansible/awx), the upstream version of Ansible Tower which is a server with a web GUI for running ansible playbooks in an enterprise environment. the goal here is to get AWX to run ansible-hardening in check-mode against the same laptop and display the result in the browser.

If you have Docker Compose with the python module [docker-compose](https://pypi.org/project/docker-compose/) installed here's a quick setup to get awx running on your host:

```bash
git clone https://github.com/ansible/awx
cd awx/installer
ansible-playbook -i inventory install.yml
```

Of course, check the [official installation instructions](https://github.com/ansible/awx/blob/devel/INSTALL.md) for your setup. It took a few times to get this running for me, had one non-intuitive error message where port 80 was busy, I changed the host_port variable to 8080. I had to undo the installation some times to get it working. I used this shell script:

```fish
#!/bin/fish

# Remove the containers
for i in awx_task awx_web awx_postgres awx_rabbitmq awx_memcached
     docker rm --force $i
end

# Remove the temporary awx folder
rm -rf /tmp/awxcompose/
```

At this point we need some awx boilerplate to get started: 
* An AWX project
* An inventory
* A machine credential
* A job template

First let's create an awx project from a [github repository](https://github.com/joalon/getting-started-ansible-hardening).

![AWX Project](/images/ansible-hardening/create-awx-project.png)

Next let's edit the Demo inventory and add the new host.

![AWX Inventory](/images/ansible-hardening/awx-add-host-to-inventory.png)

And add a credential that can be used to log in.

![AWX Machine Credential](/images/ansible-hardening/create-awx-credential.png)

Finally let's create a job template.

![AWX Template](/images/ansible-hardening/create-awx-template.png)

Don't forget to set the 'job type' to 'Check'.

![AWX Job Type](/images/ansible-hardening/awx-job-type.png)

After setting all this up you can launch a job in the job-template screen. The following run took 9 minutes:

![AWX Results](/images/ansible-hardening/awx-successful.png)

Thanks for reading! 
