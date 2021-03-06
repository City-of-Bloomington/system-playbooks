# Ansible Playbooks

Start keeping track of the way systems are set up and configured using ansible roles and playbooks. These playbooks codify and automate the processes we use at the City of Bloomington to configure machines.

Ansible is a configuration management system. For more information about ansible, see:

http://www.ansible.com/how-ansible-works

It'll help to have a copy of this repository, so go ahead and grab it:

    git clone https://github.com/City-of-Bloomington/system-playbooks.git

## Terminology

The primary machine used to issue those commands is called the control machine. Hosts are the remote machines that you configure with ansible. For more details about terms, see here:

http://docs.ansible.com/ansible/glossary.html

To use ansible, you'll need a *nix based control machine/server that is used to configure and deploy any number of hosts and services. 

## Configure host machines

### Base OS installation on hosts

Ansible assumes you have already set up the new host/destination machine with a base OS installation. There are many options for where to put a base OS:

  - Install a base OS on an actual hardware instance of a machine.
  - Install a base OS on a virtual machine. VirtualBox or VMWare are common options.
      - VirtualBox is a free and capable solution for creating virtual machines on your local machine.
      - VMWare is a commercial solution that also makes the base OS install very straightforward.
  - Use Vagrant to spin up a base virtual machine. Vagrant does have the abilitiy to call a configuration management solution like ansible automatically. VirtualBox and VMWare are both options here too. If you're using VMWare for virtualization, you'll need the commercial version of Vagrant to work with VMWare.
  - Use a cloud service provider that provides the base system for you
  - Use Docker containers? (Still working on this...)

For VirtualBox, you'll need a "Host-only Adapter" in order to access the machine. When the VM is powered off, add another network interface:

    Virtual Box -> Devices -> Network -> Network Settings...
    Adapter 2 -> Enable Network Adapter
    Attached to: Host-only Adapter
    Name: vboxnet0

Then, when the machine is powered on, do:

    #find all adapters... should be a new one that is unconfigured
    ip addr
    #enp0s8 for me
    sudo vi /etc/network/interfaces
    #add new section for interface, adapting from lines already there

    # The local network interface
    auto enp0s8
    iface enp0s8 inet dhcp

    sudo /etc/init.d/networking restart
    #check for new ip with:
    ifconfig

Typically, openssh-server is available on server installations by default. Make sure that the host machine is accessible via SSH:

    sudo apt-get update
    sudo apt-get -y install openssh-server

Ansible requires Python installed on the host machine, as well:

    bash
    sudo apt-get install -y python

You'll also need a user account that can sudo.


## Install ansible on control machine

Up-to-date details for installing an ansible control machine are available here:

http://docs.ansible.com/ansible/intro_installation.html

If ansible is not installed on the local machine, it is a good idea to assign a static IP address to the control machine. It's easier to access the machine that way. See above for notes about configuring the network interface. In this case, we'll replace:

    iface enp0s8 inet dhcp

with:

    iface enp0s8 inet static
    address 192.168.56.22
    netmask 255.255.255.0

If ansible is being installed on a VM, it may make sense to update the hostname:

    sudo vi /etc/hostname
    sudo vi /etc/hosts
    sudo service hostname restart

Be sure to install any missing requirements:

    sudo apt-get install build-essential libssl-dev libffi-dev python-dev python-pip sshpass
    pip install --upgrade pip

Finally, install ansible with pip:

    pip install ansible


## Configure SSH public key connections

SSH public key connections allow you to configure a control machine with SSH access to a host machine without needing to supply a password every time.

https://help.ubuntu.com/community/SSH/OpenSSH/Keys

Generate local keys on the control machine:

    ssh-keygen -t rsa

then transfer the control machine's public key to the host/destination for the user that can sudo. See the link above for manual transfer process, or use ssh-copy-id. On OSX, this works:

https://github.com/beautifulcode/ssh-copy-id-for-OSX

    ssh-copy-id username@hostname

You can test this with:

    ssh username@hostname

If you're not prompted for a password, it worked!


## Ansible Configuration

### Tell ansible about hosts

Configure the host machines that you want to manage with ansible. Find the IP by looking at ifconfig on VM itself, and add it to a hosts file. The default hosts file is "/etc/ansible/hosts", but you can create one anywhere and specify it in an environment variable:

    echo "192.168.24.151" > hosts.txt
    export ANSIBLE_INVENTORY=~/path/to/scripts/for/ansible/hosts.txt

You can also pass the hosts file in as a parameter ("-i") on the command line.


### Ansible configuration defaults

Ansible expects playbooks, roles, etc (e.g. this repository) to be in /etc/ansible by default.

