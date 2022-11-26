# Ansible openHAB Server
Complete Ansible project to set up an openHAB server on a Raspberry Pi

**Work In Progress**
* Project is functional
* Documentation unfinished


## Content
* [What does it do?](#what-does-it-do)
* [Preparation](#preparation)
    * [Install Ansible](#install-ansible)
    * [Prepare your Raspberry Pi](#prepare-your-raspberry-pi)
    * [Prepare Work Environment](#prepare-work-environment)
* [group_vars](#group-vars)
    * [all](#all)
    * [raspbian](#raspbian)
    * [openhab-server](#openhab)
* [TODO](#todo)
* [Sources](#sources)



## What does it do?
The aim is to take a fresh Raspbian installation, do some hardening, install git & Docker and have a ready to use private openHAB server. The steps break down to:
* Bootstrap the Pi ([Role prepare-raspberry](https://github.com/fex01/ansible-prepare-raspberry/blob/main/README.md))
    * rename user
        * rename the default user *pi* to *user_name*
        * set a new user password *user_password*
        * rename the home dir */home/pi* to */home/{{ user_name }}*
        * create a symlink */home/pi* linking to */home/{{ user_name }}*
    * rename host
        * change the hostname to *host_name*
        * update `/etc/hosts`
    * configure locale ([Role arillso.localization](https://github.com/arillso/ansible.localization/blob/master/README.md))
        * set timezone to *localization_timezone*
    * activate auto-updates / -upgrades ([Role weareinteractive.apt](https://github.com/weareinteractive/ansible-apt/blob/master/README.md))
    * SSH config
        * set sensible defaults for the SSH server
        * write public SSH keys to *user_name*'s *authorized_keys* file
        * disable SSH *root* login
        * enable SSH strict mode
        * disable X11 forwarding
        * disable SSH password login (role will fail if not at least one key is provided in *ssh_public_keys*)
        * allow SSH access only to *user_name*
        * add an SSH banner
        * option to set additional SSH settings
    * install and configure a firewall (ufw) ([Role weareinteractive.ufw](https://github.com/weareinteractive/ansible-ufw/blob/master/README.md))
* install & configure git with the option to checkout an existing openHAB config repo ([Role weareinteractive.git](https://github.com/weareinteractive/ansible-git/blob/master/README.md))
* install Docker ([Role geerlingguy.docker-arm](https://github.com/geerlingguy/ansible-role-docker_arm/blob/master/README.md))
* create an OpenHAB Docker instance ([Role openhab](https://github.com/fex01/ansible-openhab#readme))
    * create an openHAB service account
    * add *user_name* to the openHAB service group
    * if exists: checkout the git repository with your openHAB configuration
    * create missing folders & set permissions
    * pull the latest openHAB image and recreate the instance if it changed
* create and configure a frontail Docker instance to provide a Web logger experience similar to openHABian
* minimize writes to the SD Card (Copied from [rkoshak](https://github.com/rkoshak)s post [Ansible Revisited](https://community.openhab.org/t/ansible-revisited/105754) in the [openHAB community](https://community.openhab.org))
* install Samba and create a SMB share for the openHAB config ([Role bertvv.samba](https://github.com/bertvv/ansible-role-samba#readme))
* set the SSH entry point for *user_name* to *openhab_home*



## Preparation
If not stated otherwise all steps have to be done on your Ansible machine, not on the target.



### Install Ansible
If you are familiar with Ansible than just skip this subchapter.
Otherwise: To run Ansible Plays you need a machine running Ansible. That might be a Mac, a Linux machine or even via the Windows Subsystem for Linux. It does not have to be a physical machine, a VM is fine enough. You might want to have a look into the [Ansible docs][ansible-intro], for now it's enough if you install Ansible.
(The following steps are for Ubuntu, for details / other OSs have a look into the [Ansible docs][ansible-install].
* `sudo apt update`
* `sudo apt install software-properties-common`
* `sudo apt-add-repository --yes --update ppa:ansible/ansible`
* `sudo apt install ansible`
* `sudo apt install python3-argcomplete`
* `sudo activate-global-python-argcomplete3`



### Prepare your Raspberry Pi
* find the URL for the newest Raspberry Pi OS Lite version at https://www.raspberrypi.com/software/operating-systems/
* write image to SD card https://www.raspberrypi.org/documentation/installation/installing-images/README.md
    * enable SSH: `touch /Volumes/boot/ssh`<sup id="a5">[5](#f5)</sup> 
    * [optional] enable wifi: `vim /Volumes/boot/wpa_supplicant.conf`
    ```yml
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    network={
        ssid="YOUR_WIFI_SSID"
        psk="YOUR_WIFI_PASSWORD"
        key_mgmt=WPA-PSK
    }
    ```
    * eject & insert SD card into your Raspi
    * power up your Raspberry Pi
* if missing, generate ssh public key on your Ansible machine: `ssh-keygen -o`
* if you are repeating the setup process delete old host keys
    * delete old host key (name): `ssh-keygen -f "/home/<username>/.ssh/known_hosts" -R "raspberrypi"`
    * delete old host key (IP): `ssh-keygen -f "/home/<username>/.ssh/known_hosts" -R "xxx.xxx.xxx.xxx"`
* copy ssh public key to your Pi (password raspberry): `ssh-copy-id pi@raspberrypi`



### Prepare Work Environment 
* clone this repo into a location of your choice `git clone https://github.com/fex01/ansible-openhabserver.git`
* checkout all git submodules `git submodule update --init --recursive`



## group_vars
While Ansibles knows a number of options where to set variables (details see [Ansible - Using Variables][ansible-variables]), the basic option is to link them to machines in your Inventory (file [hosts](https://github.com/fex01/ansible-openhabserver/blob/main/hosts)). 
Our target is named 'openhab' and a member of the groups 'openhab-server', 'raspbian' and the special group 'all', which means the variables for all these groups (called group_vars) and the variables for our target (called host_vars) apply.


### all

```yml
---
# group_vars file for all

# default system user
ansible_user: "{{ user_name }}"

############################################################
########################Bootstrapping#######################
############################################################

# SSH
# List your SSH keys here, one per line. Be sure to remove the '[]',
# if you add dependencies to this list.
# ssh_public_keys:
#   - ssh-rsa xxxxxxxxxxxxxxxxxxxxxxxxx link@my-machine
# Note: The first of these keys needs to be one that Ansible is using. You get your public SSH key
# by running `cat ~/.ssh/id_rsa.pub`.
ssh_public_keys: []
# String to present when connecting to host over ssh
ssh_banner: "Welcome to {{ user_name }}'s {{ host_name }} server\n"
```


### raspbian

```yml
---
# group_vars file for raspbian

############################################################
########################Bootstrapping#######################
############################################################

# The system timezone
localization_timezone: "Europe/Berlin"

# upgrade system: safe | full | dist | no
apt_upgrade: full
# Do “apt-get upgrade –download-only” every n-days (0=disable)
apt_download_upgradeable_packages: 1
# Do “apt-get autoclean” every n-days (0=disable)
apt_auto_clean_interval: 1
# Split the upgrade into the smallest possible chunks so that
# they can be interrupted with SIGUSR1. This makes the upgrade
# a bit slower but it has the benefit that shutdown while a upgrade
# is running is possible (with a small delay)
apt_unattended_upgrades_minimal_steps: yes
# Send email to this address for problems or packages upgrades
# If empty or unset then no email is sent, make sure that you
# have a working mail setup on your system. A package that provides
# 'mailx' must be installed. E.g. "user@example.com"
apt_mails: []
# Automatically reboot *WITHOUT CONFIRMATION*
# if the file /var/run/reboot-required is found after the upgrade
apt_unattended_upgrades_automatic_reboot: yes
# If automatic reboot is enabled and needed, reboot at the specific
# time instead of immediately
# Values: now | 02:00 | ...
apt_unattended_upgrades_automatic_reboot_time: 04:00

# SSH
# Basic sshd config will be done, use this to set additional values
ssh_sshd_config_settings:
  - var_name: "MaxAuthTries"
    value: "3"
  - var_name: "PubkeyAuthentication"
    value: "yes"
  - var_name: "AuthorizedKeysFile"
    value: ".ssh/authorized_keys"
  - var_name: "IgnoreUserKnownHosts"
    value: "yes"
  - var_name: "IgnoreRhosts"
    value: "yes"
  - var_name: "AllowAgentForwarding"
    value: "no"
  - var_name: "AllowTcpForwarding"
    value: "no"
  - var_name: "GatewayPorts"
    value: "no"
  - var_name: "PermitTTY"
    value: "yes"
  - var_name: "Protocol"
    value: "2"

# UFW rules should always allow SSH to keep Ansible functioning
ufw_rules:
  - { rule: "allow", port: "{{ ssh_port }}", proto: "tcp", comment: 'SSH' }
```


### openhab-server

```yml
---
# group_vars file for openhab

ansible_host: "xxx.xxx.xxx.xxx"
ansible_port: "{{ ssh_port }}"

############################################################
########################Bootstrapping#######################
############################################################

# main part of the config is in group_vars/raspbian

# system user name
user_name: "link"
# password for system user
# create: 
#   * create hash: ansible all -i localhost, -m debug -a "msg={{ 'my_secret_password' | password_hash('sha512') }}"
#   * encrypt hash: ansible-vault encrypt_string --vault-id openhab@prompt --stdin-name 'user_password_hash'
# use in play: ansible-playbook <playbook>.yml --vault-id openhab@prompt
# default: my_secret_password
user_password_hash:
# SSH
# grant SSH access only for
ssh_users: 
  - "{{ user_name }}"
# Sets the ssh port
ssh_port: 22

# private key to clone from gitserver will be deploy by w.users to /home/{{ user_name }}/.ssh/id_rsa
#   * create private / public key pair: `ssh-keygen -o -C "link@openhab"`
#   * copy public key to git server /home/git/.ssh/authorized_keys
#   * copy the private key file to ansible machine `scp -C -P 22 link@openhab:/home/link/.ssh/id_rsa ./id_rsa.original`
#   * encrypt content with `cat id_rsa.original | ansible-vault encrypt_string --vault-id openhab@prompt --stdin-name 'ssh_private_key'`
ssh_private_key:


############################################################
############################ufw#############################
############################################################
# additional firewall rules - careful, overwrites existing rules
openhab_ufw_rules:
  - { rule: "allow", port: "{{ ssh_port }}", proto: "tcp", comment: 'SSH' }
  - { rule: "allow", port: "8080", proto: "tcp", comment: 'openHAB HTTP' }
  - { rule: "allow", port: "8443", proto: "tcp", comment: 'openHAB HTTPS' }
  - { rule: "allow", port: "5007", proto: "tcp", comment: 'openHAB LSP' }
  - { rule: "allow", port: "9125", proto: "tcp", comment: 'openHAB Homematic XML-RPC' }
  - { rule: "allow", port: "9126", proto: "tcp", comment: 'openHAB Homematic BIN-RPC' }
  - { rule: "allow", port: "9001", proto: "tcp", comment: 'frontail' }
  - { rule: "allow", port: "9124", proto: "tcp", comment: 'homekit' }
  - { rule: "allow", port: "445", proto: "tcp", comment: 'SMB' }
  - { rule: "allow", port: "445", proto: "udp", comment: 'SMB' }

############################################################
############################git#############################
############################################################

# username for git commits
git_commit_user_name: link
# user email for git commits
git_commit_user_email: link@example.org


############################################################
###########################docker###########################
############################################################
apt_packages:
  - python3-setuptools
  - python3-dev
pip_package: python3-pip
docker_pip_executable: pip3
docker_users:
  - "{{ user_name }}"


############################################################
##########################openhab###########################
############################################################
openhab_version: latest
openhab_hostname: openhab
openhab_home: /opt/docker
openhab_repo:
  - accept_hostkey: yes
    dest: "{{ openhab_home }}/openhab"
    key_file: "/home/{{ user_name }}/.ssh/id_rsa"
    repo:


############################################################
#########################frontail###########################
############################################################
frontail_version: latest
frontail_hostname: frontail 
frontail_openhab_home: "{{ openhab_home }}/openhab"


############################################################
###########################samba############################
############################################################
# max. SMB2 for macOS compatibility
samba_server_min_protocol: SMB2
samba_apple_extensions: yes
# no for for macOS compatibility
samba_mitigate_cve_2017_7494: no
samba_server_string: "Welcome to the {{ host_name }} file share"
samba_users:
  - name: openhab
    password: openhab
samba_shares:
  - name: openhab
    comment: "file access to docker root directory"
    valid_users: openhab
    write_list: openhab
    group: openhab
    browseable: 'yes'
    path: "{{ openhab_home }}"
    owner: openhab
    group: openhab
``` 
