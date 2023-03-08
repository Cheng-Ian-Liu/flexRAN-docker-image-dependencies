# 0. Background

The purpose of this document is to summarize all the host-level and K8s-level configurations we made to the platform to support O-RAN DU workload on-boarding.

# 1. Preparation

## 1.1. Prequisition

EKS-A install completed with Generic Ubuntu OS up and running. Follow this EKS-A documentation: https://anywhere.eks.amazonaws.com/docs/getting-started/production-environment/baremetal-getstarted/ 

## 1.2. BIOS/Firmware Configurations

Server BIOS, Firmware settings are out of scope of this blog, but please follow your server OEM's recommendations to meet the real-time requirements.

# 2. Host OS Configuration

## 2.1. RT kernel install

Follow this article to install the RT kernel for Ubuntu 22.04, beta version: https://ubuntu.com/blog/real-time-ubuntu-released

A Ubuntu Pro subscription is needed in order to install RT kernel patch. To attach your machine to a Pro subscription, please run:

```
pro attach <free token> 
```

To enable the real-time beta kernel, run:

```
pro enable realtime-kernel --beta
```

Then reboot the server, and your kernel version should become RT kernal. Notice that your RT kernel version may be newer given there are continuous newer RT kernel beta releases from Ubuntu RT community

```
root@eksa-du:/home/ec2-user# uname -ar
Linux eksa-du 5.15.0-1025-realtime #28-Ubuntu SMP PREEMPT_RT Fri Oct 28 23:19:16 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

After the server is rebooted, install the following packages that will be used later when setting CPU core frequency
Note: if your RT kernel version is newer than 5.15.0-1025, update the release version below accordingly

```
sudo apt install linux-tools-5.15.0-1025-realtime
sudo apt install linux-cloud-tools-5.15.0-1025-realtime
```

## 2.2. RT kernel configuration

Install TuneD:

```shell
$ apt install tuned
$ ln -s /boot/grub/grub.cfg /etc/grub2.cfg
$ vim /etc/grub.d/00_tuned
```

Add following line to the end of this file

```shell
echo "export tuned_params"
```

Edit /etc/tuned/realtime-variables.conf to add isolated_cores=1-31, 33-63: (in the case of 32pCores/64vCores CPU, exclude 0 and 32 as sibiling vCores for house keeping purpose). You need to change the core ID according to the number of CPU Cores in your server.

```shell
isolated_cores=1-31,33-63
```

Edit /usr/lib/tuned/realtime/tuned.conf to add nohz and rcu related parameters:

```shell
cmdline_realtime=+isolcpus=${managed_irq}${isolated_cores} intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=${isolated_cores} rcu_nocbs=${isolated_cores} rcu_nocb_poll
```

Activate Real-Time Profile:

```shell
$ tuned-adm profile realtime
```

Check tuned_params:

```shell
$ grep tuned_params= /boot/grub/grub.cfg
set tuned_params="skew_tick=1 isolcpus=1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll"
```

The other parameters of this set of best known configuration can be simply added in /etc/default/grub as an example below:


```shell
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt usbcore.autosuspend=-1 selinux=0 enforcing=0 nmi_watchdog=0 crashkernel=auto softlockup_panic=0 audit=0 mce=off hugepagesz=1G hugepages=32 hugepagesz=2M hugepages=0 default_hugepagesz=1G kthread_cpus=0,32 irqaffinity=0,32 skew_tick=1 isolcpus=1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll"
```


Apply the changes by update the grub configuration file.

```shell
$ sudo update-grub
$ sudo reboot
```

Reboot the server, and check the kernel parameter, which should look like below (and your system may have different core configurations):

```shell
root@eksa-du:/home/ec2-user# cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-5.15.0-1025-realtime root=UUID=b18bef28-24eb-4b61-836a-61a75205358f ro intel_iommu=on iommu=pt usbcore.autosuspend=-1 selinux=0 enforcing=0 nmi_watchdog=0 crashkernel=auto softlockup_panic=0 audit=0 mce=off hugepagesz=1G hugepages=32 hugepagesz=2M hugepages=0 default_hugepagesz=1G kthread_cpus=0,32 irqaffinity=0,32 skew_tick=1 isolcpus=1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll autoinstall ds=nocloud-net
```


## 2.3. Configure the CPU Frequency and cstate

Further improve the deterministic and power efficiency by configuring CPU frequency and disable C-State

```shell
$ apt install msr-tools
$ cpupower frequency-set -g performance
```

Set cpu core frequency to 2.5GHz (check with your CPU provider on the ideal CPU frequency to set according to your CPU model)

```shell
$ wrmsr -a 0x199 0x1900
```

Set cpu uncore (i.e. 0x620) to fixed – maximum allowed. Disable c6 and c1e

```shell
$ wrmsr -a 0x620 0x1e1e
$ cpupower idle-set -d 3
$ cpupower idle-set -d 2
```



# 3. Kubernetes Layer Configurations

## 3.1. Kubernetes plugins installation

Obtain the kubeconfig file of the target cluster, and execute the commands below to install additional K8s plugins

Below plugins are required:

### 3.1.1 multus:

Clone Multus CNI repo

```
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git && cd multus-cni
```

Apply Multus daemonset (thick) to your EKS-A cluster

```
kubectl apply -f ./deployments/multus-daemonset-thick.yml
```

Verify that you have Multus pods running

```
kubectl get pods -A | grep -i multus
```

You can validate the multus plugin by spinning up a sample pod with 2nd interface via ipvlan or macvlan CNI.

Change the interface name and IP address accordingly in the macvlan NAD example below, based on your environment

```
$cat macvlan-conf-cheng.yml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eno1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "172.17.1.0/24",
        "rangeStart": "172.17.1.31",
        "rangeEnd": "172.17.1.33",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "172.17.1.1"
      }
    }'    
