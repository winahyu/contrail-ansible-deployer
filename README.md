# Contrail 5.0 AIO on Centos 7.4 Installation

For a quick setup, here is the physical configuration for the All-In-One Contrail v5.0 machine or VM with preferred resources:

One machine for AIO Contrail:

CPU: 8
RAM: 32GB
HDD: 120GB
NIC: 1
One machine for the deployer/installer can use any machine/VM with Centos 7.4 OS.

OS for All-in-One Installation
On both machines install Centos OS v7.4  from http://mirrors.mit.edu/centos/7/isos/x86_64/ Download the file .CentOS-7-x86_64-minimal-1708.iso. and install.

Preparation of the Base Host Environment

### Step 1. Contrail AIO Machine

Upgrade kernel on Contrail AIO Machine

Because of Docker overlay filesystem requirements, Centos OS v7.4 host needs to be upgraded to the kernel at least to 3.10.0-862 version, below is the steps to upgrade the kernel.

```
yum install kernel-3.10.0-862.3.2.el7.x86_64
yum install kernel-devel-3.10.0-862.3.2.el7.x86_64
yum install kernel-headers-3.10.0-862.3.2.el7.x86_64
grub2-set-default 0
```

Disable the NetworkManager; Use a static IP setting

Next before rebooting the machine to apply the new kernel, make sure the host network configuration uses static configuration and disable NetworkManager  service for the interface. Below are the steps to run for this part of the setup.

```
service NetworkManager stop
systemctl disable NetworkManager
vi /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
IPADDR=10.10.20.x
NETMASK=255.255.255.0
GATEWAY=10.10.20.y
DNS1=8.8.8.8

reboot
```

### Step 2. Deployer / Installer Machine

After configuring Contrail AIO machine, now is the time to work inside Installer machine. First make sure the Contrail AIO machine is reachable from the Installer host. A few additional steps are needed before installation can start. First, the machine needs to have Ansible and Git. Use Git to clone contrail-ansible-deployer from the Juniper GitHub website.

```
cd ~/
yum -y install epel-release 
yum -y install git ansible-2.4.2.0
git clone http://github.com/Juniper/contrail-ansible-deployer
cd contrail-ansible-deployer
```

This AIO install only needed to edit one file specific to the installation: ~/contrail-ansible-deployer/config/instances.yaml.

Below is the contents of the AIO .instances.yaml. file used for this deployment:

```
Provider_config:
  bms:
    ssh_pwd: somepassword # <<-- password of your AIO host
    ssh_user: root
    ntpserver: ntp.someserver.net
    domainsuffix: local
instances:
  bms1:
    provider: bms
    ip: 10.233.255.7    #  <<-- change this ip to your AIO host
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      vrouter:
      openstack:
      openstack_compute:
contrail_configuration:
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
kolla_config:
  kolla_globals:
    enable_swift: no
    enable_haproxy: no
  kolla_passwords:
    keystone_admin_password: Contrail123 # This will be the admin password
global_configuration:
  REGISTRY_PRIVATE_INSECURE: True
```

### Contrail Installation Process

From deployer/installer these two Ansible playbook commands need to be executed to deploy Contrail v5.0:

```
cd ~/contrail-ansible-deployer/

ansible-playbook -i inventory/ playbooks/configure_instances.yml
ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_contrail.yml
```

The return result from both commands will provide enough information if something fails,. For successful deployment, the expected result is shown below.

```
[root@ Installer contrail-ansible-deployer]# ansible-playbook -i inventory/ playbooks/configure_instances.yml

******************************************************************************
skipping: [10.233.255.7]

PLAY RECAP 
******************************************************************************
10.233.255.7    : ok=39   changed=28   unreachable=0    failed=0
localhost       : ok=10   changed=2    unreachable=0    failed=0

[root@ Installer contrail-ansible-deployer]# ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_contrail.yml

******************************************************************************

TASK [install_contrail : untaint node] 
******************************************************************************
skipping: [10.233.255.7] => (item={'value': {u'ip': u'10.233.255.7', u'roles': 
{u'control': None, u'openstack_compute': None, u'config_database': None, u'analytics': 
None, u'webui': None, u'vrouter': None, u'openstack': None, u'analytics_database': None, u'config': 
None}, u'provider': u'bms'}, 'key': u'bms1'})

PLAY RECAP 
******************************************************************************
10.233.255.7    : ok=495  changed=230  unreachable=0    failed=0
localhost       : ok=7    changed=2    unreachable=0    failed=0

[root@ Installer contrail-ansible-deployer]#
```

### Contrail Web UI.

Accessing the web UI after installation is completed.

OpenStack -> http://<IP-ADDR>/horizon
Contrail -> https://<IP-ADDR>:8143

Username: admin, Password: Contrail123, domain: default.


