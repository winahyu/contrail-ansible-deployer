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
yum install kernel-3.10.0-957.12.2.el7.x86_64
yum install kernel-devel-3.10.0-957.12.2.el7.x86_64
yum install kernel-headers-3.10.0-957.12.2.el7.x86_64
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

### Troubleshooting.

first to check, make sure all containers are up.

```
docker ps

CONTAINER ID        IMAGE                                                                       COMMAND                  CREATED             STATUS              PORTS               NAMES
36fa1236712d        opencontrailnightly/contrail-vrouter-agent:latest                           "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             vrouter_vrouter-agent_1
3a5cf5db888b        opencontrailnightly/contrail-nodemgr:latest                                 "/entrypoint.sh /b..."   4 weeks ago         Up 23 hours                             vrouter_nodemgr_1
7272722f189f        opencontrailnightly/contrail-analytics-alarm-gen:latest                     "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             analytics_alarm-gen_1
e89f38d721de        opencontrailnightly/contrail-analytics-api:latest                           "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             analytics_api_1
97b6b640210d        opencontrailnightly/contrail-analytics-collector:latest                     "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             analytics_collector_1
e104fd1c84fb        opencontrailnightly/contrail-nodemgr:latest                                 "/entrypoint.sh /b..."   4 weeks ago         Up 23 hours                             analytics_nodemgr_1
9e77748c6711        opencontrailnightly/contrail-analytics-query-engine:latest                  "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             analytics_query-engine_1
7137cfa35526        opencontrailnightly/contrail-nodemgr:latest                                 "/entrypoint.sh /b..."   4 weeks ago         Up 23 hours                             analyticsdatabase_nodemgr_1
6419caeb94b2        opencontrailnightly/contrail-external-zookeeper:latest                      "/contrail-entrypo..."   4 weeks ago         Up 23 hours                             analyticsdatabase_zookeeper_1
0b5a5db43d20        opencontrailnightly/contrail-external-cassandra:latest                      "/contrail-entrypo..."   4 weeks ago         Up 23 hours                             analyticsdatabase_cassandra_1
938c0a50b830        opencontrailnightly/contrail-external-kafka:latest                          "/docker-entrypoin..."   4 weeks ago         Up 23 hours                             analyticsdatabase_kafka_1
19fd7d00797e        opencontrailnightly/contrail-controller-control-named:latest                "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             control_named_1
a46dd131bd0e        opencontrailnightly/contrail-nodemgr:latest                                 "/entrypoint.sh /b..."   4 weeks ago         Up 23 hours                             control_nodemgr_1
c4af8bbf68d1        opencontrailnightly/contrail-controller-control-control:latest              "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             control_control_1
42b38adf8aff        opencontrailnightly/contrail-controller-control-dns:latest                  "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             control_dns_1
8a5f3fc02e45        opencontrailnightly/contrail-controller-webui-web:latest                    "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             webui_web_1
76129276575a        opencontrailnightly/contrail-controller-webui-job:latest                    "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             webui_job_1
c4e972f2a290        opencontrailnightly/contrail-controller-config-svcmonitor:latest            "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             config_svcmonitor_1
b26e550e0753        opencontrailnightly/contrail-nodemgr:latest                                 "/entrypoint.sh /b..."   4 weeks ago         Up 23 hours                             config_nodemgr_1
1b8f5ac84573        opencontrailnightly/contrail-controller-config-schema:latest                "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             config_schema_1
8e067db2d3f5        opencontrailnightly/contrail-controller-config-api:latest                   "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             config_api_1
6414303579ce        opencontrailnightly/contrail-controller-config-devicemgr:latest             "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             config_devicemgr_1
8825b437031a        opencontrailnightly/contrail-external-zookeeper:latest                      "/contrail-entrypo..."   4 weeks ago         Up 23 hours                             configdatabase_zookeeper_1
1d6a8d702d60        opencontrailnightly/contrail-external-cassandra:latest                      "/contrail-entrypo..."   4 weeks ago         Up 23 hours                             configdatabase_cassandra_1
8e6290172cc1        opencontrailnightly/contrail-external-rabbitmq:latest                       "/contrail-entrypo..."   4 weeks ago         Up 23 hours                             configdatabase_rabbitmq_1
42e85e83c57a        redis:4.0.2                                                                 "docker-entrypoint..."   4 weeks ago         Up 23 hours                             redis_redis_1
0fc50753f919        kolla/centos-binary-barbican-worker:ocata                                   "kolla_start"            4 weeks ago         Up 23 hours                             barbican_worker
c6c4da0417b4        kolla/centos-binary-barbican-keystone-listener:ocata                        "kolla_start"            4 weeks ago         Up 23 hours                             barbican_keystone_listener
3c9007fe76fe        kolla/centos-binary-barbican-api:ocata                                      "kolla_start"            4 weeks ago         Up 23 hours                             barbican_api
2f154b9bbfcc        kolla/centos-binary-horizon:ocata                                           "kolla_start"            4 weeks ago         Up 23 hours                             horizon
cc89d904399c        kolla/centos-binary-heat-engine:ocata                                       "kolla_start"            4 weeks ago         Up 23 hours                             heat_engine
80f142d0f775        kolla/centos-binary-heat-api-cfn:ocata                                      "kolla_start"            4 weeks ago         Up 23 hours                             heat_api_cfn
aced6a0e3424        kolla/centos-binary-heat-api:ocata                                          "kolla_start"            4 weeks ago         Up 23 hours                             heat_api
479207e79a71        kolla/centos-binary-neutron-metadata-agent:ocata                            "kolla_start"            4 weeks ago         Up 23 hours                             neutron_metadata_agent
f3f4301708ef        kolla/centos-binary-neutron-server:ocata                                    "kolla_start"            4 weeks ago         Up 23 hours                             neutron_server
f78372a0acfa        kolla/centos-binary-nova-compute-ironic:ocata                               "kolla_start"            4 weeks ago         Up 23 hours                             nova_compute_ironic
a305757ff389        kolla/centos-binary-nova-compute:ocata                                      "kolla_start"            4 weeks ago         Up 23 hours                             nova_compute
f70bf7f7abdc        kolla/centos-binary-nova-novncproxy:ocata                                   "kolla_start"            4 weeks ago         Up 23 hours                             nova_novncproxy
82711c770d35        kolla/centos-binary-nova-consoleauth:ocata                                  "kolla_start"            4 weeks ago         Up 23 hours                             nova_consoleauth
21581f99e321        kolla/centos-binary-nova-conductor:ocata                                    "kolla_start"            4 weeks ago         Up 23 hours                             nova_conductor
93802a3c3872        kolla/centos-binary-nova-scheduler:ocata                                    "kolla_start"            4 weeks ago         Up 23 hours                             nova_scheduler
003ece6ab316        kolla/centos-binary-nova-api:ocata                                          "kolla_start"            4 weeks ago         Up 23 hours                             nova_api
0f32e18dd05f        kolla/centos-binary-nova-placement-api:ocata                                "kolla_start"            4 weeks ago         Up 23 hours                             placement_api
b6e184ebec37        kolla/centos-binary-nova-libvirt:ocata                                      "kolla_start"            4 weeks ago         Up 23 hours                             nova_libvirt
144e0a7d42e3        kolla/centos-binary-nova-ssh:ocata                                          "kolla_start"            4 weeks ago         Up 23 hours                             nova_ssh
792b784df11f        opencontrailnightly/contrail-openstack-ironic-notification-manager:latest   "/entrypoint.sh /u..."   4 weeks ago         Up 23 hours                             ironic_notification_manager
e875eb88ba7e        kolla/centos-binary-ironic-conductor:ocata                                  "kolla_start"            4 weeks ago         Up 23 hours                             ironic_conductor
ff9207ba76ba        kolla/centos-binary-ironic-api:ocata                                        "kolla_start"            4 weeks ago         Up 23 hours                             ironic_api
1ba0d5cb774e        kolla/centos-binary-ironic-pxe:ocata                                        "kolla_start"            4 weeks ago         Up 23 hours                             ironic_pxe
4f67c8010e41        kolla/centos-binary-glance-registry:ocata                                   "kolla_start"            4 weeks ago         Up 23 hours                             glance_registry
a5506a37b353        kolla/centos-binary-glance-api:ocata                                        "kolla_start"            4 weeks ago         Up 23 hours                             glance_api
fadb53029f6d        kolla/centos-binary-keystone:ocata                                          "kolla_start"            4 weeks ago         Up 23 hours                             keystone
27ebfd772472        kolla/centos-binary-rabbitmq:ocata                                          "kolla_start"            4 weeks ago         Up 23 hours                             rabbitmq
3969adda3237        kolla/centos-binary-iscsid:ocata                                            "kolla_start"            4 weeks ago         Up 23 hours                             iscsid
95ddf3daa223        kolla/centos-binary-mariadb:ocata                                           "kolla_start"            4 weeks ago         Up 23 hours                             mariadb
3ff6c7db0672        kolla/centos-binary-cron:ocata                                              "kolla_start"            4 weeks ago         Up 23 hours                             cron
e94b2ef99cf1        kolla/centos-binary-kolla-toolbox:ocata                                     "kolla_start"            4 weeks ago         Up 23 hours                             kolla_toolbox
8fc44b3744bb        kolla/centos-binary-fluentd:ocata                                           "kolla_start"            4 weeks ago         Up 23 hours                             fluentd
35abdaf4fc82        kolla/centos-binary-memcached:ocata                                         "kolla_start"            4 weeks ago         Up 23 hours                             memcached

```
