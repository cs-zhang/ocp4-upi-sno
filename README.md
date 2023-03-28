# Installing single node OpenShift to PowerVM

For current OpenShift release, the `openshift-install` supports to create the special ignition file for SNO installation. This method can be used for any platform.

## Requirements for installing OpenShift on a single node

To do the SNO installation, we need two VMs, one works as bastion and another one as OCP node, the minimum hardware requirements as show as below:

| VM   | vCPU  |  Memory  | Storage |
| :----|-----:|--------:|-------:|
| Bastion | 2   |  8GB    |  50 GB  |
| SNO     | 8   |  16GB   | 120GB   |

The `bastion` is used to setup required services and require to be able to run as root. The SNO node need o have the static IP assigned with internet access.

## Bastion setup
The bastion's `SELINUX` has to be set to `permissive` mode, otherwise ansible playbook will fail, to do it open file `/etc/selinux/config` and set `SELINUX=permissive`.

We will use PXE for SNO installation, that requires the following services to be configured and run:
- DNS -- to define `api`, `api-int` and `*.apps` 
- DHCP -- to enable PXE and assign IP to SNO node
- HTTP -- to provide ignition and RHCOS rootfs image 
- TFTP -- to enable PXE

we will install `dnsmasq` to support DNS, DHCP and PXE, `httpd` for HTTP.

Here are some values used in sample configurations:
- 9.47.87.82 -- SNO node's IP
- 9.47.87.83 -- Bastion's IP
- 9.47.95.254 -- Network route or gateway
- ocp.io  -- SNO's domain name
- sno -- SNO's cluster_id


### dnsmasq setup

Here is a sample configuration file for `/etc/dnsmasq.conf`:
```
#################################
# DNS
##################################
#domain-needed
# don't send bogus requests out on the internets
bogus-priv
# enable IPv6 Route Advertisements
enable-ra
bind-dynamic
no-hosts
#  have your simple hosts expanded to domain
expand-hosts


interface=env32
# set your domain for expand-hosts
domain=sno.ocp.io
local=/sno.ocp.io/
address=/apps.sno.ocp.io/9.47.87.82
server=9.9.9.9

addn-hosts=/etc/dnsmasq.d/addnhosts


##################################
# DHCP
##################################
dhcp-ignore=tag:!known
dhcp-leasefile=/var/lib/dnsmasq/dnsmasq.leases

dhcp-range=9.47.87.82,static

dhcp-option=option:router,9.47.95.254
dhcp-option=option:netmask,255.255.240.0
dhcp-option=option:dns-server,9.47.87.83

dhcp-host=fa:b0:45:27:43:20,sno-82,9.47.87.82,infinite


###############################
# PXE
###############################
enable-tftp
tftp-root=/var/lib/tftpboot
dhcp-boot=boot/grub2/powerpc-ieee1275/core.elf
```
and `/etc/dnsmasq.d/addnhosts` file:
```
9.47.87.82 sno-82 api api-int
```

### PXE setup
To enable PXE for PowerVM, we need to install `grub2` with:
```shell
grub2-mknetdir --net-directory=/var/lib/tftpboot
```

Here is the sample `/var/lib/tftpboot/boot/grub2/grub.cfg`:
```shell
default=0
fallback=1
timeout=1

if [ ${net_default_mac} == fa:b0:45:27:43:20 ]; then
default=0
fallback=1
timeout=1
menuentry "CoreOS (BIOS)" {
   echo "Loading kernel"
   linux "/rhcos/kernel" ip=dhcp rd.neednet=1 ignition.platform.id=metal ignition.firstboot coreos.live.rootfs_url=http://9.47.87.83:8000/install/rootfs.img ignition.config.url=http://9.47.87.83:8000/ignition/sno.ign

   echo "Loading initrd"
   initrd  "/rhcos/initramfs.img"
}
fi
```

