# Installation of SBP for Testing Storage Driver  

### Pre-config (Ubuntu 16.04) 
All the installation work is tested on Ubuntu 16.04, please make sure you have installed the right one. Also root user is REQUIRED before the installation work starts. 

### Install Dependencies

```cassandraql
apt-get update && apt-get install -y git make curl wget libltdl7 libseccomp2 libffi-dev gawk
```

### Install Docker

```cassandraql
wget https://download.docker.com/linux/ubuntu/dists/xenial/pool/stable/amd64/docker-ce_18.06.1~ce~3-0~ubuntu_amd64.deb
dpkg -i docker-ce_18.06.1~ce~3-0~ubuntu_amd64.deb
```

### Install docker-compose

```cassandraql
curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### Install Golang

```cassandraql
wget https://storage.googleapis.com/golang/go1.13.1.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.13.1.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile
echo 'export GOPATH=$HOME/gopath' >> /etc/profile
source /etc/profile
```

### To setup environment
Run following steps
```cassandraql
vi sodafoundation.sh
# Set global configuration
cat <<EOF  >/etc/opensds.conf  
[osdsapiserver] 
api_endpoint = 127.0.0.1:50040 
auth_strategy = noauth 
# If https is enabled, the default value of cert file 
# is /opt/opensds-security/sodafoundation/api-cert.pem, 
# and key file is /opt/opensds-security/sodafoundation/api-key.pem 
https_enabled = False 
beego_https_cert_file = 
beego_https_key_file = 

[osdslet] 
api_endpoint = 127.0.0.1:50049 

[osdsdock] 
api_endpoint = 127.0.0.1:50050 
# Specify which backends should be enabled, sample,ceph,cinder,lvm,nfs,netapp-eseries-san and so on. 
enabled_backends = netapp-eseries-san 
[database] 
endpoint = 127.0.0.1:2379, 127.0.0.1:2380 
driver = etcd 
EOF 

# Connecting NetApp-Eseries-Storage  

cat <<EOF >/root/netapp_eseries_storage.yml 
backendOptions: 
  Username: "admin" 
  Password: "password" 
  DriverName: "netapp-eseries-san" 
  HostDataIP: "127.0.0.1" 
  Size: 100 
  HostType: "" 
  AccessGroup: "" 
  Version: 1 
pool: 
  eseries-pool: 
    storageType: block 
    availabilityZone: default 
    multiAttach: true 
    extras: 
      dataStorage: 
        provisioningPolicy: Thin 
        compression: false 
        deduplication: false 
      ioConnectivity: 
        accessProtocol: iscsi 
EOF 

#Docker-Compose-File 

cat <<EOF >/root/docker-compose.yml 
version: '2' 
services: 
  osdsapiserver: 
    image: 'opensdsio/opensds-apiserver:latest' 
    tty: true 
    network_mode: "host" 
    volumes: 
    - /etc/opensds:/etc/opensds 
    depends_on: 
    - osdsdb 
    - osdsauthchecker 
    restart: on-failure 
    command: /bin/bash -c '/usr/bin/osdsapiserver -logtostderr' 
  osdslet: 
    image: 'opensdsio/opensds-controller:latest' 
    tty: true 
    network_mode: "host" 
    volumes: 
    - /etc/opensds:/etc/opensds 
    depends_on: 
    - osdsdb 
    restart: on-failure 
    command: /bin/bash -c '/usr/bin/osdslet -logtostderr' 
  osdsdock: 
    image: 'gamma60/opensds-dock:latest' 
    tty: true 
    privileged: true 
    volumes: 
    - /etc/opensds:/etc/opensds 
    - /etc/ceph:/etc/ceph 
    - /etc/tgt:/etc/tgt:shared 
    - /run/:/run/:shared 
    - /dev/:/dev/ 
    - /etc/localtime:/etc/localtime:ro 
    - /lib/modules:/lib/modules:ro 
    depends_on: 
    - osdsdb 
    network_mode: "host" 
    restart: on-failure 
    command: /bin/bash -c '/usr/sbin/tgtd && /usr/bin/osdsdock -logtostderr' 
  osdsdb: 
    image: 'quay.io/coreos/etcd:latest' 
    tty: true 
    network_mode: "host" 
    volumes: 
    - /usr/share/ca-certificates/:/etc/ssl/certs 
    restart: on-failure 
  osdsauthchecker: 
    image: 'opensdsio/opensds-authchecker:latest' 
    tty: true 
    network_mode: "host" 
    privileged: true 
    restart: on-failure 
  osdsdashboard: 
    image: 'opensdsio/dashboard:latest' 
    tty: true 
    network_mode: "host" 
    restart: on-failure 
    environment: 
    - OPENSDS_AUTH_URL=http://127.0.0.1/identity 
    - OPENSDS_HOTPOT_URL=http://127.0.0.1:50040 
    - OPENSDS_GELATO_URL=http://127.0.0.1:8089 
```
Run docker-compose.yml file 
```
docker-compose up -d  
```

### Testing Steps
```cassandraql
wget https://github.com/sodafoundation/api/releases/download/v0.12.0/soda-api-v0.12.0-linux-amd64.tar.gz
tar zxvf soda-api-v0.12.0-linux-amd64.tar.gz
cp soda-api-v0.12.0-linux-amd64/bin/* /usr/local/bin
chmod 755 /usr/local/bin/osdsctl

export OPENSDS_ENDPOINT=http://{{ apiserver_cluster_ip }}:50040
export OPENSDS_AUTH_STRATEGY=noauth
```

### Check status & Logs:
```cassandraql
docker ps -a
docker logs { container ID }
```

#### Check Pool List
```
osdsctl pool list
```

#### Create default profile
```
osdsctl profile create '{"name": "default", "description": "default policy", "storageType": "block"}'
```


#### Create Volume
```
#### Create Volume
```
osdsctl volume create 1 --name=test-001

# List Volumes
osdsctl volume list

# This will give list of volumes created using opensds
