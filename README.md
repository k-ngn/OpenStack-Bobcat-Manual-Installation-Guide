# OpenStack-Bobcat-Manual-Installation-Guide


> [!NOTE]
> **This guide provide step-by-step install OpenStack for minimum lab environment with muti-nodes** 

Openstack version: **OpenStack 2023.2 Bobcat**
OS: CentOS Stream 9

Network requirements:
2 interfaces: 
- 1 for management 
- 1 for provide internet access

**VMs configuration using in this guide:**
Virtualization software: VMware workstation
Controller node: 
- CPU: 2
- RAM: 4GB
- Network:
	1. `IP_EXTERNAL`: 192.168.43.103 (NAT network - to provide internet access)
	2. `IP_MGNT`: 10.0.0.51

Compute node: 
- CPU: 4
- RAM: 6GB
- Network:
	1. `IP_EXTERNAL`: 192.168.43.104 (NAT network - to provide internet access)
	2. `IP_MGNT` 10.0.0.52 

> [!IMPORTANT]
> - All commands running in this guide is run under **root** user
> - Use `IP_MGNT_CONTROLLER` and `IP_MGNT_COMPUTE` to ssh into server for prevent interruption when setup openstack network

## Prerequisite Installation

**Install and configure for both node**

- Run command `ip a` to get your interface name then replace to the script below before running: 

```bash
hostnamectl set-hostname controller
yum update -y
yum install epel-release -y
yum update -y 
yum install -y wget byobu git vim crudini nano

nmcli con modify EXTERNAL_INTERFACE_NAME ipv4.addresses IP_EXTERNAL/24
nmcli con modify EXTERNAL_INTERFACE_NAME ipv4.gateway IP_EXTERNAL_GATEWAY
nmcli con modify EXTERNAL_INTERFACE_NAME ipv4.dns IP_EXTERNAL_DNS
nmcli con modify EXTERNAL_INTERFACE_NAME ipv4.method manual
nmcli con modify EXTERNAL_INTERFACE_NAME connection.autoconnect yes

nmcli con modify MNT_INTERFACE_NAME ipv4.addresses IP_MGNT/24
nmcli con modify MNT_INTERFACE_NAME ipv4.method manual
nmcli con modify MNT_INTERFACE_NAME connection.autoconnect yes

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sudo systemctl disable firewalld
sudo systemctl stop firewalld
init 6
```

For example of network:
```bash
nmcli con modify ens160 ipv4.addresses 192.168.43.103/24
nmcli con modify ens160 ipv4.gateway 192.168.43.1
nmcli con modify ens160 ipv4.dns 192.168.43.1
nmcli con modify ens160 ipv4.method manual
nmcli con modify ens160 connection.autoconnect yes

nmcli con modify ens192 ipv4.addresses 10.0.0.51/24
nmcli con modify ens192 ipv4.method manual
nmcli con modify ens192 connection.autoconnect yes
```

- Open host file: `/etc/hosts` and add these lines below:

```bash
10.0.0.51 controller
10.0.0.52 compute
```

#### 1. Install NTP 

**Install and configure controller node**

- Install the packages:
```bash
yum install chrony -y
```