```
Apply the NAD

```
kubectl apply -f macvlan-conf-cheng.yml

kubectl get net-attach-def
NAME           AGE
macvlan-conf   81m
```

Create a sample pod

```
$ cat multus-macvlan-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multus-macvlan-test-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:  # specification of the pod's contents
  restartPolicy: Never
  containers:
  - name: test-pod-2
    image: "busybox"
    command: ["top"]
    stdin: true
    tty: true
```
```
$ kubectl apply -f multus-macvlan-test-pod.yaml
pod/multus-macvlan-test-pod created

$ kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
multus-macvlan-test-pod   1/1     Running   0          6s
```
Get into the pod and verify there are two interfaces

```
$ kubectl exec -it multus-macvlan-test-pod sh
/ #
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 1E:9A:44:9B:13:97
          inet addr:192.168.0.188  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fe80::1c9a:44ff:fe9b:1397/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:7 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:746 (746.0 B)  TX bytes:796 (796.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

net1      Link encap:Ethernet  HWaddr 12:A8:12:95:F6:A4
          inet addr:172.17.1.33  Bcast:172.17.1.255  Mask:255.255.255.0
          inet6 addr: fe80::10a8:12ff:fe95:f6a4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:147 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:13720 (13.3 KiB)  TX bytes:928 (928.0 B)
```

### 3.1.2 SRIOV (cni and network device plugin):
  
  Follow SRIOV instruction on SRIOV GitHub - <https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin#sr-iov-network-device-plugin>.
  
  - High-level workflow:
  
    1. Install SR-IOV CNI and SR-IOV Network Device Plugin
    2. (For accelerator only) Load device's (Physical function if it is SR-IOV capable) kernel module and bind the driver to the PF
    3. Create required Virtual functions
    4. Bind all VF with right drivers
    5. Create a resource config map
    6. Run SR-IOV Network Device Plugin (as daemonset), or restart it whenever you make changes to the configmap of SR-IOV Network Device Plugin
    

  - SR-IOV CNI plugin install
  
  First install go-1.16
  
  ```
  # on the target EKS-A node
  $ wget https://go.dev/dl/go1.16.linux-amd64.tar.gz
  $ tar -xvf go1.16.linux-amd64.tar.gz -C /usr/local/
  $ export PATH=$PATH:/usr/local/go/bin
  # check by go env
  $ go env
  ```
  
  Download the sriov cni plugin source code and build
  
  ```
  # continue on the target EKS-A node
  $ git clone https://github.com/k8snetworkplumbingwg/sriov-cni.git
  $ cd sriov-cni
  $ git checkout v2.2
  $ mkdir bin
  # download and install golint
  $ go get -u -v golang.org/x/lint/golint
  $ cp ~/go/bin/golint bin/
  $ go env -w GO111MODULE=off
  $ make
  ```
  
  Copy the cni binaries into the CNI folder of each worker node
  
  ```
  $ cd build
  $ cp sriov /opt/cni/bin
  ```
  
  - SRIVO Network Device Plugin git and image pull

  
  ```shell
  # get on the admin machine, where dockerd is running
  $ cd /root
  $ git clone https://github.com/intel/sriov-network-device-plugin
  ```

  - SRIOV VF configuration
  
  Get on the target EKS-A machine
  
  Creating VFs with sysfs

  First select a compatible NIC on which to create VFs and record its name (shown as PF_NAME below).

  To create 8 virtual functions on the NIC port for backhaul/midhaul traffic, run: 
  (Note: do not create the SR-IOV VFs on the Host OAM NIC port, which would cause the loss of OAM connectivity)

  `echo 8 > /sys/class/net/${PF_NAME}/device/sriov_numvfs`
  
  e.g. `root@eksa-du:/home/ec2-user# echo 8 > /sys/class/net/eno2/device/sriov_numvfs`

  To check that the VFs have been successfully created run:

  ```
  root@eksa-du:/home/ec2-user# lspci | grep "Virtual Function"
  18:10.1 Ethernet controller: Intel Corporation X550 Virtual Function
  18:10.3 Ethernet controller: Intel Corporation X550 Virtual Function
  18:10.5 Ethernet controller: Intel Corporation X550 Virtual Function
  18:10.7 Ethernet controller: Intel Corporation X550 Virtual Function
  18:11.1 Ethernet controller: Intel Corporation X550 Virtual Function
  18:11.3 Ethernet controller: Intel Corporation X550 Virtual Function
  18:11.5 Ethernet controller: Intel Corporation X550 Virtual Function
  18:11.7 Ethernet controller: Intel Corporation X550 Virtual Function
  ```

   This method requires the creation of VFs each time the node resets. This can be handled automatically by placing the above command in a script that runs on startup such as /etc/rc.local, or via systemd service
   
   [Optional] If vlan configuration is needed on the VFs, then use similar commands below:
   
   ```
   ip link set enp81s0f0 vf 0 mac 00:11:22:33:00:00 vlan 26 trust on
   ip link set enp81s0f0 vf 1 mac 00:11:22:33:00:10 vlan 26 trust on
   ip link set enp81s0f2 vf 0 mac 00:11:22:33:02:00 vlan 27 trust on
   ```
  
  - DPDK binding 

  Before we apply SRIOV device plugin, we need to install DPDK on the target machine host, and bind VF devices with DPDK drivers.
  
  Theoretically, we can just move the dpdk-devbind.py script over to the target machine, without installing the whole DPDK package on the target machine. But for testing/troubleshooting purpose, let's install the DPDK package as described below.

Install DPDK pre-req packages

```
$ apt install libnuma-dev libhugetlbfs-dev build-essential cmake meson pkgconf python3-pyelftools
```

Download DPDK

```
# get on the target EKS-A machine
$ cd /opt/  
$ wget http://static.dpdk.org/rel/dpdk-21.11.tar.xz  
$ tar xf /opt/dpdk-21.11.tar.xz
```

Build and install DPDK

Note: this step can take 10mins, so run it in a tmux session if possible

```shell
$ cd /opt/dpdk_21.11
$ meson build
$ cd build
$ ninja
$ ninja install
```

Now you can use the usertools (i.e. dpdk-devbind.py) that comes with DPDK package to check device drivers in use, and bind VF drivers
  
  Check current devices and drivers: (below is just an example, and your system output may vary)

  ```
  root@eksa-du:/opt/dpdk-21.11/usertools# dpdk-devbind.py -s
  
  # example output
  
  Network devices using kernel driver
  ===================================
  0000:18:00.0 'Ethernet Controller 10G X550T 1563' if=eno1 drv=ixgbe unused=vfio-pci *Active*
  0000:18:00.1 'Ethernet Controller 10G X550T 1563' if=eno2 drv=ixgbe unused=vfio-pci
  0000:18:10.1 'X550 Virtual Function 1565' if=eno2v0 drv=ixgbevf unused=vfio-pci
  0000:18:10.3 'X550 Virtual Function 1565' if=eno2v1 drv=ixgbevf unused=vfio-pci
  0000:18:10.5 'X550 Virtual Function 1565' if=eno2v2 drv=ixgbevf unused=vfio-pci
  0000:18:10.7 'X550 Virtual Function 1565' if=eno2v3 drv=ixgbevf unused=vfio-pci
  0000:18:11.1 'X550 Virtual Function 1565' if=eno2v4 drv=ixgbevf unused=vfio-pci
  0000:18:11.3 'X550 Virtual Function 1565' if=eno2v5 drv=ixgbevf unused=vfio-pci
  0000:18:11.5 'X550 Virtual Function 1565' if=eno2v6 drv=ixgbevf unused=vfio-pci
  0000:18:11.7 'X550 Virtual Function 1565' if=eno2v7 drv=ixgbevf unused=vfio-pci
  0000:19:00.0 'I350 Gigabit Network Connection 1521' if=enp25s0f0 drv=igb unused=vfio-pci
  0000:19:00.1 'I350 Gigabit Network Connection 1521' if=enp25s0f1 drv=igb unused=vfio-pci
  0000:19:00.2 'I350 Gigabit Network Connection 1521' if=eno3 drv=igb unused=vfio-pci
  0000:19:00.3 'I350 Gigabit Network Connection 1521' if=eno4 drv=igb unused=vfio-pci

  Other Baseband devices
  ======================
  0000:c3:00.0 'Device 0d5c' unused=vfio-pci

  No 'Crypto' devices detected
  ============================

  DMA devices using kernel driver
  ===============================
  0000:00:01.0 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.1 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.2 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.3 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.4 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.5 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.6 'Device 0b00' drv=ioatdma unused=vfio-pci
  0000:00:01.7 'Device 0b00' drv=ioatdma unused=vfio-pci

  No 'Eventdev' devices detected
  ==============================

  No 'Mempool' devices detected
  =============================

  No 'Compress' devices detected
  ==============================

  No 'Misc (rawdev)' devices detected
  ===================================

  No 'Regex' devices detected
  ===========================
  ```
  
  Bind the last four VFs of eno2 to vfio-pci:
  
  `dpdk-devbind.py -b vfio-pci 18:11.1 18:11.3 18:11.5 18:11.7`
  
  Re-run the dpdk-devbind.py -s command should show the VFs are binded with DPDP compatible drivers (i.e. vfio-pci)
  
  Bind the top four VFs of eno2 to ixgbevf (default vf driver, so there should be no change):
  
  ```
  root@eksa-du:/opt/dpdk-21.11/usertools# dpdk-devbind.py -b ixgbevf 18:10.1 18:10.3 18:10.5 18:10.7
  Notice: 0000:18:10.1 already bound to driver ixgbevf, skipping
  Notice: 0000:18:10.3 already bound to driver ixgbevf, skipping
  Notice: 0000:18:10.5 already bound to driver ixgbevf, skipping
  Notice: 0000:18:10.7 already bound to driver ixgbevf, skipping
  ```
  

- Configure FEC and Fronthaul NIC SRIOV (example as below)

Build igb_uio (for Accelerator PF driver binding purpose)

```shell
$ cd /opt
$ git clone http://dpdk.org/git/dpdk-kmods
$ cd dpdk-kmods/linux/igb_uio
$ make
```

Note: the kernel module loading needs to be re-do everytime the server rebooted, therefore, it is recommend to persist these configurations in a service

```shell
$ modprobe vfio-pci
$ modprobe uio
$ insmod /opt/dpdk-kmods/linux/igb_uio/igb_uio.ko
$ lspci | grep acc
c3:00.0 Processing accelerators: Intel Corporation Device 0d5c

# note: the lspci command is how you get the device ID (i.e. 0d5c) and its PCI address (i.e. c3:00.0), which are needed in following configurations 

$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -s | grep 0d5c
0000:c3:00.0 'Device 0d5c' unused=igb_uio,vfio-pci

# bind accelerator PF
# change the pci address according to your environment
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b igb_uio 0000:c3:00.0
$ echo 0 > /sys/bus/pci/devices/0000:c3:00.0/max_vfs
$ echo 2 > /sys/bus/pci/devices/0000:c3:00.0/max_vfs

# bind accelerator VF
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -s

Baseband devices using DPDK-compatible driver
=============================================
0000:c3:00.0 'Device 0d5c' drv=igb_uio unused=vfio-pci

Other Baseband devices
======================
0000:c4:00.0 'Device 0d5d' unused=igb_uio,vfio-pci
0000:c4:00.1 'Device 0d5d' unused=igb_uio,vfio-pci

$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 0000:c4:00.0

# pf-bb-config (build and then config the accelerator)
$ cd /opt
$ git clone https://github.com/intel/pf-bb-config.git
$ cd /opt/pf-bb-config
$ ./build.sh
$ ./pf_bb_config ACC100 -c acc100/acc100_config_vf_5g.cfg
== pf_bb_config Version v22.07-0-g7843e91 ==
Tue Dec  6 04:47:52 2022:INFO:Queue Groups: 4 5GUL, 4 5GDL, 0 4GUL, 0 4GDL
Tue Dec  6 04:47:52 2022:INFO:Configuration in VF mode
Tue Dec  6 04:47:53 2022:INFO: ROM version MM 99ANA5
Tue Dec  6 04:47:54 2022:INFO:DDR Training completed in 1362 ms
Tue Dec  6 04:47:54 2022:INFO:PF ACC100 configuration complete
Tue Dec  6 04:47:54 2022:INFO:ACC100 PF [0000:c3:00.0] configuration complete!


# configure fronthaul NIC
# Note: change your pci address accordingly

$ echo 0 > /sys/bus/pci/devices/0000:4b:00.0/sriov_numvfs
$ echo 4 > /sys/bus/pci/devices/0000:4b:00.0/sriov_numvfs
$ ip link set ens9f0 vf 0 mac 00:11:22:33:00:00
$ ip link set ens9f0 vf 1 mac 00:11:22:33:00:10
$ ip link set ens9f0 vf 2 mac 00:11:22:33:00:20
$ ip link set ens9f0 vf 3 mac 00:11:22:33:00:30
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -s
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 0000:4b:02.0 0000:4b:02.1
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 0000:4b:02.2 0000:4b:02.3
$ echo 0 > /sys/bus/pci/devices/0000:4b:00.1/sriov_numvfs
$ echo 4 > /sys/bus/pci/devices/0000:4b:00.1/sriov_numvfs
$ modprobe vfio-pci
$ ip link set ens9f1 vf 2 mac 00:11:22:33:00:21
$ ip link set ens9f1 vf 3 mac 00:11:22:33:00:31
$ ip link set ens9f1 vf 0 mac 00:11:22:33:00:01
$ ip link set ens9f1 vf 1 mac 00:11:22:33:00:11
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 0000:4b:0a.0 0000:4b:0a.1
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 0000:4b:0a.2 0000:4b:0a.3
```

  Now you are ready to apply SRIOV device plugin configuration and start the dp_daemonset
  
  - SRIOV Device Plugin configuration  
  
On the admin machine, within the gitcloned folder the sriov-network-device-plugin, below is an example to cofigure SRIOV DP configure map.

Note: Modify the device name according to your target machine configuration (use "lspci -nn" and "dpdk-bind.py -s" output in the target machine to find out the device ID information needed in the configMap configuration below).

More configuration examples can be found in sriov-device-plugin github page: https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin 

 ```
 $ cd sriov-network-device-plugin 
 $ cp deployments/configMap.yaml deployments/configMap.yaml.orig
 $ cat <<EOF > deployments/configMap.yaml
 apiVersion: v1
 kind: ConfigMap
 metadata:
   name: sriovdp-config
   namespace: kube-system
 data:
   config.json: |
     {
         "resourceList": [
             {
               "resourceName": "intel_fec_5g",
                 "deviceType": "accelerator",
                 "selectors": {
                     "vendors": ["8086"],
                     "devices": ["0d5d"],
                     "drivers": ["vfio-pci"]
                 }
             },
             {
               "resourceName": "intel_sriov_dpdk",
                 "selectors": {
                     "vendors": ["8086"],
                     "devices": ["1565"],
                     "drivers": ["vfio-pci"],
                     "pfNames": ["eno2"]
                 }
             },
             {
               "resourceName": "intel_sriov_netdevice",
                 "selectors": {
                     "vendors": ["8086"],
                     "devices": ["1565"],
                     "drivers": ["ixgbevf"],
                     "pfNames": ["eno2"]
                 } 
             }
         ]
     }
  EOF
  ```
  
  Here is another example of confgimap where you have multiple NICs, and each NIC needs to be configured with both DPDK and non-DPDK VFs.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [{
                "resourceName": "intel_sriov_netdevice_fh",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["1889"],
                    "drivers": ["iavf"],
                    "pfNames": ["enp81s0f0"]
                }
            },
            {
                "resourceName": "intel_sriov_dpdk_fh",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["1889"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["enp81s0f0"]
                }
            },
            {
                "resourceName": "intel_srio_netdevice_mh",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["1889"],
                    "drivers": ["iavf"],
                    "pfNames": ["enp81s0f2"]
                }
            },
            {
                "resourceName": "intel_sriov_dpdk_mh",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["1889"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["enp81s0f2"]
                }
            },
            {
                "resourceName": "intel_fec_5g",
                "deviceType": "accelerator",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["0d5d"],
                    "drivers": ["vfio-pci"]
                }
            }
        ]
    }
 ```
 
Then after you create the configmap (some version of sriov-dp folder structure maybe slightly change, but the configmap and daemonset files are two key files), you should be able to see all the K8s resources allocatable (e.g. SRIOV DPDK, non-DPDK/net-device, Accelerator, HugePage Mem, etc.)
  
  ```
  $ kubectl create -f deployments/configMap.yaml  
  $ kubectl create -f deployments/k8s-v1.16/sriovdp-daemonset.yaml  
  $ kubectl get node <your-k8s-worker-node-name> -o json | jq '.status.allocatable' 
  {
    "cpu": "62",
    "ephemeral-storage": "885476035962",
    "hugepages-1Gi": "32Gi",
    "hugepages-2Mi": "0",
    "intel.com/intel_fec_5g": "1",
    "intel.com/intel_sriov_dpdk": "4",
    "intel.com/intel_sriov_netdevice": "4",
    "memory": "97850360Ki",
    "pods": "110"
  }
  ```

Note: everytime there is SR-IOV VF configuration changes on the host level, you need to re-apply the sriov-dp configmap.yaml and re-create the sriov-dp daemonset (example below) in order for K8s to pick up the updated resources and show it in the K8s resource.

```
kubectl delete -f deployments/k8s-v1.16/sriovdp-daemonset.yaml
kubectl apply -f deployments/k8s-v1.16/sriovdp-daemonset.yaml
```

  - Test SRIOV Network Device Plugin
  
  On the admin machinve, create two Network Attachment Definitions for sriov-netdevice and sriov-dpdk resources
  
  (1) for sriov-netdevice NAD, modify the IP address/gateway accordingly to your enviornment
  ```
  $ cat <<EOF > sriov-netdevice.yaml
  apiVersion: "k8s.cni.cncf.io/v1"
  kind: NetworkAttachmentDefinition
  metadata:
    name: sriov-netdevice1
    annotations:
      k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_netdevice
  spec:
    config: '{
    "type": "sriov",
    "cniVersion": "0.3.1",
    "name": "sriov-network",
    "ipam": {
      "type": "host-local",
      "subnet": "x.x.x.x/24",
      "routes": [{
        "dst": "0.0.0.0/0"
      }],
      "gateway": "x.x.x.x"
    }
  }'
  EOF
  
  kubectl create -f sriov-netdevice.yaml
  ```
  
  (2) for sriov-dpdk NAD, modify the IP address/gateway accordingly to your enviornment
  ```
  cat <<EOF > sriov-dpdk-crd.yaml
  apiVersion: "k8s.cni.cncf.io/v1"
  kind: NetworkAttachmentDefinition
  metadata:
    name: sriov-dpdk1
    annotations:
      k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_dpdk
  spec:
    config: '{
    "type": "sriov",
    "cniVersion": "0.3.1",
    "name": "sriov-dpdk"
  }'
  EOF
  
  kubectl create -f sriov-dpdk-crd.yaml
  ```
  
  Check the NADs are created
  ```
  $ kubectl get net-attach-def -A
  NAMESPACE   NAME               AGE
  default     sriov-dpdk1        4s
  default     sriov-netdevice1   11s
  ```
  
  Create testpod with SR-IOV resource request
  
  ```
  cat <<EOF > pod-sriov-test.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: sriov-test-pod1
    annotations:
      k8s.v1.cni.cncf.io/networks: sriov-netdevice1, sriov-dpdk1
  spec:
    containers:
    - name: appcntr1
      image: centos/tools
      imagePullPolicy: IfNotPresent
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300000; done;" ]
      resources:
        requests:
          intel.com/intel_sriov_netdevice: '1'
          intel.com/intel_sriov_dpdk: '1'
        limits:
          intel.com/intel_sriov_netdevice: '1'
          intel.com/intel_sriov_dpdk: '1'
  EOF
  
  kubectl create –f pod-sriov-test.yaml
  ```
  
  ### 3.1.3 PTP Plugin
  
  Precision Timing Protocol (PTP) is needed for the DU workload to get more accurate timing information and synchronize with other O-RAN components (e.g. RU). It is ISV vendor's responsibility to provide/install PTP plugin on top of the EKS-A platform. Just note that to avoid the timing source conflicts between PTP and NTP (default on the platform), we need to disable the NTP source on the platform as follows:
  
  ```
  systemctl status chronyd
  systemctl stop chronyd
  systemctl status chronyd
  ```
  
  
## 3.2 K8s Native CPU Manager Configuration

  Since Kubernetes v1.16.1, there is the support of the CPU manager and topology manager. It is important to configure these K8s feature to meet isolated CPU requirement of real-time DU workload.
  
  In this section, steps are provided on how to enable and use these features. You can get more information from the Kubernetes document:
    https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/
    https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/
  These features are controlled by the kubelet* on the worker node. To enable these features, change the kuberlet configuration of worker node and restart kubelet.

  On the target EKS-A node, modify the /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf file and add more parameters to the KUBELET_CONFIG_ARGS as follows: (note: modify the reserved CPU vCore based on your CPU total cores, we want to reserve a sibling vCore pair for OS and Kubelet/Kubeadm to use)
  
  `Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --cpu-manager-policy=static --reserved-cpus=0,32 --topology-manager-policy=best-effort"`
  
  Note to avoid kubelet restart loop when modifying the kubelet parameters for single-node system, remove the following file for the lock on cpu_manager_state
  
  ```
  rm -rf /var/lib/kubelet/cpu_manager_state
  ```
  
  Then restart Kubelet by:

  ```
  $ systemctl daemon-reload
  $ systemctl restart kubelet
  $ systemctl status kubelet
  ```
 
  
  Test a pod with static CPU manager (notice to use the Ubuntu-based pod image, as performance degradation noticed with CentOS based pod image)
  
  On the admin server
  
  ```
  $ cat <<EOF > test-cpu-manager-ubuntu.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: test-cpu-manager-ubuntu
    name: test-cpu-manager-ubuntu
  spec:
    containers:
    - image: ubuntu:latest
      command:
      - sleep
      - inf
      securityContext:
        privileged: true
      name: example
      resources:
        requests:
          cpu: "16"
          memory: "1Gi"
        limits:
          cpu: "16"
          memory: "1Gi"
  EOF
  ```
  
  Login the POD/container and check the taskset:
  ```
  $ kubectl apply -f test-cpu-manager-ubuntu.yaml
  $ kubectl exec test-cpu-manager-ubuntu -it bash
  $ taskset -p 1
  pid 1's current affinity mask: 1fe000001fe
  ```
  This means core 1-8 and 33-40 are allocated to the pod, or you can use the following command to find out the allocated cores
  
  ```
  $ grep Cpus_allowed_list /proc/self/status
  Cpus_allowed_list:      1-8,33-40
  ```
  
  Now you can run another cyclictest from within the pod to validate the real-time performance
  
  ```
  e.g. cyclict test command from within the pod - for Ubuntu based pod
  
  $apt update -y && apt install -y git make gcc libnuma-dev
  $git clone https://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git
  $cd rt-tests/
  $make
  
  #run the following in a tmux session to avoid ssh disconnect
  $grep Cpus_allowed_list /proc/self/status
  # Cpus_allowed_list:	1-8,33-40
  
  $taskset -c 1-8,33-40 ./cyclictest -m -p95 -h 15 -a 2-8,33-40 -t 15 --mainaffinity=1 -D 12h
  
  # /dev/cpu_dma_latency set to 0us
  policy: fifo: loadavg: 0.68 0.90 0.96 1/2314 3555

  T: 0 ( 3541) P:95 I:1000 C:  23197 Min:      2 Act:    3 Avg:    2 Max:       3
  T: 1 ( 3542) P:95 I:1000 C:  23197 Min:      2 Act:    2 Avg:    2 Max:       3
  T: 2 ( 3543) P:95 I:1000 C:  23197 Min:      2 Act:    2 Avg:    2 Max:       3
  T: 3 ( 3544) P:95 I:1000 C:  23198 Min:      2 Act:    3 Avg:    2 Max:       3
  T: 4 ( 3545) P:95 I:1000 C:  23198 Min:      2 Act:    3 Avg:    2 Max:       5
  T: 5 ( 3546) P:95 I:1000 C:  23199 Min:      2 Act:    2 Avg:    2 Max:       3
  T: 6 ( 3547) P:95 I:1000 C:  23200 Min:      2 Act:    3 Avg:    2 Max:       3
  T: 7 ( 3548) P:95 I:1000 C:  23199 Min:      2 Act:    2 Avg:    2 Max:       4
  T: 8 ( 3549) P:95 I:1000 C:  23199 Min:      2 Act:    2 Avg:    2 Max:       4
  T: 9 ( 3550) P:95 I:1000 C:  23199 Min:      2 Act:    3 Avg:    2 Max:       4
  T:10 ( 3551) P:95 I:1000 C:  23199 Min:      2 Act:    3 Avg:    2 Max:       4
  T:11 ( 3552) P:95 I:1000 C:  23199 Min:      2 Act:    3 Avg:    2 Max:       4
  T:12 ( 3553) P:95 I:1000 C:  23199 Min:      2 Act:    2 Avg:    2 Max:       5
  T:13 ( 3554) P:95 I:1000 C:  23199 Min:      2 Act:    3 Avg:    2 Max:       4
  T:14 ( 3555) P:95 I:1000 C:  23199 Min:      2 Act:    2 Avg:    2 Max:       3
  ```
  
# Next Steps

Now the host and CaaS platform is configured and ready for vDU workload on-boarding. You can test it with Intel FlexRAN, which is a sample vDU implementation by Intel (https://github.com/intel/FlexRAN).  Or please bring your own vDU workload on-board and test it. Let us know how it goes!



  

