# Installation Openshift 4.x



## CHAPTER 1. Requirement for POC Openshift 4.x

> **WARNING :** Please make sure the spec is same and size memory is very important.. Don't set Memory under 16 GB for Cluster Openshift 4.x

1. Bastion = 1 (vCPU = 4 , Memory = 4 GB, Harddisk = 50 GB)
2. Bootstrap = 1 (vCPU = 4, Memory = 16 GB, Harddisk = 100 GB)
3. Master = 1 (vCPU = 4, Memory = 16 GB, Harddisk = 100 GB)
4. Worker = 1 (vCPU = 4, Memory = 16 GB, Harddisk 1 = 100 GB)

==============================================================================

Network Requirement :

1. Master, Worker and Bootstrap : 1 NIC ( Host-only ), Please make sure not DHCP Server in this Connection.
- Host-only ( 10.10.10.0/24 ) - PLEASE CLEAR NOT DHCP SERVER IN THIS NETWORK

 
2. Bastion : 2 NIC (1 NIC for External, 1 NIC for Host-only)
* bastion for DNS Server, HAProxy Load Balancer, DNSMasq, TFTP Server and Router

  - External ( 192.168.1.0/24 ) - INTERNET ACCESS
  - Host-only ( 10.10.10.0/24 ) - PLEASE CLEAR NOT DHCP SERVER IN THIS NETWORK
  

Bastion : 10.10.10.1 /24

Bootstrap : 10.10.10.10 /24

Master : 10.10.10.11 /24

Worker : 10.10.10.12 - 10.10.10.17 /24