- Edit the `/etc/chrony.conf` file:
```bash
server NTP_SERVER iburst
```
For `NTP_SERVER`, you can get from this [link]([pool.ntp.org: NTP Servers in Vietnam, vn.pool.ntp.org](https://www.ntppool.org/zone/vn)), or you can leave it as default

> [!NOTE]
> By default, the controller node synchronizes the time via a pool of public servers. However, you can optionally configure alternative servers such as those provided by your organization. 

- To enable other nodes to connect to the chrony daemon on the controller node, add this key to the same `chrony.conf` file mentioned above:
```bash
allow <MANAGEMENT_CIDR_RANGE>
```

`MANAGEMENT_CIDR_RANGE` example: `10.0.0.0/24`, replace based on your setup

- Restart and enable the NTP service:
```bash
systemctl enable chronyd.service
systemctl start chronyd.service
```

**Install and configure controller node**

Install the packages:
```bash
yum install chrony -y
```

- Edit the `/etc/chrony.conf` file:
```bash
server controller iburst
```
- Comment out the `pool 2.debian.pool.ntp.org offline iburst` line.
- Restart and enable the NTP service.
```bash
systemctl enable chronyd.service
systemctl start chronyd.service
```

#### 2. Install OpenStack packages:

**Install and configure all nodes**

**CentOS 9**
```bash
dnf install -y dnf-plugins-core
dnf config-manager --set-enabled crb

dnf install -y centos-release-openstack-bobcat
dnf upgrade-y
dnf install -y python3-openstackclient
dnf install -y openstack-selinux
```

#### 3. Install SQL Database:

**Install and configure controller node**

```bash
yum install mariadb mariadb-server python3-PyMySQL -y
```

- Create and edit the `/etc/mysql/mariadb.conf.d/99-openstack.cnf` file and add these:
```bash
[mysqld]
bind-address = <IP_MGNT_CONTROLLER>

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

- Start and enable the database service:
```bash
systemctl enable mariadb.service
systemctl start mariadb.service
```

- Secure the database service by running the `mysql_secure_installation` script. In particular, choose a suitable password for the database `root` account:
```bash
mysql_secure_installation
```

- For `mysql_secure_installation` guide: 
```
Enter current password for root (enter for none): ENTER
Switch to unix_socket authentication [Y/n] n
Change the root password? [Y/n] y
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```
Replace `ROOT_PASSWORD` with proper password you want

#### 4. Install Message Queue:

**Install and configure controller node**

- Install RabbitMQ and Cloudsmith Signing Keys:
```bash
## primary RabbitMQ signing key
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc'
## modern Erlang repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key'
## RabbitMQ server repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key'
```

- Add repo:
```bash
nano /etc/yum.repos.d/rabbitmq.repo
```

- Paste in these contents and save the file:
```bash
# In /etc/yum.repos.d/rabbitmq.repo

##
## Zero dependency Erlang RPM
##

[modern-erlang]
name=modern-erlang-el9
# Use a set of mirrors maintained by the RabbitMQ core team.
# The mirrors have significantly higher bandwidth quotas.
baseurl=https://yum1.rabbitmq.com/erlang/el/9/$basearch
        https://yum2.rabbitmq.com/erlang/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-noarch]
name=modern-erlang-el9-noarch
# Use a set of mirrors maintained by the RabbitMQ core team.
# The mirrors have significantly higher bandwidth quotas.
baseurl=https://yum1.rabbitmq.com/erlang/el/9/noarch
        https://yum2.rabbitmq.com/erlang/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-source]
name=modern-erlang-el9-source
# Use a set of mirrors maintained by the RabbitMQ core team.
# The mirrors have significantly higher bandwidth quotas.
baseurl=https://yum1.rabbitmq.com/erlang/el/9/SRPMS
        https://yum2.rabbitmq.com/erlang/el/9/SRPMS
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1


##
## RabbitMQ Server
##

[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/$basearch
        https://yum1.rabbitmq.com/rabbitmq/el/9/$basearch
repo_gpgcheck=1
enabled=1
# Cloudsmith's repository key and RabbitMQ package signing key
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9-noarch]
name=rabbitmq-el9-noarch
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/noarch
        https://yum1.rabbitmq.com/rabbitmq/el/9/noarch
repo_gpgcheck=1
enabled=1
# Cloudsmith's repository key and RabbitMQ package signing key
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9-source]
name=rabbitmq-el9-source
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/SRPMS
        https://yum1.rabbitmq.com/rabbitmq/el/9/SRPMS
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
gpgcheck=0
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md
```

- Update package metadata:
```bash
dnf update -y
```

- Next install dependencies from the standard repositories:
```bash
## install these dependencies from standard OS repositories
dnf install -y socat logrotate
```

- Finally, install modern Erlang and RabbitMQ:
```bash
## install RabbitMQ and zero dependency Erlang
dnf install -y erlang rabbitmq-server
```

- Add the `openstack` user:

```bash
rabbitmqctl add_user openstack RABBIT_PASS
```
Replace `RABBIT_PASS` with a suitable password.

- Permit configuration, write, and read access for the `openstack` user:

```bash
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

- Enable `rabbitmq-server`
```bash
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

#### 5. Install Memcached:

**Install and configure controller node**

- Install the packages:
```bash
yum install memcached python3-memcached -y
```

- Edit the `/etc/memcached.conf` file and configure the service to use the management IP address of the controller node. This is to enable access by other nodes via the management network:
```bash
OPTIONS="-l 127.0.0.1,::1,controller"
```

- Restart the Memcached service:
```bash
systemctl enable memcached.service
systemctl start memcached.service
```

#### 6. Install Etcd

**Install and configure controller node**

- Install the `etcd` package:
```bash
yum install etcd -y
```

- Edit the `/etc/default/etcd` file and set the `ETCD_INITIAL_CLUSTER`, `ETCD_INITIAL_ADVERTISE_PEER_URLS`, `ETCD_ADVERTISE_CLIENT_URLS`, `ETCD_LISTEN_CLIENT_URLS` to the management IP address of the controller node to enable access by other nodes via the management network:

```bash
sudo sed -i '/ETCD_NAME=/cETCD_NAME="controller"' /etc/default/etcd
sudo sed -i '/ETCD_DATA_DIR=/cETCD_DATA_DIR="/var/lib/etcd"' /etc/default/etcd
sudo sed -i '/ETCD_INITIAL_CLUSTER_STATE=/cETCD_INITIAL_CLUSTER_STATE="new"' /etc/default/etcd
sudo sed -i '/ETCD_INITIAL_CLUSTER_TOKEN=/cETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"' /etc/default/etcd
sudo sed -i '/ETCD_INITIAL_CLUSTER=/cETCD_INITIAL_CLUSTER="controller=http://<IP_MGNT_CONTROLLER>:2380"' /etc/default/etcd
sudo sed -i '/ETCD_INITIAL_ADVERTISE_PEER_URLS=/cETCD_INITIAL_ADVERTISE_PEER_URLS="http://<IP_MGNT_CONTROLLER>:2380"' /etc/default/etcd
sudo sed -i '/ETCD_ADVERTISE_CLIENT_URLS=/cETCD_ADVERTISE_CLIENT_URLS="http://<IP_MGNT_CONTROLLER>:2379"' /etc/default/etcd
sudo sed -i '/ETCD_LISTEN_PEER_URLS=/cETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"' /etc/default/etcd
sudo sed -i '/ETCD_LISTEN_CLIENT_URLS=/cETCD_LISTEN_CLIENT_URLS="http://<IP_MGNT_CONTROLLER>:2379"' /etc/default/etcd
```
Replace `IP_MGNT_CONTROLLER` with your controller management IP

- Enable and start the etcd service:
```bash
systemctl enable etcd
systemctl start etcd
```

---
## OpenStack Services Installation

At a minimum, you need to install the following services. Install the services in the order specified below:
- Identity service – [keystone installation for 2023.2 (Bobcat)](https://docs.openstack.org/keystone/2023.2/install/) 
- Image service – [glance installation for 2023.2 (Bobcat)](https://docs.openstack.org/glance/2023.2/install/)
- Placement service – [placement installation for 2023.2 (Bobcat)](https://docs.openstack.org/placement/2023.2/install/)
- Compute service – [nova installation for 2023.2 (Bobcat)](https://docs.openstack.org/nova/2023.2/install/)
- Networking service – [neutron installation for 2023.2 (Bobcat)](https://docs.openstack.org/neutron/2023.2/install/)
We advise to also install the following components after you have installed the minimal deployment services:
- Dashboard – [horizon installation for 2023.2 (Bobcat)](https://docs.openstack.org/horizon/2023.2/install/)
- Block Storage service – [cinder installation for 2023.2 (Bobcat)](https://docs.openstack.org/cinder/2023.2/install/)

#### Keystone Installation (Identity service)

**Install and configure controller node**

- Create the `keystone` database and grant proper access to the `keystone` database:
```bash
mysql -u root -pROOT_PASSWORD -e "CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS'; 
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
FLUSH PRIVILEGES;"
```
Replace `KEYSTONE_DBPASS` with a suitable password

- Run the following command to install the keystone packages:
```bash
yum install openstack-keystone httpd python3-mod_wsgi -y
```

- Edit the `/etc/keystone/keystone.conf` file and complete the following actions:
```bash
sudo crudini --set /etc/keystone/keystone.conf database connection "mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone"
sudo crudini --set /etc/keystone/keystone.conf token provider "fernet"
```
Replace `KEYSTONE_DBPASS` with the password you chose for the database.

- Populate the Identity service database:
```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

- Initialize Fernet key repositories:
```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

> [!NOTE]
> The `--keystone-user` and `--keystone-group` flags are used to specify the operating system’s user/group that will be used to run keystone. These are provided to allow running keystone under another operating system user/group. In the example below, we call the user & group `keystone`.

- Bootstrap the Identity service:
```bash
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
Replace `ADMIN_PASS` with a suitable password for an administrative user.

- Edit the `/etc/httpd/conf/httpd.conf` file and configure the `ServerName` option to reference the controller node:
```bash
ServerName controller
```
The `ServerName` entry will need to be added if it does not already exist

- Create a link to the `/usr/share/keystone/wsgi-keystone.conf` file:
```bash
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

- Restart the Apache service:
```bash
systemctl enable httpd.service
systemctl start httpd.service
```
- Create client environment scripts for the `admin`: 

```bash
cat << EOF > /root/admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```
These values shown here are the default ones created from `keystone-manage bootstrap`.
Replace `ADMIN_PASS` with the password used in the `keystone-manage bootstrap` command in `keystone-install-configure-ubuntu`.

- Activate by
```bash
source /root/admin-openrc
```

Verify operation
- Activate admin profile:
```bash
source /root/admin-openrc
```
- As the `admin` user, request an authentication token:
```bash
openstack token issue
```
This command uses the password for the `admin` user

Sample result: 

- This guide uses a service project that contains a unique user for each service that you add to your environment. Create the `service` project:
```bash
openstack project create --domain default \
  --description "Service Project" service
```


#### Glance Installation (Image service)

**Install and configure controller node**

- Create the `glance` database and grant proper access to the `glance` database:
```bash
mysql -u root -pROOT_PASSWORD -e "CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
FLUSH PRIVILEGES;"
```
Replace `GLANCE_DBPASS` with a suitable password.

- Activate `admin` profile
```bash
source /root/admin-openrc
```

- Create the `glance` user
```bash
openstack user create --domain default --password-prompt glance
```

- Add the `admin` role to the `glance` user and `service` project:
```bash
openstack role add --project service --user glance admin
```

- Create the `glance` service entity:
```bash
openstack service create --name glance \
  --description "OpenStack Image" image
```

- Create the Image service API endpoints:
```bash
openstack endpoint create --region RegionOne \
  image public http://controller:9292

openstack endpoint create --region RegionOne \
  image internal http://controller:9292

openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```

- Install the glance packages:

```bash
yum install openstack-glance -y
```

Edit the `/etc/glance/glance-api.conf` file and complete the following actions
- In the `[database]` section, configure database access:
```bash
sudo crudini --set /etc/glance/glance-api.conf database connection "mysql+pymysql://glance:GLANCE_DBPASS@controller/glance"
```
Replace `GLANCE_DBPASS` with the password you chose for the Image service database.

- In the `[keystone_authtoken]` and `[paste_deploy]` sections, configure Identity service access:
```bash
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri "http://controller:5000"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url "http://controller:5000"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers "controller:11211"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type "password"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name "Default"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name "Default"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name "service"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken username "glance"
sudo crudini --set /etc/glance/glance-api.conf keystone_authtoken password "GLANCE_PASS"
sudo crudini --set /etc/glance/glance-api.conf paste_deploy flavor "keystone"
```
Replace `GLANCE_PASS` with the password you chose for the `glance` user in the Identity service.

- In the `[glance_store]` section, configure the local file system store and location of image files:
```bash
sudo crudini --set /etc/glance/glance-api.conf DEFAULT enabled_backends "fs:file"
sudo crudini --set /etc/glance/glance-api.conf glance_store default_backend "fs"
sudo crudini --set /etc/glance/glance-api.conf fs filesystem_store_datadir "/var/lib/glance/images/"
```

- In the `[oslo_limit]` section, configure access to keystone:
```bash
sudo crudini --set /etc/glance/glance-api.conf oslo_limit auth_url "http://controller:5000"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit auth_type "password"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit user_domain_id "default"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit username "glance"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit system_scope "all"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit password "GLANCE_PASS"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit endpoint_id "340be3625e9b4239a6415d034e98aace"
sudo crudini --set /etc/glance/glance-api.conf oslo_limit region_name "RegionOne"
```
Replace `GLANCE_PASS` with the password you chose for the `glance` user in the Identity service.

- Make sure that the glance account has reader access to system-scope resources (like limits):
```bash
openstack role add --user glance --user-domain Default --system all reader
```

- Populate the Image service database
```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

- Restart and enable the Image services
```bash
systemctl enable openstack-glance-api.service
systemctl start openstack-glance-api.service
```

Verify operation:
- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- Download the source image:
```bash
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

- Upload the image to the Image service using the [QCOW2] disk format, [bare] container format, and public visibility so all projects can access it:

```bash
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```

For information about the **glance** parameters, see [Image service (glance) command-line client](https://docs.openstack.org/python-glanceclient/latest/cli/details.html) in the `OpenStack User Guide`.

For information about disk and container formats for images, see [Disk and container formats for images](https://docs.openstack.org/image-guide/image-formats.html) in the `OpenStack Virtual Machine Image Guide`.

- Confirm upload of the image and validate attributes you have upload:
```bash
glance image-list
```


#### Placement Installation (Placement service)

**Install and configure controller node**

- Create the `placement` database and grant proper access to the database:
```bash
mysql -u root -pROOT_PASSWORD -e "CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
FLUSH PRIVILEGES;"
```
Replace `PLACEMENT_DBPASS` with a suitable password.

- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- Create a Placement service user using your chosen `PLACEMENT_PASS`:
```bash
openstack user create --domain default --password-prompt placement
```

- Add the Placement user to the service project with the admin role:
```bash
openstack role add --project service --user placement admin
```

- Create the Placement API entry in the service catalog:
```bash
openstack service create --name placement \
  --description "Placement API" placement
```

- Create the Placement API service endpoints:

> [!NOTE] 
> Depending on your environment, You might want to create the URL for the endpoint will vary by port (possibly 8780 instead of 8778, or no port at all) and hostname. You are responsible for determining the correct URL.

```bash
openstack endpoint create --region RegionOne \
  placement public http://controller:8778

openstack endpoint create --region RegionOne \
  placement internal http://controller:8778

openstack endpoint create --region RegionOne \
  placement admin http://controller:8778
```

- Install the placement packages:
```bash
yum install openstack-placement-api -y
```

Edit the `/etc/placement/placement.conf` file and complete the following actions:
- In the `[placement_database]` section, configure database access:
```bash
sudo crudini --set /etc/placement/placement.conf placement_database connection "mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement"
```
Replace `PLACEMENT_DBPASS` with the password you chose for the placement database.

- In the `[api]` and `[keystone_authtoken]` sections, configure Identity service access:
```bash
sudo crudini --set /etc/placement/placement.conf api auth_strategy "keystone"

sudo crudini --set /etc/placement/placement.conf keystone_authtoken auth_url "http://controller:5000/v3" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken memcached_servers "controller:11211" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken auth_type "password" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken project_domain_name "Default" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken user_domain_name "Default" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken project_name "service" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken username "placement" 
sudo crudini --set /etc/placement/placement.conf keystone_authtoken password "PLACEMENT_PASS"
```
Replace `PLACEMENT_PASS` with the password you chose for the `placement` user in the Identity service.

- Populate the `placement` database
```bash
su -s /bin/sh -c "placement-manage db sync" placement
```

- Set permission - Edit `/etc/httpd/conf.d/00-placement-api.conf` file and add this:
```
<Directory /usr/bin>
    Require all denied
    <Files "placement-api">
      <RequireAll>
        Require all granted
        Require not env blockAccess
      </RequireAll>
    </Files>
</Directory>
```

- Reload the web server to adjust to get new configuration settings for placement
```bash
systemctl restart httpd
```

Verify Installation:
- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- Perform status checks to make sure everything is in order
```bash
placement-status upgrade check
```
The output of that command will vary by release. See [placement-status upgrade check](https://docs.openstack.org/placement/2024.1/cli/placement-status.html#placement-status-checks) for details.


#### Nova Installation (Compute service)

**Install and configure controller node**

Create the `nova_api`, `nova`, and `nova_cell0` databases and grant proper access to the databases:
```bash
mysql -u root -pROOT_PASSWORD -e "CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
FLUSH PRIVILEGES;"
```
Replace `NOVA_DBPASS` with a suitable password.

- Activate `admin` profile
```bash
source /root/admin-openrc
```

- Create the `nova` user:
```bash
openstack user create --domain default --password-prompt nova
```

- Add the `admin` role to the `nova` user:
```bash
openstack role add --project service --user nova admin
```

- Create the `nova` service entity:
```bash
openstack service create --name nova \
  --description "OpenStack Compute" compute
```

- Create the Compute API service endpoints:
```bash
openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1

openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1

openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1
```

- Install the nova packages:
```bash
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-novncproxy openstack-nova-scheduler -y
```

Edit the `/etc/nova/nova.conf` file and complete the following actions:
- In the `[api_database]` and `[database]` sections, configure database access:
```bash
sudo crudini --set /etc/nova/nova.conf api_database connection "mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api"
sudo crudini --set /etc/nova/nova.conf database connection "mysql+pymysql://nova:NOVA_DBPASS@controller/nova"
```
Replace `NOVA_DBPASS` with the password you chose for the Compute databases.

- In the `[DEFAULT]` section:
```bash
sudo crudini --set /etc/nova/nova.conf DEFAULT enabled_apis "osapi_compute,metadata"
sudo crudini --set /etc/nova/nova.conf DEFAULT transport_url "rabbit://openstack:RABBIT_PASS@controller:5672/"
sudo crudini --set /etc/nova/nova.conf DEFAULT log_dir "/var/log/nova"
sudo crudini --set /etc/nova/nova.conf DEFAULT lock_path "/var/lock/nova"
sudo crudini --set /etc/nova/nova.conf DEFAULT state_path "/var/lib/nova"
```
Replace `RABBIT_PASS` with the password you chose for the `openstack` account in `RabbitMQ`.

- In the `[api]` and `[keystone_authtoken]` sections, configure Identity service access:
```bash
sudo crudini --set /etc/nova/nova.conf api auth_strategy "keystone"

sudo crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri "http://controller:5000/" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken auth_url "http://controller:5000/" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers "controller:11211" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken auth_type "password" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name "Default" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name "Default" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken project_name "service" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken username "nova" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken password "NOVA_PASS"
```
Replace `NOVA_PASS` with the password you chose for the `nova` user in the Identity service.

- In the `[service_user]` section, configure service user tokens:
```bash
sudo crudini --set /etc/nova/nova.conf service_user send_service_user_token "true"
sudo crudini --set /etc/nova/nova.conf service_user auth_url "http://controller:5000/identity"
sudo crudini --set /etc/nova/nova.conf service_user auth_strategy "keystone"
sudo crudini --set /etc/nova/nova.conf service_user auth_type "password"
sudo crudini --set /etc/nova/nova.conf service_user project_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf service_user project_name "service"
sudo crudini --set /etc/nova/nova.conf service_user user_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf service_user username "nova"
sudo crudini --set /etc/nova/nova.conf service_user password "NOVA_PASS"
```
Replace `NOVA_PASS` with the password you chose for the `nova` user in the Identity service.

- In the `[DEFAULT]` section, configure the `my_ip` option to use the management interface IP address of the controller node:
```bash
sudo crudini --set /etc/nova/nova.conf DEFAULT my_ip "IP_MGNT_CONTROLLER"
```
Replace `IP_MGNT_CONTROLLER` with your controller management IP

- In the `[vnc]` section, configure the VNC proxy to use the management interface IP address of the controller node:
```bash
sudo crudini --set /etc/nova/nova.conf vnc enabled true 
sudo crudini --set /etc/nova/nova.conf vnc server_listen " \$my_ip"
sudo crudini --set /etc/nova/nova.conf vnc server_proxyclient_address " \$my_ip"
```

- In the `[glance]` section, configure the location of the Image service API:
```bash
sudo crudini --set /etc/nova/nova.conf glance api_servers "http://controller:9292"
```

- In the `[oslo_concurrency]` section, configure the lock path:
```bash
sudo crudini --set /etc/nova/nova.conf oslo_concurrency lock_path "/var/lib/nova/tmp"
```

- In the `[placement]` section, configure access to the Placement service:
```bash
sudo crudini --set /etc/nova/nova.conf placement region_name "RegionOne"
sudo crudini --set /etc/nova/nova.conf placement project_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf placement project_name "service"
sudo crudini --set /etc/nova/nova.conf placement auth_type "password"
sudo crudini --set /etc/nova/nova.conf placement user_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf placement auth_url "http://controller:5000/v3"
sudo crudini --set /etc/nova/nova.conf placement username "placement"
sudo crudini --set /etc/nova/nova.conf placement password "PLACEMENT_PASS"
```
Replace `PLACEMENT_PASS` with the password you choose for the `placement` service user created when installing [Placement]

- Populate the `nova-api` database:
```bash 
su -s /bin/sh -c "nova-manage api_db sync" nova
```
Ignore any deprecation messages in this output.

- Register the `cell0` database:
```bash
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

- Create the `cell1` cell:
```bash
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

- Populate the nova database:
```bash
su -s /bin/sh -c "nova-manage db sync" nova
```

- Verify nova cell0 and cell1 are registered correctly:
```bash
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```
sample result (uuid might differ per deployment):
```
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
|  Name |                 UUID                 |                   Transport URL                    |                     Database Connection                      | Disabled |
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                       none:/                       | mysql+pymysql://nova:****@controller/nova_cell0?charset=utf8 |  False   |
| cell1 | f690f4fd-2bc5-4f15-8145-db561a7b9d3d | rabbit://openstack:****@controller:5672/nova_cell1 | mysql+pymysql://nova:****@controller/nova_cell1?charset=utf8 |  False   |
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
```

- Restart and enable the Compute services:
```bash
systemctl enable \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
systemctl restart \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```


**Install and configure compute node**

> [!IMPORTANT] :
> The service supports several hypervisors to deploy instances or virtual machines (VMs). For simplicity, this configuration uses the Quick EMUlator (QEMU) hypervisor with the kernel-based VM (KVM) extension on compute nodes that support hardware acceleration for virtual machines. On legacy hardware, this configuration uses the generic QEMU hypervisor. You can follow these instructions with minor modifications to horizontally scale your environment with additional compute nodes.

- Install the nova packages:
```bash
yum install openstack-nova-compute qemu-kvm libvirt virt-install -y
```

Edit the `/etc/nova/nova.conf` file and complete the following actions:
- In the `[DEFAULT]` section:

```bash

```

set in [DEFAULT]
log_dir = /var/log/nova  
lock_path = /var/lock/nova  
state_path = /var/lib/nova

```bash
sudo crudini --set /etc/nova/nova.conf DEFAULT enabled_apis "osapi_compute,metadata"
sudo crudini --set /etc/nova/nova.conf DEFAULT transport_url "rabbit://openstack:RABBIT_PASS@controller"
sudo crudini --set /etc/nova/nova.conf DEFAULT log_dir "/var/log/nova"
sudo crudini --set /etc/nova/nova.conf DEFAULT lock_path "/var/lock/nova"
sudo crudini --set /etc/nova/nova.conf DEFAULT state_path "/var/lib/nova"
```
Replace `RABBIT_PASS` with the password you chose for the `openstack` account in `RabbitMQ`.

- In the `[api]` and `[keystone_authtoken]` sections, configure Identity service access:
```bash
sudo crudini --set /etc/nova/nova.conf api auth_strategy "keystone"

sudo crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri "http://controller:5000/" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken auth_url "http://controller:5000/" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers "controller:11211" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken auth_type "password" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name "Default" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name "Default" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken project_name "service" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken username "nova" 
sudo crudini --set /etc/nova/nova.conf keystone_authtoken password "NOVA_PASS"
```
Replace `NOVA_PASS` with the password you chose for the `nova` user in the Identity service.

- In the `[service_user]` section, configure service user tokens:
```bash
sudo crudini --set /etc/nova/nova.conf service_user send_service_user_token "true"
sudo crudini --set /etc/nova/nova.conf service_user auth_url "http://controller:5000/identity"
sudo crudini --set /etc/nova/nova.conf service_user auth_strategy "keystone"
sudo crudini --set /etc/nova/nova.conf service_user auth_type "password"
sudo crudini --set /etc/nova/nova.conf service_user project_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf service_user project_name "service"
sudo crudini --set /etc/nova/nova.conf service_user user_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf service_user username "nova"
sudo crudini --set /etc/nova/nova.conf service_user password "NOVA_PASS"
```
Replace `NOVA_PASS` with the password you chose for the `nova` user in the Identity service.

- In the `[DEFAULT]` section, configure the `my_ip` option:
```bash
sudo crudini --set /etc/nova/nova.conf DEFAULT my_ip "IP_MGNT_COMPUTE"
```
Replace `IP_MGNT_COMPUTE` with compute node management IP

- In the `[vnc]` section, enable and configure remote console access:
```bash
sudo crudini --set /etc/nova/nova.conf vnc enabled true
sudo crudini --set /etc/nova/nova.conf vnc server_listen 0.0.0.0
sudo crudini --set /etc/nova/nova.conf vnc server_proxyclient_address " \$my_ip"
sudo crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://IP_MGNT_COMPUTE:6080/vnc_auto.html
```
Replace `IP_MGNT_COMPUTE` with your public IP of controller node

If the web browser to access remote consoles resides on a host that cannot resolve the `controller` hostname, you must replace `controller` with the management interface IP address of the controller node.

- In the `[glance]` section, configure the location of the Image service API:
```bash
sudo crudini --set /etc/nova/nova.conf glance api_servers "http://controller:9292"
```

- In the `[oslo_concurrency]` section, configure the lock path:
```bash
sudo crudini --set /etc/nova/nova.conf oslo_concurrency lock_path "/var/lib/nova/tmp"
```

- In the `[placement]` section, configure the Placement API:
```bash
sudo crudini --set /etc/nova/nova.conf placement region_name "RegionOne"
sudo crudini --set /etc/nova/nova.conf placement project_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf placement project_name "service"
sudo crudini --set /etc/nova/nova.conf placement auth_type "password"
sudo crudini --set /etc/nova/nova.conf placement user_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf placement auth_url "http://controller:5000/v3"
sudo crudini --set /etc/nova/nova.conf placement username "placement"
sudo crudini --set /etc/nova/nova.conf placement password "123456789"
```
Replace `PLACEMENT_PASS` with the password you choose for the `placement` user in the Identity service. Comment out any other options in the `[placement]` section.

- Determine whether your compute node supports hardware acceleration for virtual machines:
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

If this command returns a value of `one or greater`, your compute node supports hardware acceleration which typically requires no additional configuration.

```bash
sudo crudini --set /etc/nova/nova.conf libvirt virt_type kvm
```

If this command returns a value of **`zero`**, your compute node does not support hardware acceleration and you must configure `libvirt` to use QEMU instead of KVM.
- Edit the `[libvirt]` section in the `/etc/nova/nova-compute.conf` file as follows:
```bash
sudo crudini --set /etc/nova/nova.conf libvirt virt_type qemu
```

Uncomment this
```bash
compute_driver=libvirt.LibvirtDriver
```

- Start the Compute service
```bash
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

> [!NOTE] 
> If the `nova-compute` service fails to start, check `/var/log/nova/nova-compute.log`. The error message `AMQP server on controller:5672 is unreachable` likely indicates that the firewall on the controller node is preventing access to port 5672. Configure the firewall to open port 5672 on the controller node and restart `nova-compute` service on the compute node.


**Add the compute node to the cell database** (Run the following commands on the **controller** node.)

- Activate `admin` profile then confirm there are compute hosts in the database:
```bash
source /root/admin-openrc
openstack compute service list --service nova-compute
```

![[Pasted image 20241027231049.png]]

- To discover new compute hosts:
```bash
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

> [!NOTE] 
> When you add new compute nodes, you must run `nova-manage cell_v2 discover_hosts` on the controller node to register those new compute nodes. Alternatively, you can set an appropriate interval in `/etc/nova/nova.conf`
```
[scheduler]
discover_hosts_in_cells_interval = 300
```

**Verify operation of the Compute service:**
**On Controller Node:**
- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- List service components to verify successful launch and registration of each process:
```bash
openstack compute service list
```
This output should indicate two service components enabled on the controller node and one service component enabled on the compute node.

- List API endpoints in the Identity service to verify connectivity with the Identity service: Below endpoints list may differ depending on the installation of OpenStack components.
```bash
openstack catalog list
```
Ignore any warnings in this output.

- List images in the Image service to verify connectivity with the Image service:
```bash
openstack image list
```

- Check the cells and placement API are working successfully and that other necessary prerequisites are in place:
```bash
nova-manage cell_v2 simple_cell_setup
nova-status upgrade check
```

#### Neutron Installation (Networking service)

**Install and configure controller node**

- Create the `neutron` database and grant proper access to the `neutron` database, replacing `NEUTRON_DBPASS` with a suitable password:
```bash
mysql -u root -pROOT_PASSWORD -e "CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
 GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
FLUSH PRIVILEGES;"
```

- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- Create the `neutron` user:
```bash
openstack user create --domain default --password-prompt neutron
```

- Add the `admin` role to the `neutron` user:
```bash
openstack role add --project service --user neutron admin
```

- Create the `neutron` service entity:
```bash
openstack service create --name neutron \
  --description "OpenStack Networking" network
```

- Create the Networking service API endpoints
```bash
openstack endpoint create --region RegionOne \
  network public http://controller:9696

openstack endpoint create --region RegionOne \
  network internal http://controller:9696

openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```

**This guide will process to install self-service networks**

- Install neutron packages:
```bash
yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-openvswitch ebtables -y
```

Edit the `/etc/neutron/neutron.conf` file and complete the following actions:
- In the `[database]` section, configure database access:
```bash
sudo crudini --set /etc/neutron/neutron.conf database connection "mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron"
```
Replace `NEUTRON_DBPASS` with the password you chose for the database.

- In the `[DEFAULT]` section, enable the Modular Layer 2 (ML2) plug-in and router service:
```bash
sudo crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin "ml2" 
sudo crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins "router"
```

- In the `[DEFAULT]` section, configure `RabbitMQ` message queue access:
```bash
sudo crudini --set /etc/neutron/neutron.conf DEFAULT transport_url "rabbit://openstack:RABBIT_PASS@controller"
```
Replace `RABBIT_PASS` with the password you chose for the `openstack` account in RabbitMQ.

- In the `[DEFAULT]` and `[keystone_authtoken]` sections, configure Identity service access:
```bash
sudo crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy "keystone"

sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri "http://controller:5000" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url "http://controller:5000" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers "controller:11211" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type "password" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name "Default" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name "Default" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name "service" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken username "neutron" 
sudo crudini --set /etc/neutron/neutron.conf keystone_authtoken password "NEUTRON_PASS"
```
Replace `NEUTRON_PASS` with the password you chose for the `neutron` user in the Identity service.

- In the `[DEFAULT]` and `[nova]` sections, configure Networking to notify Compute of network topology changes:
```bash
sudo crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes "true" 
sudo crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes "true"

sudo crudini --set /etc/neutron/neutron.conf nova auth_url "http://controller:5000" 
sudo crudini --set /etc/neutron/neutron.conf nova auth_type "password" 
sudo crudini --set /etc/neutron/neutron.conf nova project_domain_name "Default" 
sudo crudini --set /etc/neutron/neutron.conf nova user_domain_name "Default" 
sudo crudini --set /etc/neutron/neutron.conf nova region_name "RegionOne" 
sudo crudini --set /etc/neutron/neutron.conf nova project_name "service" 
sudo crudini --set /etc/neutron/neutron.conf nova username "nova" 
sudo crudini --set /etc/neutron/neutron.conf nova password "NOVA_PASS"
```
Replace `NOVA_PASS` with the password you chose for the `nova` user in the Identity service.

- In the `[oslo_concurrency]` section, configure the lock path:
```bash
sudo crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path "/var/lib/neutron/tmp"
```

The ML2 plug-in uses the Linux bridge mechanism to build layer-2 (bridging and switching) virtual networking infrastructure for instances. Edit the `/etc/neutron/plugins/ml2/ml2_conf.ini` file and complete the following actions:
- In the `[ml2]` section, enable flat, VLAN, and VXLAN networks:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers "flat,vlan,vxlan"
```

- In the `[ml2]` section, enable VXLAN self-service networks:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types "vxlan"
```

- In the `[ml2]` section, enable the Linux bridge and layer-2 population mechanisms:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers "openvswitch,l2population"
```

> [!WARNING]
>After you configure the ML2 plug-in, removing values in the `type_drivers` option can lead to database inconsistency.

> [!NOTE]
> The Linux bridge agent only supports VXLAN overlay networks.

- In the `[ml2]` section, enable the port security extension driver:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers "port_security"
```

- In the `[ml2_type_flat]` section, configure the provider virtual network as a flat network:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks "provider"
```

- In the `[ml2_type_vxlan]` section, configure the VXLAN network identifier range for self-service networks:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges "1:1000"
```

The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups. Edit the `/etc/neutron/plugins/ml2/openvswitch_agent.ini` file and complete the following actions:
- In the `[ovs]` section, map the provider virtual network to the provider physical bridge and configure the IP address of the physical network interface that handles overlay networks:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings "provider:br-provider"
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip "IP_MGNT_CONTROLLER"
```

- Ensure Openvswitch service is running:
```
sudo systemctl status openvswitch
# If not running
sudo systemctl start openvswitch
```

- Ensure `PROVIDER_BRIDGE_NAME` external bridge is created and `PROVIDER_INTERFACE_NAME` is added to that bridge
```bash
export PROVIDER_BRIDGE_NAME=br-provider
export PROVIDER_INTERFACE_NAME=EXTERNAL_INT_CONTROLLER
ovs-vsctl add-br $PROVIDER_BRIDGE_NAME
ovs-vsctl add-port $PROVIDER_BRIDGE_NAME $PROVIDER_INTERFACE_NAME
```
Replace `EXTERNAL_INT_CONTROLLER` with your external interface of controller

Example:
```bash
export PROVIDER_BRIDGE_NAME=br-provider
export PROVIDER_INTERFACE_NAME=ens160
ovs-vsctl add-br $PROVIDER_BRIDGE_NAME
ovs-vsctl add-port $PROVIDER_BRIDGE_NAME $PROVIDER_INTERFACE_NAME
```

- Setting temp IP for bridge interface based on your setup:

```bash
ip a flush EXTERNAL_INT_CONTROLLER
ip a add IP_EXTERNAL_CONTROLLER/24 dev br-provider
ip link set br-provider up
ip r add default via IP_GATEWAY_EXTERNAL_CONTROLLER
echo "nameserver IP_GATEWAY_EXTERNAL_CONTROLLER" > /etc/resolv.conf
```

For example: 
```bash
ip a flush ens160
ip a add 192.168.43.103/24 dev br-provider
ip link set br-provider up
ip r add default via 192.168.43.1
echo "nameserver 192.168.43.1" > /etc/resolv.conf
```

- Persist network for bridge interface 
Example: 
```bash
nmcli connection modify br-provider ipv4.addresses 192.168.43.103/24
nmcli connection modify br-provider ipv4.gateway 192.168.43.1
nmcli connection modify br-provider ipv4.dns 192.168.43.1
nmcli connection modify br-provider ipv4.method manual
nmcli connection modify br-provider connection.autoconnect yes
```
Replace these IP with your EXTERNAL IP configuration of controller node. Make sure dont change the `br-provider` interface name

```bash
nmcli connection modify ens160 master br-provider
nmcli connection modify ens160 connection.autoconnect yes
```
Replace the `ens160` with your EXTERNAL IP interface name of controller node. 

- Restart network service
```
sudo systemctl restart network
```

- In the `[agent]` section, enable VXLAN overlay networks and enable layer-2 population:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types "vxlan"
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population "true"
```

- In the `[securitygroup]` section, enable security groups and configure the Open vSwitch native or the hybrid iptables firewall driver:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup enable_security_group "true"
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver "openvswitch"
```

- In the case of using the hybrid iptables firewall driver, ensure your Linux operating system kernel supports network bridge filters by verifying all the following `sysctl` values are set to `1`:
```bash
echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables = 1' | sudo tee -a /etc/sysctl.conf
sudo modprobe br_netfilter
sudo /sbin/sysctl -p
```

The Layer-3 (L3) agent provides routing and NAT services for self-service virtual networks. Edit the `/etc/neutron/l3_agent.ini` file and complete the following actions:
- In the `[DEFAULT]` section, configure the Open vSwitch interface driver:
```bash
sudo crudini --set /etc/neutron/l3_agent.ini DEFAULT interface_driver "openvswitch"
```

The DHCP agent provides DHCP services for virtual networks. Edit the `/etc/neutron/dhcp_agent.ini` file and complete the following actions:
- In the `[DEFAULT]` section, configure the Open vSwitch interface driver, Dnsmasq DHCP driver, and enable isolated metadata so instances on provider networks can access metadata over the network:
```bash
sudo crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver "openvswitch"
sudo crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver "neutron.agent.linux.dhcp.Dnsmasq"
sudo crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata "true"
```

The metadata agent provides configuration information such as credentials to instances. Edit the `/etc/neutron/metadata_agent.ini` file and complete the following actions:
- In the `[DEFAULT]` section, configure the metadata host and shared secret:
```bash
sudo crudini --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host "controller"
sudo crudini --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret "METADATA_SECRET"
```
Replace `METADATA_SECRET` with a suitable secret for the metadata proxy.

Configure the Compute service to use the Networking service. Edit the `/etc/nova/nova.conf` file and perform the following actions:
- In the `[neutron]` section, configure access parameters, enable the metadata proxy, and configure the secret:
```bash
sudo crudini --set /etc/nova/nova.conf neutron auth_url "http://controller:5000"
sudo crudini --set /etc/nova/nova.conf neutron auth_type "password"
sudo crudini --set /etc/nova/nova.conf neutron project_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf neutron user_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf neutron region_name "RegionOne"
sudo crudini --set /etc/nova/nova.conf neutron project_name "service"
sudo crudini --set /etc/nova/nova.conf neutron username "neutron"
sudo crudini --set /etc/nova/nova.conf neutron password "NEUTRON_PASS"
sudo crudini --set /etc/nova/nova.conf neutron service_metadata_proxy "true"
sudo crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret "METADATA_SECRET"
```
Replace `NEUTRON_PASS` with the password you chose for the `neutron` user in the Identity service.
Replace `METADATA_SECRET` with the secret you chose for the metadata proxy.

- The Networking service initialization scripts expect a symbolic link /etc/neutron/plugin.ini pointing to the ML2 plug-in configuration file, /etc/neutron/plugins/ml2/ml2_conf.ini. If this symbolic link does not exist, create it using the following command:

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

- Populate the database
```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

- Restart the Compute API service
```bash
systemctl restart openstack-nova-api.service
```

- Restart and enable the Networking services
```bash
systemctl enable neutron-server.service \
  neutron-openvswitch-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
systemctl start neutron-server.service \
  neutron-openvswitch-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```

- Enable and start the layer-3 service:
```bash
systemctl enable neutron-l3-agent.service
systemctl start neutron-l3-agent.service
```

**Install and configure compute node**

- Install neutron packages:
```bash
yum install openstack-neutron-openvswitch -y
```

Edit the `/etc/neutron/neutron.conf` file and complete the following actions:
- In the `[database]` section, comment out any `connection` options because compute nodes do not directly access the database.
- In the `[DEFAULT]` section, configure `RabbitMQ` message queue access:
```bash
sudo crudini --set /etc/neutron/neutron.conf DEFAULT transport_url "rabbit://openstack:RABBIT_PASS@controller"
```
Replace `RABBIT_PASS` with the password you chose for the `openstack` account in RabbitMQ.

- In the `[oslo_concurrency]` section, configure the lock path:
```bash
sudo crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path "/var/lib/neutron/tmp"
```

The Open vSwitch agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups. Edit the `/etc/neutron/plugins/ml2/openvswitch_agent.ini` file and complete the following actions:
- In the `[ovs]` section, map the provider virtual network to the provider physical bridge and configure the IP address of the physical network interface that handles overlay networks:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings "provider:br-provider"
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip "IP_MGNT_COMPUTE"
```
Also replace `IP_MGNT_COMPUTE` with the management IP address of the compute node.

- Ensure `PROVIDER_BRIDGE_NAME` external bridge is created and `PROVIDER_INTERFACE_NAME` is added to that bridge
```bash
export PROVIDER_BRIDGE_NAME=br-provider
export PROVIDER_INTERFACE_NAME=ens160
ovs-vsctl add-br $PROVIDER_BRIDGE_NAME
ovs-vsctl add-port $PROVIDER_BRIDGE_NAME $PROVIDER_INTERFACE_NAME
```

Example:
```bash
export PROVIDER_BRIDGE_NAME=br-provider
export PROVIDER_INTERFACE_NAME=ens160
ovs-vsctl add-br $PROVIDER_BRIDGE_NAME
ovs-vsctl add-port $PROVIDER_BRIDGE_NAME $PROVIDER_INTERFACE_NAME
```

- Setting temp IP for bridge interface:
```bash
ip a flush ens160
ip a add 192.168.43.105/24 dev br-provider
ip link set br-provider up
ip r add default via 192.168.43.1
echo "nameserver 192.168.43.1" > /etc/resolv.conf
```

- Persist network for bridge interface 
Example: 
```bash
nmcli connection modify br-provider ipv4.addresses 192.168.43.104/24
nmcli connection modify br-provider ipv4.gateway 192.168.43.1
nmcli connection modify br-provider ipv4.dns 192.168.43.1
nmcli connection modify br-provider ipv4.method manual
nmcli connection modify br-provider connection.autoconnect yes
```
Replace these IP with your EXTERNAL IP configuration of compute node. Make sure dont change the `br-provider` interface name

```bash
nmcli connection modify ens160 master br-provider
nmcli connection modify ens160 connection.autoconnect yes
```
Replace the `ens160` with your EXTERNAL IP interface name of compute node. 

- In the `[agent]` section, enable VXLAN overlay networks and enable layer-2 population:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types "vxlan"
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population "true"
```

- In the `[securitygroup]` section, enable security groups and configure the Open vSwitch native or the hybrid iptables firewall driver:
```bash
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup enable_security_group "true"
sudo crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver "openvswitch"
```

- In the case of using the hybrid iptables firewall driver, ensure your Linux operating system kernel supports network bridge filters by verifying all the following `sysctl` values are set to `1`:
```bash
echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables = 1' | sudo tee -a /etc/sysctl.conf
sudo modprobe br_netfilter
sudo /sbin/sysctl -p
```

Configure the Compute service to use the Networking service. Edit the `/etc/nova/nova.conf` file and perform the following actions:
- In the `[neutron]` section, configure access parameters:
```bash
sudo crudini --set /etc/nova/nova.conf neutron auth_url "http://controller:5000"
sudo crudini --set /etc/nova/nova.conf neutron auth_type "password"
sudo crudini --set /etc/nova/nova.conf neutron project_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf neutron user_domain_name "Default"
sudo crudini --set /etc/nova/nova.conf neutron region_name "RegionOne"
sudo crudini --set /etc/nova/nova.conf neutron project_name "service"
sudo crudini --set /etc/nova/nova.conf neutron username "neutron"
sudo crudini --set /etc/nova/nova.conf neutron password "NEUTRON_PASS"
```
Replace `NEUTRON_PASS` with the password you chose for the `neutron` user in the Identity service.

- Restart the Compute service:
```bash
systemctl restart openstack-nova-compute.service
```

- Restart and enable the Linux bridge agent:
```bash
systemctl enable neutron-openvswitch-agent.service
systemctl start neutron-openvswitch-agent.service
```

**Verify network installation:**
- **Run on Controller Node:**
- Activate `admin` profile
```bash
source /root/admin-openrc
```

- List agents to verify successful launch of the neutron agents **(run controller node)**:
```bash 
openstack network agent list
```
The output should indicate four agents on the controller node and one agent on each compute node with state UP

#### Horizon Installation (Dashboard)

**Install and configure controller node**

- Install the packages
```bash
yum install openstack-dashboard -y
```

Edit the `/etc/openstack-dashboard/local_settings` file and complete the following actions:

- Configure the dashboard to use OpenStack services on the `controller` node:
```
OPENSTACK_HOST = "controller"
```

- In the Dashboard configuration section, allow your hosts to access Dashboard:
```
ALLOWED_HOSTS = ['*']
```

- Configure the `memcached` session storage service:
```
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
```

- Enable the Identity API version 3:
```
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
```

> [!NOTE]
> In case your keystone run at 5000 port then you would mentioned keystone port here as well i.e. OPENSTACK_KEYSTONE_URL = “http://%s:5000/identity/v3” % OPENSTACK_HOST


- Enable support for domains (if not found just add in):
```
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
```

- Configure API versions (if not found just add in):
```
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
```

- Configure `Default` as the default domain for users that you create via the dashboard (if not found just add in):
```
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
```

- Configure `user` as the default role for users that you create via the dashboard (if not found just add in):
```
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```

- Configure webroot:
```
WEBROOT = '/dashboard/'
```

- Optionally, configure the time zone:
```
TIME_ZONE = "TIME_ZONE"
```
Replace `TIME_ZONE` with an appropriate time zone identifier. For more information, see the [list of time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

Edit file `/etc/httpd/conf.d/openstack-dashboard.conf`
- Add this line at the top file
```
WSGIApplicationGroup %{GLOBAL}
```

- Change `WSGIScriptAlias /dashboard /usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi` to
```
WSGIScriptAlias /dashboard /usr/share/openstack-dashboard/openstack_dashboard/wsgi.py
```

- Change `<Directory /usr/share/openstack-dashboard/openstack_dashboard/wsgi>` to
```
<Directory /usr/share/openstack-dashboard/openstack_dashboard>
```

- Create a redirect page
```
filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://192.168.43.103/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF
```

- Reload the web server configuration:
```bash
systemctl restart httpd.service memcached.service
```

#### Cinder Installation (Block Storage service)

**Install and configure controller node**

- Create the `cinder` database and grant proper access to the `cinder` database:
```bash
mysql -u root -pROOT_PASSWORD -e "CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'CINDER_DBPASS';
FLUSH PRIVILEGES;"
```
Replace `CINDER_DBPASS` with a suitable password.

- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- Create a `cinder` user:
```bash
openstack user create --domain default --password-prompt cinder
```

- Add the `admin` role to the `cinder` user:
```bash
openstack role add --project service --user cinder admin
```

- Create the `cinderv3` service entity:
```bash
openstack service create --name cinderv3 \
  --description "OpenStack Block Storage" volumev3
```

- Create the Block Storage service API endpoints:
```bash
openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

- Install the cinder packages:
```bash
yum install openstack-cinder -y
```

Edit the `/etc/cinder/cinder.conf` file and complete the following actions:
- In the `[database]` section, configure database access:
```bash
sudo crudini --set /etc/cinder/cinder.conf database connection "mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder"
```
Replace `CINDER_DBPASS` with the password you chose for the Block Storage database.

- In the `[DEFAULT]` section, configure `RabbitMQ` message queue access:
```bash
sudo crudini --set /etc/cinder/cinder.conf DEFAULT transport_url "rabbit://openstack:RABBIT_PASS@controller"

```
Replace `RABBIT_PASS` with the password you chose for the `openstack` account in `RabbitMQ`.

- In the `[DEFAULT]` and `[keystone_authtoken]` sections, configure Identity service access:
```bash
sudo crudini --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone

sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken www_authenticate_uri http://controller:5000 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller:5000 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken memcached_servers controller:11211 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_type password 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken project_domain_name default 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken user_domain_name default 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken project_name service 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken username cinder 
sudo crudini --set /etc/cinder/cinder.conf keystone_authtoken password CINDER_PASS
```
Replace `CINDER_PASS` with the password you chose for the `cinder` user in the Identity service.

- In the `[DEFAULT]` section, configure the `my_ip` option to use the management interface IP address of the controller node:
```bash
sudo crudini --set /etc/cinder/cinder.conf DEFAULT my_ip <IP_MGNT_CONTROLLER>
```
Replace `IP_MGNT_CONTROLLER` with management controller IP

- In the `[oslo_concurrency]` section, configure the lock path:
```bash
sudo crudini --set /etc/cinder/cinder.conf oslo_concurrency lock_path "/var/lib/cinder/tmp"
```

- Populate the Block Storage database:
```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```

Edit the `/etc/nova/nova.conf` file and add the following to it:
```bash
sudo crudini --set /etc/nova/nova.conf cinder os_region_name "RegionOne"
```

- Restart the Compute API service:
```bash
systemctl restart openstack-nova-api.service
```

- Restart the Block Storage services:
```bash
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```

**Verify cinder installation: **

- Activate `admin` profile:
```bash
source /root/admin-openrc
```

- List service components to verify successful launch of each process:
```bash
openstack volume service list
```


## Launch Instance

#### Create virtual networks

**On Controller Node**
**Create the provider network**

```bash
source /root/admin-openrc
```

- Create the network:
```bash
openstack network create  --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider
```

- Create a subnet on the network:
```bash
openstack subnet create --network provider \
  --allocation-pool start=START_IP_ADDRESS,end=END_IP_ADDRESS \
  --dns-nameserver DNS_RESOLVER --gateway PROVIDER_NETWORK_GATEWAY \
  --subnet-range PROVIDER_NETWORK_CIDR provider
```
Replace these values based on your EXTERNAL INTERFACE you setup on vm

Example:
```bash
openstack subnet create --network provider \
  --allocation-pool start=192.168.43.150,end=192.168.43.160 \
  --dns-nameserver 8.8.4.4 --gateway 192.168.43.1 \
  --subnet-range 192.168.43.0/24 provider
```

**Create the self-service network**

- Create the network:
```bash
openstack network create selfservice
```

- Create a subnet on the network:
```bash
openstack subnet create --network selfservice \
  --dns-nameserver DNS_RESOLVER --gateway SELFSERVICE_NETWORK_GATEWAY \
  --subnet-range SELFSERVICE_NETWORK_CIDR selfservice
```
Replace these values based on what you want

**Example**: The self-service network uses 172.16.1.0/24 with a gateway on 172.16.1.1. A DHCP server assigns each instance an IP address from 172.16.1.2 to 172.16.1.254. All instances use 8.8.4.4 as a DNS resolver.
```bash
openstack subnet create --network selfservice \
  --dns-nameserver 8.8.4.4 --gateway 172.16.1.1 \
  --subnet-range 172.16.1.0/24 selfservice
```

Self-service networks connect to provider networks using a virtual router that typically performs bidirectional NAT. Each router contains an interface on at least one self-service network and a gateway on a provider network.

The provider network must include the `router:external` option to enable self-service routers to use it for connectivity to external networks such as the Internet. The `admin` or other privileged user must include this option during network creation or add it later. In this case, the `router:external` option was set by using the `--external` parameter when creating the `provider` network.

- Create the router:
```bash
openstack router create router
```

- Add the self-service network subnet as an interface on the router:
```bash
openstack router set router --external-gateway provider
```

- Set a gateway on the provider network on the router:
```bash
openstack router set router --external-gateway provider
```

Verify operation
- List network namespaces. You should see one `qrouter` namespace and two `qdhcp` namespaces.
```bash
ip netns
```

- List ports on the router to determine the gateway IP address on the provider network:
```bash
openstack port list --router router
```

####  Create flavor
- Create flavor (m1.nano) for Cirror image (testing purposes)
```bash
openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
```

#### Generate a key pair
- Generate a key pair and add a public key (or use your exist key):
```bash
ssh-keygen -q -N ""
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

- Verify addition of the key pair:
```
openstack keypair list
```

#### Add security group rules
By default, the `default` security group applies to all instances and includes firewall rules that deny remote access to instances. For Linux images such as CirrOS, we recommend allowing at least ICMP (ping) and secure shell (SSH).

Add rules to the `default` security group:
- Permit ICMP (ping):
```bash
openstack security group rule create --proto icmp default
```

- Permit secure shell (SSH) access:
```bash
openstack security group rule create --proto tcp --dst-port 22 default
```

#### Launch a Test Instance
To launch an instance, you must at least specify the flavor, image name, network, security group, key, and instance name.
- List available flavors:
```bash
openstack flavor list
```

- List available images:
```bash
openstack image list
```

- List available networks:
```bash
openstack network list
```
This instance uses the `selfservice` self-service network. However, you must reference this network using the ID instead of the name.

- List available security groups:
```bash
openstack security group list
```

- Launch the test instance:
```bash
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=SELFSERVICE_NET_ID --security-group default \
  --key-name mykey selfservice-instance
```
Replace `SELFSERVICE_NET_ID` with the ID of the `selfservice` network.

- Check the status of your instance:
```bash
openstack server list
```
The status changes from `BUILD` to `ACTIVE` when the build process successfully completes.

#### Access the instance using a virtual console
- Obtain a [Virtual Network Computing (VNC)](https://docs.openstack.org/install-guide/common/glossary.html#term-Virtual-Network-Computing-VNC) session URL for your instance and access it from a web browser:
```bash
openstack console url show selfservice-instance
```

- Verify access to the self-service network gateway:
```bash
ping -c 4 172.16.1.1
```

- Verify access to the internet:
```bash
ping -c 4 openstack.org
```