Download RHCOS image files from [here](https://mirror.openshift.com/pub/openshift-v4/ppc64le/dependencies/rhcos/4.12/latest/) for PXE:
```shell
export RHCOS_URL=https://mirror.openshift.com/pub/openshift-v4/ppc64le/dependencies/rhcos/4.12/latest/
cd /var/lib/tftpboot/rhcos
wget ${RHCOS_URL}/rhcos-live-kernel-ppc64le -o kernel
wget ${RHCOS_URL}/rhcos-live-initramfs.ppc64le.img -o initramfs.img
cd /var//var/www/html/install/
wget ${RHCOS_URL}/rhcos-live-rootfs.ppc64le.img -o rootfs.img
```

### Create the ignition file
To create the ignition file for SNO, we need to create the `install-config.yaml` file, first we need to create the work directory to hold the file:
```shell
mkdir -p ~/sno-work
cd ~/sno-work
```
Using following sample file to create the required `install-config.yaml`:
```yaml
apiVersion: v1
baseDomain: <domain>
compute:
- name: worker
  replicas: 0 
controlPlane:
  name: master
  replicas: 1 
metadata:
  name: <cluster_id>
networking: 
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 9.47.80.0/20 
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
bootstrapInPlace:
  installationDisk: <device> 
pullSecret: '<pull_secret>' 
sshKey: |
  <ssh_key> 
```
Note: 
- `<domain>`: Set the domain for your cluster, like `ocp.io`.
- `<cluster_id>`: Set the cluster_id for your cluster, like `sno`.
- `<ssh_key>`: Add the public SSH key from the administration host so that you can log in to the cluster after installation. 
- `<pull_secret>`: Copy the pull secret from the Red Hat OpenShift Cluster Manager and add the contents to this configuration setting.
- `<device>`: Provide the device name where the RHCOS will be installed, like `/dev/sda` or `/dev/disk/by-id/scsi-36005076d0281005ef000000000026803`.

Download the `openshift-install`:
```shell
wget wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/clients/ocp/4.12.0/openshift-install-linux-4.12.0.tar.gz
tar xzvf openshift-install-linux-4.12.0.tar.gz
```
Create the ignition file and copy it to http directory:
```shell
./openshift-install --dir=~/sno-work create create single-node-ignition-config
cp ~/sno-work/single-node-ignition-config.ign /var/www/html/ignition/sno.ign
restorecon -vR /var/www/html || true
```

Now `bastion` has all required files and configurations to install SNO.

## SNO installation
There two steps for SNO installation, first the SNO LPAR need to boot up with PXE, then monitor the installation progress.

### Network boot
To boot powerVM with netboot, there are two ways to do it: using SMS interactively to select bootp or using `lpar_netboot`  command on HMC. Reference to HMC doc for how to using SMS.

Here is the `lpar_netboot` command:
```shell
lpar_netboot -i -D -f -t ent -m <sno_mac> -s auto -d auto -S <server_ip> -C <sno_ip> -G <gateway> <lpar_name> default_profile <cec_name>
```
Note:
- <sno_mac>: MAC address of SNO
- <sno_ip>:  IP address of SNO
- <server_ip>: IP address of bastion (PXE server)
- <gateway>: Network's gateway IP
- <lpar_name>: SNO lpar name in HMC
- <cec_name>: System name where the sno_lpar resident at

### Monitoring the progress
After the SNO lpar boot up with PXE, we can use `openshift-install` to monitor the progress of installation.

```shell
# first need to wait for bootstrap complete
./openshift-install wait-for bootstrap-complete
# after it return successfully, using following cmd to wait for completed
./openshift-install wait-for install-complete
```
Also we can use `oc` to check installation status:
```shell
export KUBECONFIG=~/sno-work/auth/kubeconfig
# check SNO node status
oc get nodes
# check installation status
oc get clusterversion
# check cluster operators
oc get co
# check pod status
oc get pod -A
```

