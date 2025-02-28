# 0. Background

The purpose of this document is to summarize all the host-level and K8s-level configurations we made to the platform to support Intel FlexRAN on-boarding.
The document is modified based on Intel Open Source FlexRAN installation guide (https://github.com/intel/flexRAN-docker-image-dependencies), and added specific changes tailored for our environment.

# 1. FlexRAN™ software reference stack

Intel provides a vRAN reference architecture in the form of the FlexRAN™ software reference stack,
which demonstrates how to optimize VDU software implementations using Intel® C++ class libraries,
leveraging the Intel® Advanced Vector Extensions 512 (Intel® AVX-512) instruction set.
The multi-threaded design allows a single VDU software implementation to scale to meet the requirements of multiple deployment scenarios,
scaling from single small cells deployments, optimized D-RAN deployments or servicing large number of 5G cells in C-RAN pooled deployments.
As a SW implementation, it is also capable of supporting LTE, 5G narrow band and 5G massive MIMO deployments all from the same SW stack using the O-RAN 7.2x split.
The FlexRAN™ software reference solution framework by Intel is shown in below diagram:  

![image](https://user-images.githubusercontent.com/94888960/199504629-afdf2518-f328-403d-8155-c38364d1d593.png)

# 2. FlexRAN™ docker image

Since 2022, intel FlexRAN team is publishing docker image to docker hub. The purpose is to make more and more potential users can easily enter the door and play the game.
The docker image include only binaries, runtime dependency libraries, configure files and several typical cases. If downloader had already been a NDA customer of Intel,
They can get corresponding source code, more test cases and supports from Intel FlexRAN team.
If you are new entry users and just want to do a quick try, please follow below guides. if you have further intention, please contact Intel FlexRAN Marketing team.  

# 3. User Guide

## 3.1. HW list

|   Category   |                                    Icelake – SP (Intel Reference)                                  |     Icelake – SP ( Lab Testing)   |
| :----------: | :------------------------------------------------------------------------------------------------: | :-------------------------------: |
|    Board     |                                  Intel Server Board M50CYP2SBSTD                                   |   Dell XR11/Supermicro Servers    |
|     CPU      |                              1x Intel® Xeon® Gold 6338N CPU @2.20 GHz                              |              Same CPU             |
|    Memory    |                                   8x16GB DDR4 3200 MHz (Samsung)                                   |           128GB RAM (SMC)         | 
|   Storage    |                                     960 Gb SSD M.2 SATA 6Gb/s                                      |          960 GB NVMe (SMC)        |
|   Chassis    |                                   2 U Rackmount Server Enclosure                                   |   Dell XR11/Supermicro Servers    |
|     NIC1     |                             1x Fortville NIC X722 Base-T(LoM to CPU-0)                             |   Intel I350 (SMC)/               |
|     NIC2     | 1× Fortville 40 Gbe Ethernet PCIe XL710-QDA2 Dual Port QSFP+<br>(PCIe Add-in-card direct to CPU-0) |   Intel X550T (SMC)/              |
| Baseband dev |      Mount Bryce Card (acc100) to CPU-0,<br> Optional, use software mode only if not present       |   Mount Bryce Card (acc100)       |

## 3.2. SW list

|  Category   |        Components        |           Details (FlexRAN Reference)           |            Details (Lab Testing)                |
| :---------: | :----------------------: | :---------------------------------------------: | :---------------------------------------------: |
|  Firmware   |           IFWI           |    Includes BIOS, BMC, ME as well as FRUSDR.    |  Dell/Supermicro BIOS/BMC settings (to be add)  |          
|             |     Fortville XL710      |            8.20 0x8000a051 1.2879.0             |                  To be added                    |
|     OS      |       Ubuntu 22.04       | Ubuntu Server 22.04 Realtime kernel 5.15.0-1009 |Ubuntu Server 22.04 Realtime kernel 5.15.0-1025  |
|   Drivers   |   i40e for x700 series   |      Use the version comes with rt-5.15.0       |                  To be added                    |
| Cloudnative |        kubernetes        |                     1.22.1                      |            v1.23.13-eks-6022eca (beta)          |
|             |    Container runtime     |                  Docker 0.19.0                  |           containerd://1.5.9-0ubuntu3           |
|   FlexRAN   |  FlexRAN 22.07 pacakge   |                      22.07                      |                     same                        |
|  Toolchain  | Intel oneAPI Base Tookit |                  2022.1.2.146                   |                     same                        |
|    DPDK     |       DPDK release       |                      22.11                      |                     same                        |  


## 3.3. Prequisition

EKS-A install completed with Generic Ubuntu OS up and running. Follow this EKS-A documentation: https://anywhere.eks.amazonaws.com/docs/getting-started/production-environment/baremetal-getstarted/ 

Server BIOS settings follow server OEM and Intel's recommendation (Doc: 640685, Rev 1.4)

NIC driver and firmware are updated according to Intel's recommendation: Intel FlexRAN Document Number: 611268-13.0

(1) For E810 NIC (COTS version), here are target versions to upgrade/downgrade to:
ICE Driver Version: 1.8.8. (NOTE: do the driver update/install after RT kernel enablement, since it is kernel dependent)
https://www.intel.com/content/www/us/en/download/19630/729615/intel-network-adapter-driver-for-e810-series-devices-under-linux.html

Note: NIC driver and firmware update would cause loss of connection, therefore, need to perform from BMC console
Also, the ice driver and firmware update requires a server reboot to resume the networking

```
#from the console:
modinfo ice
cd PROCGB/Linux/ice-1.8.8
make install
modinfo ice
rmmod irdma
rmmod ice
modprobe ice
ethtool -i <NIC_name>
```

NVM Firmware Version: 3.20
https://www.intel.com/content/www/us/en/download/19626/727313/non-volatile-memory-nvm-update-utility-for-intel-ethernet-network-adapters-e810-series-linux.html

(2) For E810 NIC (Server OEM version), need to get the corresponding firmware version from the Server OEM. Below is an example for Dell:
https://www.dell.com/support/home/en-us/drivers/driversdetails?driverid=25ffj#:~:text=Linux%20Update%20Package%20Installation%3A

### 3.3.0. RT kernel install

Follow this article to install the RT kernel for Ubuntu 22.04, beta version: https://ubuntu.com/blog/real-time-ubuntu-released

To attach your personal machine to a Pro subscription, please run:

```
pro attach <free token> 
```

To enable the real-time beta kernel, run: (note: stop here and refer to the "specify a particular RT kernel version" section below, if you need a specific version of RT kernel. Otherwise, it would enable the latest RT kernel version automatically)

```
pro enable realtime-kernel
```

Then reboot the server, and your kernel version should become RT kernal. Notice that your RT kernel version may be newer given there are continuous newer RT kernel beta releases from Ubuntu

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

Note: below are some usefull commands to specify a particular RT kernel version:

```
# enable the RT kernel repo access via Ubuntu Pro, but does not enable RT kernel yet
pro enable realtime-kernel --access-only

# The above command will only enable the repositories. After that, you can install the specific version you wish with:
apt install linux-image-5.15.0-1032-realtime

# also, do not forget to install kernerl header file and other kernel related packages via: (the kernel header file will be particularly needed when you need to manually build the ice driver for the new kernel)
apt install linux-headers-5.15.0-1032-realtime linux-modules-5.15.0-1032-realtime linux-modules-extra-5.15.0-1032-realtime

# check all available kernel versions
sudo apt search linux-image | grep realtime/jammy | grep -v unsigned 


# check current kernel in use
sudo apt update && sudo apt-cache policy linux-image-realtime


# (Optional) update to the latest RT kernel version
sudo apt install linux-image-realtime


# some additional info if you have to enable grub menu at boot time to select different kernel to boo

# Menu will appear if you press and hold Shift during loading Grub, if you boot using BIOS. When your system boots using UEFI, press Esc.

# For permanent change you'll need to edit your /etc/default/grub file:

# Place a # symbol at the start of line GRUB_HIDDEN_TIMEOUT=0 to comment it out. If that line doesn't exist, then you can comment out this line  instead: # GRUB_TIMEOUT_STYLE=hidden, and then change GRUB_TIMEOUT=0 to GRUB_TIMEOUT=5, for instance, to give the grub menu a 5 second timeout before it automatically logs you in. Example below

#GRUB_DEFAULT=0
#GRUB_DEFAULT=saved
#GRUB_SAVEDEFAULT=true
GRUB_DEFAULT="1>2"
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=5


# Save changes and run sudo update-grub to apply changes.

# After selecting the right kernel to boot, you can change the default grub entry you want during boot. If you run:

sudo awk -F\' '/menuentry / {print $2}' /boot/grub/grub.cfg

You'll see all entries:

"0" Ubuntu
"1>0" Ubuntu, with Linux 5.15.0-1037-realtime
"1>1" Ubuntu, with Linux 5.15.0-1037-realtime (recovery mode)
"1>2" Ubuntu, with Linux 5.15.0-1032-realtime
"1>3" Ubuntu, with Linux 5.15.0-1032-realtime (recovery mode)

I've added in front of each line the option number for that option. So, in my example, if I want to boot 1032, I'll edit /etc/default/grub and change the option:

sudo nano /etc/default/grub

And change the line with GRUB_DEFAULT to:

GRUB_DEFAULT="1>2"

After that, you need to run:

sudo update-grub

And the next boot will come with that default option selected.

```





### 3.3.1. RT kernel configuration

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

Edit /etc/tuned/realtime-variables.conf to add isolated_cores=1-31, 33-63: (in the case of 32pCores/64vCores CPU, exclude 0 and 32 as sibiling vCores for house keeping purpose), and uncomment isolate_managed_irq=Y

```shell
isolated_cores=1-31,33-63
...
isolated_managed_irq=Y

```

Edit /usr/lib/tuned/realtime/tuned.conf to add nohz and rcu related parameters under the [bootloader] section, and also add the [cpu] section for enabling C6 state per Intel's latest Dec. 2022 doc recommendation



```shell
[bootloader]
cmdline_realtime=+isolcpus=${managed_irq}${isolated_cores} intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=${isolated_cores} rcu_nocbs=${isolated_cores} rcu_nocb_poll

[irqbalance]
banned_cpus=${isolated_cores}

[script]
script = ${i:PROFILE_DIR}/script.sh

[scheduler]
isolated_cores=${isolated_cores}

[rtentsk]

#Enable C6 State
[cpu]
force_latency=-1
```

Activate Real-Time Profile:

```shell
$ tuned-adm profile realtime
```

Check tuned_params:

```shell
$ grep tuned_params= /boot/grub/grub.cfg
set tuned_params="skew_tick=1 isolcpus=managed_irq,domain,1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll"
```

The other parameters of this set of best known configuration can be simply added in /etc/default/grub as below:

Note: in the 5G Anywhere lab system (XR-11), added vfio related config as per Intel's latest Dec. 2022 doc recommendation, also added the tuned_params at the end also, since seems it did not auto-load into grub config

```shell   
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt vfio_pci.enable_sriov=1 vfio_pci.disable_idle_d3=1 usbcore.autosuspend=-1 selinux=0 enforcing=0 nmi_watchdog=0 crashkernel=auto softlockup_panic=0 audit=0 mce=off hugepagesz=1G hugepages=32 hugepagesz=2M hugepages=0 default_hugepagesz=1G kthread_cpus=0,32 irqaffinity=0,32 skew_tick=1 isolcpus=managed_irq,domain,1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll "
```

Note: in the SMC lab system

```shell
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt usbcore.autosuspend=-1 selinux=0 enforcing=0 nmi_watchdog=0 crashkernel=auto softlockup_panic=0 audit=0 mce=off hugepagesz=1G hugepages=32 hugepagesz=2M hugepages=0 default_hugepagesz=1G kthread_cpus=0,32 irqaffinity=0,32 skew_tick=1 isolcpus=1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll"
```


Apply the changes by update the grub configuration file.

```shell
$ sudo update-grub
$ sudo reboot
```

Reboot the server, and check the kernel parameter, which should look like:



Note: in the 5G Anywhere lab system (more parameters due to Intel's latest recommendations)

```shell
root@eksa-du:/home/ec2-user# cat /proc/cmdline 
BOOT_IMAGE=/boot/vmlinuz-5.15.0-1032-realtime root=UUID=b18bef28-24eb-4b61-836a-61a75205358f ro intel_iommu=on iommu=pt vfio_pci.enable_sriov=1 vfio_pci.disable_idle_d3=1 usbcore.autosuspend=-1 selinux=0 enforcing=0 nmi_watchdog=0 crashkernel=auto softlockup_panic=0 audit=0 mce=off hugepagesz=1G hugepages=32 hugepagesz=2M hugepages=0 default_hugepagesz=1G kthread_cpus=0,32 irqaffinity=0,32 skew_tick=1 isolcpus=managed_irq,domain,1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll autoinstall ds=nocloud-net
```

Note: in the SMC lab system

```shell
root@eksa-du:/home/ec2-user# cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-5.15.0-1025-realtime root=UUID=b18bef28-24eb-4b61-836a-61a75205358f ro intel_iommu=on iommu=pt usbcore.autosuspend=-1 selinux=0 enforcing=0 nmi_watchdog=0 crashkernel=auto softlockup_panic=0 audit=0 mce=off hugepagesz=1G hugepages=32 hugepagesz=2M hugepages=0 default_hugepagesz=1G kthread_cpus=0,32 irqaffinity=0,32 skew_tick=1 isolcpus=1-31,33-63 intel_pstate=disable nosoftlockup tsc=nowatchdog nohz=on nohz_full=1-31,33-63 rcu_nocbs=1-31,33-63 rcu_nocb_poll autoinstall ds=nocloud-net
```


### 3.3.2. Configure the CPU Frequency and cstate

Further improve the deterministic and power efficiency

```shell
$ apt install msr-tools
$ cpupower frequency-set -g performance
```

Set cpu core frequency to 2.5GHz (confirmed with Intel that 2.5GHz is okey for Intel(R) Xeon(R) Gold 6338N CPU @ 2.20GHz CPU)

```shell
cpupower frequency-set -u 2500000
```


### 3.3.3. Kubernetes installation

This section is removed, as it is handled by EKS-A installation

### 3.3.4. Kubernetes plugins installation

Obtain the kubeconfig file of the target cluster, and execute the commands below to install additional K8s plugins

Below plugins are required:

- multus:

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

- SRIOV (cni and network device plugin):
  
  Follow SRIOV instruction on SRIOV GitHub - <https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin#sr-iov-network-device-plugin>.
  
  Workflow:
    1. Install SR-IOV CNI and SR-IOV Network Device Plugin
    2. (For accelerator only) Load device's (Physical function if it is SR-IOV capable) kernel module and bind the driver to the PF
    3. Create required Virtual functions
    4. Bind all VF with right drivers
    5. Create a resource config map
    6. Run SR-IOV Network Device Plugin (as daemonset), or restart it whenever you make changes to the configmap
    


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
  # get on the admin machine (or you can run it locally on target machine), where dockerd is running
  $ cd /root
  $ git clone https://github.com/intel/sriov-network-device-plugin
  ```
  
   ?? are these docker image pull needed? seems the device plugin ds deployment would pull the images from upstream registry anyways (Bin/Intel is checking, it is likely because an older version of sriov-network-device-plugin was used in Intel's doc). Confirmed, not needed for the docker pull and crictl command.
   
  ```
  $ docker pull nfvpe/sriov-device-plugin
  ```
  
  ```
  # On the target EKS-A node pull the image down to local cache also
  root@eksa-du:# crictl pull nfvpe/sriov-device-plugin
  root@eksa-du:# crictl image ls
  ```
  
  - SRIOV VF configuration
  
  Get on the target EKS-A machine
  
  Creating VFs with sysfs

  First select a compatible NIC on which to create VFs and record its name (shown as PF_NAME below).

  To create 8 virtual functions run: 
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
  
  Theoretically, we can just move the dpdk-devbind.py script over to the target machine, without installing the whole DPDK package on the target machine. But for now, we can follow the procedure in the Environment Prep section to install DPDK. Finish at least the DPDK package install (until the ninja install)
  
  Then you can use the usertools that comes with DPDK package to check device drivers in use, and bind VF drivers
  
  Check current devices and drivers: (Notice that your driver type may vary depending on the type of NIC in your system)

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
  
  Bind the last four VFs of eno2 to vfio-pci: (notice your PCIE address would be different)
  
  `dpdk-devbind.py -b vfio-pci 18:11.1 18:11.3 18:11.5 18:11.7`
  
  Re-run the dpdk-devbind.py -s command should show the VFs are binded with DPDP compatible drivers (i.e. vfio-pci)
  
  
  Now you are ready to apply SRIOV device plugin configuration and start the dp_daemonset
  
  
  - SRIOV Device Plugin configuration  
  
  on the admin machine, within the gitcloned folder the sriov-network-device-plugin, below is an example to cofigure SRIOV DP configure map, modify the device name according to your target machine configuration (use lspci -nn in the target machine to find out):  (more configuration examples can be found in sriov-device-plugin github page). Device ID can be found from the dpdk-bind.py -s output
  
  Note: the acc100 device VF configuration step is shown in the later section of this doc. Once VF is enabled on ACC100, use the VF's device ID 0d5d instead of the PF's device ID 0d5c

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
  
  # Here is another example (in Mavenir lab) where you have multiple NICs, and each NIC needs to be configured with both DPDK and non-DPDK VFs.
  
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
  
 # Then you create the configmap (some version of sriov-dp folder structure maybe slightly change, but the configmap and daemonset files are two key files)
  
  $ kubectl create -f deployments/configMap.yaml  
  $ kubectl create -f deployments/sriovdp-daemonset.yaml  
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
      "subnet": "10.56.217.0/24",
      "routes": [{
        "dst": "0.0.0.0/0"
      }],
      "gateway": "10.56.217.1"
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
  sys-1019-admin@sys1019admin-CSE-515-R407:~/EKS-A-install/sriov$ kubectl get net-attach-def -A
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
  
  Note: currently this pod test cannot be completed due to the eno2 interface is not connected in the SMC dev lab setup. Will test it once the setup is moved to permanent lab with 2nd NIC port connectivity. In the meantime, will check with Intel to get the procedure to enable SR-IOV on OAM NIC port.
  
  
  - Native CPU Manager 

  Since Kubernetes v1.16.1, there is the support of the CPU manager and topology manager. In this section, steps are provided on how to enable and use these features. You can get more information from the Kubernetes document:
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
  

## 3.4. Prepare env

### 3.4.1. DPDK pakage

Install pre-req packages

```
$ apt install libnuma-dev libhugetlbfs-dev build-essential cmake meson pkgconf python3-pyelftools
```

Note: We only need the OneAPI Basekit for building the FlexRAN image ourselves. For platform testing, we do not need to install OneAPI Basekit on the target machine



Download DPDK

```
# get on the target EKS-A machine
$ cd /opt/  
$ wget http://static.dpdk.org/rel/dpdk-21.11.tar.xz  
$ tar xf /opt/dpdk-21.11.tar.xz
```

Note: We only need the dpdk-22.07-rc3.patch (Refer to Intel Doc 645964 to get the patch) for building the FlexRAN image ourselves. For platform testing, we do not need to install dpdk-22.07-rc3.patch on the target machine


Build and install DPDK

Note: this step can take 10mins, so run it in a tmux session if possible

```shell
$ cd /opt/dpdk_21.11
$ meson build
$ cd build
$ ninja
$ ninja install
```

Note: now you completed the pre-req for "DPDK binding" section above, and can do the DPDK binding part for the backhaul SR-IOV NIC testing. However, to streamline things while you are in this section, it is better to continue below to configurate the Accelerator driver binding below, and then return the "DPDK binding" section.


Build igb_uio (for Accelerator PF driver binding purpose)

```shell
$ cd /opt
$ git clone http://dpdk.org/git/dpdk-kmods
$ cd dpdk-kmods/linux/igb_uio
$ make
```

Configure FEC and FVL SRIOV (example as below)
Note: the kernel module loading needs to be re-do everytime the server rebooted


```shell
$ modprobe vfio-pci
$ modprobe uio
$ insmod /opt/dpdk-kmods/linux/igb_uio/igb_uio.ko
$ lspci | grep acc
c3:00.0 Processing accelerators: Intel Corporation Device 0d5c

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
$ ./pf_bb_config ACC100 -c acc100/acc100_config_1vf_5g.cfg
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

Note: need to re-apply the sriov-dp configmap.yaml (if you have not updated the acc100 VF's device ID in the configmap), also the sriov-dp daemonset needs to be deleted and re-created in order for K8s to pick it up and show it in the resource.

```
kubectl delete -f deployments/sriovdp-daemonset.yaml
kubectl apply -f deployments/sriovdp-daemonset.yaml
```

Now you should be able to see all the K8s resources allocatable (e.g. SRIOV DPDK, non-DPDK/net-device, Accelerator, HugePage Mem, etc.)

```shell
kubectl get node eksa-du -o json | jq '.status.allocatable'
{
  "cpu": "62",
  "ephemeral-storage": "885476035962",
  "hugepages-1Gi": "32Gi",
  "hugepages-2Mi": "0",
  "intel.com/intel_fec_5g": "1",
  "intel.com/intel_sriov_dpdk": "4",
  "intel.com/intel_sriov_netdevice": "4",
  "memory": "96801784Ki",
  "pods": "110"
}
```


## 3.5. flexran docker image prepare

login docker hub (for external user)

```shell
$ docker login 
```
input you username/password of your docker hub account 

```shell
$ docker pull intel/flexran_vdu:v22.07
```

Note: the above method did not work for me, and I got the FlexRAN docker image from Intel team directly


## 3.6. Install flexRAN

Intel flexRAN provide helm chart or yaml files for a sample deployment of flexran test.
If user is NDA customer of flexRAN, they can get those helm chart or yaml files from quarter by quarter release package.
If user is not NDA customer of flexRAN, below give two examples:

### 3.6.1. Example yaml file for flexran timer mode test

```shell
$ cat <<EOF > /opt/flexran_timer_mode.yaml  
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: flexran-binary-release
  name: flexran-binary-release-cpu-mgmt
spec:
  containers:
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m timer ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: 64.71.162.226:5000/flexran_vdu:22.07B23
    name: flexran-l1app
    resources:
      requests:
        cpu: "30"
        memory: "32Gi"
        intel.com/intel_fec_5g: '1'
        hugepages-1Gi: 16Gi
      limits:
        cpu: "30"
        memory: "32Gi"
        intel.com/intel_fec_5g: '1'
        hugepages-1Gi: 16Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /root/flexran/tests
      readOnly: false
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m timer ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: 64.71.162.226:5000/flexran_vdu:22.07B23
    name: flexran-testmac
    resources:
      requests:
        cpu: "4"
        memory: "12Gi"
        hugepages-1Gi: 8Gi
      limits:
        cpu: "4"
        memory: "12Gi"
        hugepages-1Gi: 8Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /root/flexran/tests
      readOnly: false
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
  - name: varrun
    emptyDir: {}
  - name: tests
    hostPath:
      path: "/home/ec2-user/tests"
EOF  
  
$ kubectl create -f /opt/flexran_timer_mode.yaml
```

Make sure the /tests folder is put under the target DU machine's home directory

For timer mode, once the pod is created, you can log into the two containers (for L1 and L2 seperately) of the pods.

```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
flexran-binary-release-cpu-mgmt   2/2     Running   0          3h37m
```

On the L1 container window:

```
$ kubectl exec -it flexran-binary-release-cpu-mgmt -c flexran-l1app /bin/bash
$ cd flexran/bin/nr5g/gnb/l1/

# first find out the cpu cores allocated for L1
$ grep Cpus_allowed_list /proc/self/status

# Note: modify phycfg_timer.xml <dpdkBasebandDevice>0000:c4:00.0</dpdkBasebandDevice> to env|grep INTEL value, also double check the <dpdkBasebandFecMode> and <dpdkIovaMode> values are both 1.
# also modify the <!-- CPU Binding to Application Threads --> section of the phycfg_timer.xml file, to assign 3 cores from the cpu core allow list for FlexRAN system purpose.

# Then start the L1 process
$ ./l1.sh -e
```

On the L2 container window:

```
$ kubectl exec -it flexran-binary-release-cpu-mgmt -c flexran-testmac /bin/bash 
$ cd flexran/bin/nr5g/gnb/testmac

# Note: modify the icxsp_mu0_10mhz_4x4_hton.cfg file, in the "setcore 0x7e" section, make sure the core mask falls into the cpu allowed list of L1 container, and also does not overlap with FlexRAN system cores defined in the L1 container phycfg_timer.xml file above

# Then start the L2 process
$ ./l2.sh –testfile=icelake-sp/icxsp_mu0_10mhz_4x4_hton.cfg
```

All the test results are printed on the screen, and also saved into the local file of L1 container: /flexran/bin/nr5g/gnb/l1/l1_mlog_stats.txt



### 3.6.2. Example yaml file for xran mode test

```shell
$ cat <<EOF > /opt/flexran_xran_mode.yaml  
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: flexran-vdu
  name: flexran-vdu
spec:
  nodeSelector:
     testnode: worker1
  containers:
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m xran ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
    name: flexran-vdu
    resources:
      requests:
        memory: "12Gi"
        intel.com/intel_fec_5g: '1'
        intel.com/intel_sriov_odu: '4'
        hugepages-1Gi: 16Gi
      limits:
        memory: "12Gi"
        intel.com/intel_fec_5g: '1'
        intel.com/intel_sriov_odu: '4'
        hugepages-1Gi: 16Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /home/flexran/tests
      readOnly: false
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
  - name: varrun
    emptyDir: {}
  - name: tests
    hostPath:
      path: "/home/tmp_flexran/tests"
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: flexran-vru
  name: flexran-vru
spec:
  nodeSelector:
     testnode: worker1
  containers:
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m xran ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
    name: flexran-oru
    resources:
      requests:
        memory: "4Gi"
        intel.com/intel_sriov_oru: '4'
        hugepages-1Gi: 6Gi
      limits:
        memory: "4Gi"
        intel.com/intel_sriov_oru: '4'
        hugepages-1Gi: 6Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /home/flexran/tests
      readOnly: false
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
  - name: varrun
    emptyDir: {}
  - name: tests
    hostPath:
      path: "/home/tmp_flexran/tests"
EOF  

$ kubectl create -f /opt/flexran_xran_mode.yaml
```

For xran mode, once the container created, corresponding xran mode test will not be run up.
You need enter the pod and execute the test manually.
Below chapter give the steps to run xRAN mode test case: "sub3_mu0_20mhz_4x4"

Open a new terminal, run the following command:

```shell
$ kubectl exec -it pod-name -- bash    
$ cd flexran/bin/nr5g/gnb/l1/orancfg/sub3_mu0_20mhz_4x4/gnb/  
$ ./l1.sh -oru
```
  
Open another new terminal, run the following command:

```shell
$ kubectl exec -it pod-name -- bash    
$ cd flexran/bin/nr5g/gnb/testmac  
$ ./l2.sh –testfile=testmac_clxsp_mu0_20mhz_hton_oru.cfg  

#### Open another new terminal, run the following command:
$ kubectl exec -it pod-name -- bash

$ cd flexran/bin/nr5g/gnb/l1/orancfg/sub3_mu0_20mhz_4x4/oru/
$ ./run_o_ru.sh
```

You can run the same for other two test cases.
for 2nd xRAN mode test case: "sub3_mu0_10mhz_4x4"

Open a new terminal, run the following command:

```shell
kubectl exec -it pod-name -- bash    
cd flexran/bin/nr5g/gnb/l1/orancfg/sub3_mu0_10mhz_4x4/gnb/  
./l1.sh -oru
```
  
Open another new terminal, run the following command:

```shell
$ kubectl exec -it pod-name -- bash    
$ cd flexran/bin/nr5g/gnb/testmac  
$ ./l2.sh –testfile=testmac_clxsp_mu0_10mhz_hton_oru.cfg  

#### Open another new terminal, run the following command:
$ kubectl exec -it pod-name -- bash

$ cd flexran/bin/nr5g/gnb/l1/orancfg/sub3_mu0_10mhz_4x4/oru/
$ ./run_o_ru.sh
```

for 3rd xRAN mode test case: "sub6_mu1_100mhz_4x4"

Open a new terminal, run the following command:

```shell
$ kubectl exec -it pod-name -- bash    
$ cd flexran/bin/nr5g/gnb/l1/orancfg/sub6_mu1_100mhz_4x4/gnb/  
$ ./l1.sh -oru
```
  
Open another new terminal, run the following command:

```shell
$ kubectl exec -it pod-name -- bash    
$ cd flexran/bin/nr5g/gnb/testmac  
$ ./l2.sh –testfile=testmac_clxsp_mu1_100mhz_hton_oru.cfg  

#### Open another new terminal, run the following command:
$ kubectl exec -it pod-name -- bash

$ cd flexran/bin/nr5g/gnb/l1/orancfg/sub6_mu1_100mhz_4x4/oru/
$ ./run_o_ru.sh
```
  
## 3.7. Core pining

Intel docker image also provide the support of core pining feature.
In order to enable this feature, you need to make below configuration and change of yaml file.

### 3.7.1. Configuration

Enable core pining feature： 

```shell
$ cat  <<EOF > core_pining_kubelet_config.sh
#!/bin/bash
pathfile=/var/lib/kubelet/config.yaml
sed -i 's/cpuManagerReconcilePeriod: 0s/cpuManagerReconcilePeriod: 10s/g' $pathfile

cat >> $pathfile << EOF
cpuManagerPolicy: static
systemReserved:
  cpu: 2000m
  memory: 2000Mi
kubeReserved:
  cpu: 1000m
  memory: 1000Mi
EOF
rm -rf /var/lib/kubelet/cpu_manager_state
systemctl restart kubelet
EOF
  
$ sh core_pining_kubelet_config.sh
```

Disable core pining feature:
$ cat <<EOF > uncore_pining_kubelet_config.sh
#!/bin/bash

pathfile=/var/lib/kubelet/config.yaml
#pathfile=config.yaml
sed -i 's/cpuManagerReconcilePeriod: 10s/cpuManagerReconcilePeriod: 0s/g' $pathfile

sed -i 's/cpuManagerPolicy: static/ /g' $pathfile
sed -i 's/systemReserved:/ /g' $pathfile
sed -i 's/cpu: 2000m/ /g' $pathfile
sed -i 's/memory: 2000Mi/ /g' $pathfile
sed -i 's/kubeReserved:/ /g' $pathfile
sed -i 's/cpu: 1000m/ /g' $pathfile
sed -i 's/memory: 1000Mi/ /g' $pathfile
rm -rf /var/lib/kubelet/cpu_manager_state
systemctl restart kubelet
EOF

### 3.7.2. Example yaml file for flexran timer mode test (with core pining)

```shell
$ cat <<EOF > /opt/flexran_timer_mode.yaml  
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: flexran-dockerimage_release
  name: flexran-dockerimage-release
spec:
  nodeSelector:
     testnode: worker1
  containers:
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m timer ; cd /home/flexran/bin/nr5g/gnb/l1/; ./l1.sh -e ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
    name: flexran-l1app
    resources:
      requests:
        memory: "12Gi"
        intel.com/intel_fec_5g: '1'
        hugepages-1Gi: 16Gi
      limits:
        memory: "12Gi"
        intel.com/intel_fec_5g: '1'
        hugepages-1Gi: 16Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /home/flexran/tests
      readOnly: false
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sleep 10; sh docker_entry.sh -m timer ; cd /home/flexran/bin/nr5g/gnb/testmac/; ./l2.sh --testfile=icelake-sp/icxsp.cfg; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
    name: flexran-testmac
    resources:
      requests:
        memory: "6Gi"
        hugepages-1Gi: 4Gi
      limits:
        memory: "6Gi"
        hugepages-1Gi: 4Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /home/flexran/tests
      readOnly: false
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
  - name: varrun
    emptyDir: {}
  - name: tests
    hostPath:
      path: "/home/tmp_flexran/tests"
EOF  
  
$ kubectl create -f /opt/flexran_timer_mode.yaml
```

for timer mode, once the container created, corresponding timer mode test will be run up. And you can check POD status thru - "kubectl describe po pode-name".  
You can also check the status of RAN service thru - "kubectl logs -f pode-name -c container-name"
  
### 3.7.3. Example yaml file for xran mode test (with core pining)

```shell
$ cat <<EOF > /opt/flexran_xran_mode.yaml  
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: flexran-vdu
  name: flexran-vdu
spec:
  nodeSelector:
     testnode: worker1
  containers:
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m xran ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
    name: flexran-vdu
    resources:
      requests:
        memory: "24Gi"
        intel.com/intel_fec_5g: '1'
        intel.com/intel_sriov_odu: '4'
        hugepages-1Gi: 24Gi
      limits:
        memory: "24Gi"
        intel.com/intel_fec_5g: '1'
        intel.com/intel_sriov_odu: '4'
        hugepages-1Gi: 24Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /home/flexran/tests
      readOnly: false
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
  - name: varrun
    emptyDir: {}
  - name: tests
    hostPath:
      path: "/home/tmp_flexran/tests"
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: flexran-vru
  name: flexran-vru
spec:
  nodeSelector:
     testnode: worker1
  containers:
  - securityContext:
      privileged: false
      capabilities:
        add:
          - IPC_LOCK
          - SYS_NICE
    command: [ "/bin/bash", "-c", "--" ]
    args: ["sh docker_entry.sh -m xran ; top"]
    tty: true
    stdin: true
    env:
    - name: LD_LIBRARY_PATH
      value: /opt/oneapi/lib/intel64
    image: amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
    name: flexran-oru
    resources:
      requests:
        memory: "24Gi"
        intel.com/intel_sriov_oru: '4'
        hugepages-1Gi: 16Gi
      limits:
        memory: "24Gi"
        intel.com/intel_sriov_oru: '4'
        hugepages-1Gi: 16Gi
    volumeMounts:
    - name: hugepage
      mountPath: /hugepages
    - name: varrun
      mountPath: /var/run/dpdk
      readOnly: false
    - name: tests
      mountPath: /home/flexran/tests
      readOnly: false
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
  - name: varrun
    emptyDir: {}
  - name: tests
    hostPath:
      path: "/home/tmp_flexran/tests"
EOF  

$ kubectl create -f /opt/flexran_xran_mode.yaml
```

For xran mode, once the container created, corresponding xran mode test will not be run up.
You need enter the pod and execute the test manually. the steps are the same as the one without core pining 

## 3.8. ISV vDU on-boarding
  
### PTP

Before enabling PTP service, need to disable the NTP service in the system
  
```
systemctl status chronyd
systemctl stop chronyd
systemctl status chronyd
```
  
  
## 3.9. Legal Disclaimer

For GPL/LGPL open source libs/components used by flexran docker image at run time.
User can find the used version in below git hub repo: <https://github.com/intel/flexRAN-docker-image-dependencies>
  