==============================================================================

 ![OCP_4x_bootstrap.png](https://github.com/openshift-telco/openshift4x-poc/blob/master/static/OCP_4x_bootstrap.png?raw=true) 

​                                            



## CHAPTER 2. Set DNS and PTR

Setting A Record in bind :

```
root@bastion# git clone https://github.com/ProjectOprec/instalasi-openshift

root@bastion# yum -y install bind bind-utils

root@bastion# setenforce 0

root@bastion# cp openshift4.x/dns/named.conf /etc/named.conf
```

Setting for PTR :

```
root@bastion# cp openshift4.x/dns/10.10.10.in-addr.arpa /var/named/
```

Setting for A and SRV Record :

```
root@bastion# cp openshift4.x/dns/ocp4poc.example.com /var/named/
```

Please restart the service :

```
root@bastion# systemctl restart named

root@bastion# systemctl enable named
```

Please make sure DNS can reply your Query, Detail IP you can check /var/named/ocp4poc.example.com :

```
root@bastion# dig @localhost -t srv _etcd-server-ssl._tcp.ocp4poc.example.com.

root@bastion# dig @localhost bootstrap.ocp4poc.example.com

root@bastion# dig -x 10.10.10.10
```

Add line nameserver to localhost

```
root@bastion# cat /etc/resolv.conf

nameserver 127.0.0.1
nameserver 8.8.8.8
```

NOTE: Update `/var/named/ocp4poc.example.com and 10.10.10.in-addr.arpa` to match environment



### CHAPTER 3. Set HAProxy for Load Balancer

```
root@bastion# yum -y install haproxy

root@bastion# cp openshift4.x/haproxy/haproxy.cfg /etc/haproxy/
```

Please edit IP Address for Bootstrap , master and worker.. Please double check in Your DNS Setting

You can inspect **/etc/haproxy/haproxy.cfg** : 

```
Port 6443 : bootstrap and master ( API)
Port 22623 : bootstrap and master ( machine config)
Port 80 : worker ( ingress http)
Port 443 : worker ( ingress https)
Port 9000 : GUI for HAProxy
```



## CHAPTER 4. Preparation Installation Core OS

Please download in cloud.redhat.com and choose Red Hat Openshift Cluster Manager :

![image-20191121164418499](https://raw.githubusercontent.com/h4ckersmooth88/openshift4.x/master/image-20191121164418499.png)

Choose Bare Metal :

![image-20191121164524796](https://raw.githubusercontent.com/h4ckersmooth88/openshift4.x/master/image-20191121164524796.png)



![image-20191121164655226](https://raw.githubusercontent.com/h4ckersmooth88/openshift4.x/master/image-20191121164655226.png)

Download RHCOS :

https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.x/latest/

Download Command-Line Interface :

https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/



## CHAPTER 5. Set Web Server

```
root@bastion# yum -y install httpd
```

change the port Listen to **Port 8000**

```
root@bastion# cp openshift4.x/httpd/httpd.conf /etc/httpd/conf/httpd.conf

root@bastion# mkdir -p /var/www/html/metal/
```



Please check location your download installer RHCOS :

```
root@bastion# cp rhcos-4.x.0-x86_64-metal-bios.raw.gz /var/www/html/metal
```

Start the services :

```
root@bastion# systemctl start httpd
root@bastion# systemctl enable httpd
```



## CHAPTER 6. Set Tftpd Server and DNSMasq

```
root@bastion# yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

root@bastion# yum -y install tftp-server dnsmasq syslinux-tftpboot tree python36 jq oniguruma
```



Disable DNS in DNSmasq by setting `port=0` 

```
vi /etc/dnsmasq.conf
...
port=0
...
```

```
root@bastion# cp openshift.4.x/dnsmasq/dnsmasq-pxe.conf /etc/dnsmasq.d/dnsmasq-pxe.conf
```

NOTE: Update `/etc/dnsmasq.d/dnsmasq-pxe.conf` to match environment

```
root@bastion# mkdir -pv /var/lib/tftpboot/pxelinux.cfg/

root@bastion# cp openshift4.x/pxelinux.cfg/default /var/lib/tftpboot/pxelinux.cfg/default
```

Copy Installer Image and Kernel ( Please make sure your source installer CoreOS)

```
root@bastion# mkdir -p /var/lib/tftpboot/rhcos/

root@bastion# cp rhcos-4.x.0-x86_64-installer-initramfs.img /var/lib/tftpboot/rhcos/rhcos-initramfs.img

root@bastion# cp rhcos-4.x.0-x86_64-installer-kernel /var/lib/tftpboot/rhcos/rhcos-kernel
```

You can inspect the file /var/lib/tftpboot/pxelinux.cfg/default

Please check make sure your environment :

1. coreos.inst.install_dev=sda or vda 
2. coreos.inst.ignition_url and coreos.inst.image_url

```
root@bastion# systemctl restart tftp

root@bastion# systemctl enable fftp

root@bastion# systemctl restart dnsmasq

root@bastion# systemctl enable dnsmasq
```



## CHAPTER 7. Prepare Router and Firewall

```
root@bastion# chmod 777 openshift4.x/patch/firewall.sh
```

Please edit interface NIC with your environment firewall.sh

```
root@bastion# ./firewall.sh
```



## CHAPTER 8. Prepare Ignition File

Extract tools openshift

```
root@bastion# tar -xvf openshift-client-linux-4.x.2.tar.gz

root@bastion# tar -xvf openshift-install-linux-4.x.2.tar.gz

root@bastion# mv oc kubectl openshift-install /usr/bin/
```

Create the installation manifests : ( Please make sure execute command openshift-install in directory /root/ocp4poc/)

NOTE: please make sure Work Directory to create manifest and ignition config in /root/ocp4poc

```
root@bastion# mkdir /root/ocp4poc/
root@bastion# cd /root/ocp4poc/
```

Please insert the credential : ( Sample you can check in openshift4.x/patch/install-config-UPDATETHIS.yaml)

```
root@bastion# cat install-config.yaml
```



```
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 1
metadata:
  name: ocp4poc
networking:
  clusterNetworks:
  - cidr: 10.254.0.0/16
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths": ...}' <<- INSERT YOUR Secret in web cloud.redhat.com
sshKey: 'ssh-ed25519 AAAA...' <<-- insert ssh Pub
EOF
```

```
root@bastion# cp install-config.yaml /root/ocp4poc/
```

Please make sure install-config.yaml in /root/ocp4poc/ directory

```
root@bastion# openshift-install create manifests
```

Prevent Pods from being scheduled on the control plane machines

```
root@bastion# sed -i 's/mastersSchedulable: true/mastersSchedulable: false/g' manifests/cluster-scheduler-02-config.yml
```

Copy Patching Network Config

```
root@bastion# cp openshift4.x/patch/ 10-*.yaml /root/ocp4poc/openshift/
```

Generate ignition configs

```
root@bastion# openshift-install create ignition-configs
```

Copy the Ignition to Web Server

```
root@bastion# cd /root/ocp4poc/

root@bastion# cp *.ign /var/www/html/
```



For now you can Booting Bootstrap and Also Master ONLY with PXE Boot, please don't start Worker Node :

 ![Bootstrap Menu](https://github.com/openshift-telco/openshift4x-poc/raw/master/static/pxe-boot-bootstrap.png) 



Monitoring Bootstrap

```
root@bastion# openshift-install wait-for bootstrap-complete --log-level debug
```

You can investigation with Bootstrap node

```
root@bastion# ssh core@bootstrap.ocp4poc.example.com

core@bootstrap$ journalctl
```

if success :

```
DEBUG OpenShift Installer v4.x.1
DEBUG Built from commit e349157f325dba2d06666987603da39965be5319
INFO Waiting up to 30m0s for the Kubernetes API at https://api.ocp4poc.example.com:6443...
INFO API v1.14.6+868bc38 up
INFO Waiting up to 30m0s for bootstrapping to complete...
DEBUG Bootstrap status: complete
INFO It is now safe to remove the bootstrap resources

```

if bootstrap resources is done, so please shutdown the VM and start all worker nodes



## CHAPTER 9. Monitoring Installation Master and Worker Openshift 4.x

Login to the cluster :

```
root@bastion# export KUBECONFIG=/root/ocp4poc/auth/kubeconfig
```

Set Your Registry to Ephemeral

```
root@bastion# oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

Monitor Request Certificate from your machine

```
root@bastion# watch oc get csr
root@bastion#  oc get csr --no-headers | awk '{print $1}' | xargs oc adm certificate approve
```

You can monitor the progress installation :

```
root@bastion# oc get co

NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.x.2     True        False         True       
cloud-credential                           4.x.2     True        False         False     
cluster-autoscaler                         4.x.2     True        False         False     
console                                    4.x.2     True        False         True       
dns                                        4.x.2     False       True          True      
image-registry                             4.x.2     False       True          False     
ingress                                    4.x.2     False       True          False     
insights                                   4.x.2     True        False         True      
kube-apiserver                             4.x.2     True        True          True       
kube-controller-manager                    4.x.2     True        False         True       
kube-scheduler                             4.x.2     True        False         True       
machine-api                                4.x.2     True        False         False     
machine-config                             4.x.2     False       False         True       
marketplace                                4.x.2     True        False         False     
monitoring                                 4.x.2     False       True          True       
network                                    4.x.2     True        True          False     
node-tuning                                4.x.2     False       False         True       
openshift-apiserver                        4.x.2     False       False         False     
openshift-controller-manager               4.x.2     False       False         False     
openshift-samples                          4.x.2     True        False         False     
operator-lifecycle-manager                 4.x.2     True        False         False     
operator-lifecycle-manager-catalog         4.x.2     True        False         False     
operator-lifecycle-manager-packageserver   4.x.2     False       True          False     
service-ca                                 4.x.2     True        True          False     
service-catalog-apiserver                  4.x.2     True        False         False     
service-catalog-controller-manager         4.x.2     True        False         False     
storage                                    4.x.2     True        False         False     
```

Check the password and Web console :

```
root@bastion# openshift-install wait-for install-complete 
```