It is possible to work in a different location. Let ansible know with a local ansible.cfg file:

    cd /path/to/scripts/for/ansible
    vi ansible.cfg

For a starting point:

    curl -O https://raw.githubusercontent.com/ansible/ansible/devel/examples/ansible.cfg

Let ansible know where to find the roles:

    roles_path    = /etc/ansible/roles:/path/to/scripts/for/ansible

> Note: A sample ansible.cfg file is now included with this repository

### Test ansible:

At this point, if you run:

    ansible all -m ping -i hosts.txt

it should be "success"!

If you get an error, Ubuntu 16.04 server is no longer shipping with Python 2. You can make sure it is available with:

    ansible-playbook playbooks/bootstrap.yml -i hosts.txt --ask-become-pass

If you're using a VM, this is a good chance to take a snapshot of your host so it's easy to revert back to this point for testing.

## Using ansible

### Applying system configurations

Ansible uses the concept of "roles" to group configurations for a specific purpose. The roles can be combined to form a playbook to configure a certain type of system.

Pick a playbook, review the configured roles, then give it a go:

    ansible-playbook playbooks/linux.yml -i hosts.txt --ask-become-pass

If the playbook completes successfully, *congratulations!* You should have a working server configured for the corresponding service.


For playbooks that utilize passwords in the vault, pass in the '--ask-vault-pass' parameter:

    ansible-playbook playbooks/linux.yml -i hosts.txt --ask-become-pass --ask-vault-pass

For details about the vault:

[Variables and Vaults](group_vars/)

As processes are refined, or as operating systems change, it is important to test and adapt the corresponding playbooks and roles to reflect current requirements.

> *Note*:
> 
> "--ask-become-pass" and "--ask-vault-pass" are also set to True in ansible.cfg, so they won't need to be passed in explicitly if that is in place

## Roles

It can be helpful to abstract tasks that show up in more than one playbook / server as roles. 

To create new a new role:

    ansible-galaxy init cob.apache

When getting started with a new role, create it in the system-playbooks/roles directory. Once multiple playbooks make use of a role, abstract it out.

### Role Development

For development, roles need to be checked out from Github directly:

    git clone https://github.com/City-of-Bloomington/ansible-role-linux.git ./roles/City-of-Bloomington.linux
    git clone https://github.com/City-of-Bloomington/ansible-role-apache.git ./roles/City-of-Bloomington.apache
    git clone https://github.com/City-of-Bloomington/ansible-role-mysql.git ./roles/City-of-Bloomington.mysql
    git clone https://github.com/City-of-Bloomington/ansible-role-postgresql.git ./roles/City-of-Bloomington.postgresql
    git clone https://github.com/City-of-Bloomington/ansible-role-wsgi.git ./roles/City-of-Bloomington.wsgi
    git clone https://github.com/City-of-Bloomington/ansible-role-php.git ./roles/City-of-Bloomington.php
    git clone https://github.com/City-of-Bloomington/ansible-role-solr.git ./roles/City-of-Bloomington.solr
    git clone https://github.com/city-of-bloomington/ansible-role-fn1.git ./roles/City-of-Bloomington.fn1
    git clone https://github.com/city-of-bloomington/ansible-role-nodejs.git ./roles/City-of-Bloomington.nodejs
    git clone https://github.com/city-of-bloomington/ansible-role-tomcat.git ./roles/City-of-Bloomington.tomcat
    git clone https://github.com/city-of-bloomington/ansible-role-ckan.git ./roles/City-of-Bloomington.ckan
    git clone https://github.com/City-of-Bloomington/ansible-role-drupal.git ./roles/City-of-Bloomington.drupal
    
### External Roles

Some playbooks utilize external roles. These require using ansible-galaxy to pull them down and make them available locally.

To grab them all, in the main directory of this system-playbooks project, run the following:

    ansible-galaxy install --roles-path ./roles -r roles.yml

These roles are then available for use by playbooks.

Individual roles can be installed manually and included in playbooks. As an example, configuring MySQL:

https://github.com/geerlingguy/ansible-role-mysql  

This is included in the playbook with:

    - geerlingguy.mysql        

This requires using ansible-galaxy to pull down the role first:

    sudo ansible-galaxy install geerlingguy.mysql

This results in:

    "extracting geerlingguy.mysql to /etc/ansible/roles/geerlingguy.mysql" 


TODO:

Instructions for adding github oauth token:

https://github.com/settings/tokens



## References

Previously, we maintained instructions for manually configuring a linux system available here:

http://city-of-bloomington.github.io/LinuxInstallHelp/Ubuntu-Installation.html

However, these may grow out of date. This repository is a replacement for those.


This was a helpful guide for getting ansible configured initially:

https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-an-ubuntu-12-04-vps

Thanks!
