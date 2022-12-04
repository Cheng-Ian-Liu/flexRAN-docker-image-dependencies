# 0. Background

The purpose of this document is to summarize all the host-level and K8s-level configurations we made to the platform to support Intel FlexRAN on-boarding.
The document is modified based on Intel Open Source FlexRAN installation guide (https://github.com/intel/flexRAN-docker-image-dependencies), based on specific changes tailored for our environment.

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

EKS-A install completed with Generic Ubuntu OS up and running

### 3.3.0. RT kernel install

Follow this article to install the RT kernel for Ubuntu 22.04, beta version: https://ubuntu.com/blog/real-time-ubuntu-released

To attach your personal machine to a Pro subscription, please run:

```
pro attach <free token> 
```

To enable the real-time beta kernel, run:

```
pro enable realtime-kernel --beta
```

Then reboot the server, and your kernel version should become RT kernal

```
root@eksa-du:/home/ec2-user# uname -ar
Linux eksa-du 5.15.0-1025-realtime #28-Ubuntu SMP PREEMPT_RT Fri Oct 28 23:19:16 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

After the server is rebooted, install the following packages that will be used later when setting CPU core frequency

```
sudo apt install linux-tools-5.15.0-1025-realtime
sudo apt install linux-cloud-tools-5.15.0-1025-realtime
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

Edit /etc/tuned/realtime-variables.conf to add isolated_cores=1-31, 33-63: (in the case of 32pCores/64vCores CPU, exclude 0 and 32 as sibiling vCores for house keeping purpose)

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

The other parameters of this set of best known configuration can be simply added in /etc/default/grub as below:

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
$ wrmsr -a 0x199 0x1900
```


Set cpu uncore (i.e. 0x620) to fixed – maximum allowed. Disable c6 and c1e

```shell
$ wrmsr -a 0x620 0x1e1e
$ cpupower idle-set -d 3
$ cpupower idle-set -d 2
```

### 3.3.3. Kubernetes installation

This section is removed, as it is handled by EKS-A installation

### 3.3.4. Kubernetes plugins installation

Obtain the kubeconfig file of the target cluster, and execute the commands below to install additional K8s plugins

Blow plugins are required:

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
  - follow SRIOV instruction on SRIOV GitHub - <https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin#sr-iov-network-device-plugin>.
  below is the command for installation:
  
  ```shell
  # on the admin machine, where dockerd is running
  $ cd /root
  $ git clone https://github.com/intel/sriov-network-device-plugin
  $ docker pull nfvpe/sriov-device-plugin
  ```
  
  - SRIOV VF configuration
  
  Creating VFs with sysfs

  First select a compatible NIC on which to create VFs and record its name (shown as PF_NAME below).

  To create 8 virtual functions run: (Note: do not create the SR-IOV VFs on the OAM NIC port)

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

   This method requires the creation of VFs each time the node resets. This can be handled automatically by placing the above command in a script that is run on startup such as /etc/rc.local.
  
  
  - SRIOV DP configuration  
  below is an example to cofigure SRIOV DP configure map:  

      ```shell
      $ cd sriov-network-device-plugin 
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
                          "devices": ["0d5d"]  
                      }  
                  },  
                  {  
                    "resourceName": "intel_sriov_odu",  
                      "selectors": {  
                          "vendors": ["8086"],  
                          "devices": ["154c"],  
                          "drivers": ["vfio-pci"],  
                          "pfNames": ["ens9f1"]  
                      }  
                  },  
                  {  
                    "resourceName": "intel_sriov_oru",  
                      "selectors": {  
                          "vendors": ["8086"],  
                          "devices": ["154c"],  
                          "drivers": ["vfio-pci"],  
                          "pfNames": ["ens9f0"]  
                      }  
                  }  
              ]  
          }  
      EOF  
      $ kubectl create -f deployments/configMap.yaml  
      $ kubectl create -f deployments/k8s-v1.16/sriovdp-daemonset.yaml  
      $ kubectl get node <your-k8s-worker> -o json | jq '.status.allocatable' 
      {  
      "cpu": "28",  
      "ephemeral-storage": "143494008185",  
      "hugepages-1Gi": "48Gi",  
      "intel.com/intel_sriov_dpdk": "4",  
      "intel.com/intel_sriov_netdevice": "4",  
      "memory": "48012416Ki",  
      "pods": "110"  
      }
      ```

  - Native CPU Manager 

  enable this plugin by following below link:  

  - <https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/>  
  - <https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/> 

## 3.4. Prepare env

### 3.4.1. DPDK pakage

Download DPDK

```shell
$ cd /opt/  
$ wget http://static.dpdk.org/rel/dpdk-21.11.tar.xz  
$ tar xf /opt/dpdk-21.11.tar.xz
```

Build and install DPDK

```shell
$ cd /opt/dpdk_21.11
$ meson build
$ cd build
$ ninja
$ ninja install
```

Build igb_uio

```shell
$ cd /opt
$ git clone http://dpdk.org/git/dpdk-kmods
$ cd dpdk-kmods/linux/igb_uio
$ make
```

Configure FEC and FVL SRIOV (example as below)

```shell
$ modprobe vfio-pci
$ modprobe uio
$ insmod /opt/dpdk-kmods/linux/igb_uio/igb_uio.ko
$ lspci | grep 0d5c

$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b igb_uio 0000:b1:00.0
$ echo 0 > /sys/bus/pci/devices/0000:b1:00.0/max_vfs
$ echo 2 > /sys/bus/pci/devices/0000:b1:00.0/max_vfs
$ /opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 0000:b2:00.0
$ cd /opt/pf-bb-config
$ ./pf_bb_config ACC100 -c acc100/acc100_config_vf_5g.cfg

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

After configuration, need to restart SRIOV docker container to make VF resource ready.

```shell
$ cat <<EOF > /opt/restart_sriov_container.sh
  docker ps | grep sriov
  docker kill `docker ps | grep sriov | head -n 1 | awk -F ' ' '{print $1}'`
  while ((`docker ps | grep sriov | wc -l` < 2 ))
  do
     sleep 3
     docker ps | grep sriov >/dev/null 2>&1
     echo "..."
  done
EOF
$chmod +x /opt/restart_sriov_container.sh; sh /opt/restart_sriov_container.sh
```

After restart sriov container, the resources can be seen thru below command:

```shell
$ kubectl get node dockerimagerel -o json | jq '.status.allocatable'
{  
  "cpu": "125",  
  "ephemeral-storage": "427010803415",  
  "hugepages-1Gi": "60Gi",  
  "hugepages-2Mi": "0",  
  "intel.com/intel_fec_5g": "1",  
  "intel.com/intel_sriov_odu": "4",  
  "intel.com/intel_sriov_oru": "4",  
  "memory": "193028Mi",  
  "pods": "110"  
}  
```

label node 

```shell
$ kubectl label node dockerimagerel testnode=worker1
```

## 3.5. docker image prepare

login docker hub (for external user)

```shell
$ docker login 
```
input you username/password of your docker hub account 

```shell
$ docker pull intel/flexran_vdu:v22.07
```

for internal user (intel PAE), they can use another way to download docker image from internal harbor: 
login docker hub 

```shell
$ docker login amr-registry-pre.caas.intel.com
```
input your username/passward of your harbor account

```shell
$ docker pull amr-registry-pre.caas.intel.com/flexran/flexran_vdu:v22.07
```

## 3.6. Install flexRAN thru helm

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
        memory: "6Gi"
        intel.com/intel_fec_5g: '1'
        hugepages-1Gi: 8Gi
      limits:
        memory: "6Gi"
        intel.com/intel_fec_5g: '1'
        hugepages-1Gi: 8Gi
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
  
## 3.8. Legal Disclaimer

For GPL/LGPL open source libs/components used by flexran docker image at run time.
User can find the used version in below git hub repo: <https://github.com/intel/flexRAN-docker-image-dependencies>
  
## 3.9. Customer Support Declare

For further support, please contact intel flexRAN marketing team and FAE/PAE team.
