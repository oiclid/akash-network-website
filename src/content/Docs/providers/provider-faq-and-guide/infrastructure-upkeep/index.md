---
categories: ["Providers"]
tags: []
weight: 2
title: "Infrastructure, Upkeep, and Advanced Operations"
linkTitle: "Infrastructure, Upkeep, and Advanced Operations"
---


## Maintaining and Rotating Kubernetes/etcd Certificates: A How-To Guide

> The following doc is based on [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/) & [https://www.txconsole.com/posts/how-to-renew-certificate-manually-in-kubernetes](https://www.txconsole.com/posts/how-to-renew-certificate-manually-in-kubernetes)

When K8s certs expire, you won't be able to use your cluster. Make sure to rotate your certs proactively.

The following procedure explains how to rotate them manually.

Evidence that the certs have expired:

```
root@node1:~# kubectl get nodes -o wide
error: You must be logged in to the server (Unauthorized)
```

You can always view the certs expiration using the `kubeadm certs check-expiration` command:

```
root@node1:~# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[check-expiration] Error reading configuration from the Cluster. Falling back to default configuration

CERTIFICATE                         EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                          Feb 20, 2023 17:12 UTC   <invalid>       ca                      no
apiserver                           Mar 03, 2023 16:42 UTC   10d             ca                      no
!MISSING! apiserver-etcd-client
apiserver-kubelet-client            Feb 20, 2023 17:12 UTC   <invalid>       ca                      no
controller-manager.conf             Feb 20, 2023 17:12 UTC   <invalid>       ca                      no
!MISSING! etcd-healthcheck-client
!MISSING! etcd-peer
!MISSING! etcd-server
front-proxy-client                  Feb 20, 2023 17:12 UTC   <invalid>       front-proxy-ca          no
scheduler.conf                      Feb 20, 2023 17:12 UTC   <invalid>       ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Feb 18, 2032 17:12 UTC   8y              no
!MISSING! etcd-ca
front-proxy-ca          Feb 18, 2032 17:12 UTC   8y              no
root@node1:~#
```

### Rotate K8s Certs

#### Backup etcd DB

It is crucial to back up your etcd DB as it contains your K8s cluster state! So make sure to backup your etcd DB first before rotating the certs!

##### Take the etcd DB Backup

> Replace the etcd key & cert with your locations found in the prior steps

```
export $(grep -v '^#' /etc/etcd.env | xargs -d '\n')
etcdctl -w table member list
etcdctl endpoint health --cluster -w table
etcdctl endpoint status --cluster -w table
etcdctl snapshot save node1.etcd.backup
```

You can additionally backup the current certs:

```
tar czf etc_kubernetes_ssl_etcd_bkp.tar.gz /etc/kubernetes /etc/ssl/etcd
```

#### Renew the Certs

> IMPORTANT: For an HA Kubernetes cluster with multiple control plane nodes, the `kubeadm certs renew` command (followed by the `kube-apiserver`, `kube-scheduler`, `kube-controller-manage`r pods and `etcd.service` restart) needs to be executed on all the control-plane nodes, on one control plane node at a time, starting with the primary control plane node. This approach ensures that the cluster remains operational throughout the certificate renewal process and that there is always at least one control plane node available to handle API requests. To find out whether you have an HA K8s cluster (multiple control plane nodes) use this command `kubectl get nodes -l node-role.kubernetes.io/control-plane`

Now that you have the etcd DB backup, you can rotate the K8s certs using the following commands:

##### Rotate the k8s Certs

```
kubeadm certs renew all
```

##### Update your kubeconfig

```
mv -vi /root/.kube/config /root/.kube/config.old
cp -pi /etc/kubernetes/admin.conf /root/.kube/config
```

##### Bounce the following services in this order

```
kubectl -n kube-system delete pods -l component=kube-apiserver
kubectl -n kube-system delete pods -l component=kube-scheduler
kubectl -n kube-system delete pods -l component=kube-controller-manager
systemctl restart etcd.service
```

##### Verify the Certs Status

```
kubeadm certs check-expiration
```

Repeat the process for all control plane nodes, one at a time, if you have a HA Kubernetes cluster.



## Provider Bid Script Migration - GPU Models

A new bid script for Akash Providers has been released that now includes the ability to specify pricing of multiple GPU models.

This document details the recommended procedure for Akash providers needing migration to the new bid script from prior versions.

##### New Features of Bid Script Release

- Support for parameterized price targets (configurable through the Akash/Provider Helm chart values), eliminating the need to manually update your bid price script
- Pricing based on GPU model, allowing you to specify different prices for various GPU models

##### How to Migrate from Prior Bid Script Releases

##### STEP 1 - Backup your current bid price script

> This command will produce an `old-bid-price-script.sh` file which is your currently active bid price script with your custom modifications

```
helm -n akash-services get values akash-provider -o json | jq -r '.bidpricescript | @base64d' > old-bid-price-script.sh
```

##### STEP 2 - Verify Previous Custom Target Price Values

```
cat old-bid-price-script.sh | grep ^TARGET
```

##### Example/Expected Output

```
# cat old-bid-price-script.sh | grep ^TARGET
TARGET_CPU="1.60"          # USD/thread-month
TARGET_MEMORY="0.80"       # USD/GB-month
TARGET_HD_EPHEMERAL="0.02" # USD/GB-month
TARGET_HD_PERS_HDD="0.01"  # USD/GB-month (beta1)
TARGET_HD_PERS_SSD="0.03"  # USD/GB-month (beta2)
TARGET_HD_PERS_NVME="0.04" # USD/GB-month (beta3)
TARGET_ENDPOINT="0.05"     # USD for port/month
TARGET_IP="5"              # USD for IP/month
TARGET_GPU_UNIT="100"      # USD/GPU unit a month
```

#### STEP 3 - Backup Akash/Provider Config

> This command will backup your akash/provider config in the `provider.yaml` file (excluding the old bid price script)

```
helm -n akash-services get values akash-provider | grep -v '^USER-SUPPLIED VALUES' | grep -v ^bidpricescript > provider.yaml
```

#### STEP 4 - Update provider.yaml File Accordingly

> Update your `provider.yaml` file with the price targets you want. If you don't specify these keys, the bid price script will default values shown below

`price_target_gpu_mappings` sets the GPU price in the following way and in the example provided:

- `a100` nvidia models will be charged by `120` USD/GPU unit a month
- `t4` nvidia models will be charged by `80` USD/GPU unit a month
- Unspecified nvidia models will be charged `130` USD/GPU unit a month (if `*` is not explicitly set in the mapping it will default to `100` USD/GPU unit a month)
- Extend with more models your provider is offering if necessary with syntax of `<model>=<USD/GPU unit a month>`
- If your GPU model has different possible RAM specs - use this type of convention: `a100.40Gi=900,a100.80Gi=1000`

```
price_target_cpu: 1.60
price_target_memory: 0.80
price_target_hd_ephemeral: 0.02
price_target_hd_pers_hdd: 0.01
price_target_hd_pers_ssd: 0.03
price_target_hd_pers_nvme: 0.04
price_target_endpoint: 0.05
price_target_ip: 5
price_target_gpu_mappings: "a100=120,t4=80,*=130"
```

#### STEP 5 - Download New Bid Price Script

```
mv -vi price_script_generic.sh price_script_generic.sh.old

wget https://raw.githubusercontent.com/akash-network/helm-charts/main/charts/akash-provider/scripts/price_script_generic.sh
```

#### STEP 6 - Upgrade Akash/Provider Chart to Version 6.0.5

```
helm repo update akash

helm search repo akash/provider
```

##### Expected/Example Output

```
# helm repo update akash
# helm search repo akash/provider
NAME          	CHART VERSION	APP VERSION	DESCRIPTION
akash/provider	6.0.5        	0.4.6      	Installs an Akash provider (required)
```

#### STEP 7 - Upgrade akash-provider Deployment with New Bid Script

```
helm upgrade akash-provider akash/provider -n akash-services -f provider.yaml --set bidpricescript="$(cat price_script_generic.sh | openssl base64 -A)"
```

##### Verification of Bid Script Update

```
helm list -n akash-services | grep akash-provider
```

##### Expected/Example Output

```
# helm list -n akash-services | grep akash-provider
akash-provider         	akash-services	28      	2023-09-19 12:25:33.880309778 +0000 UTC	deployed	provider-6.0.5                	0.4.6
```

## GPU Provider Troubleshooting

Should your Akash Provider encounter issues during the installation process or in post install hosting of GPU resources, follow the troubleshooting steps in this guide to isolate the issue.

> _**NOTE**_ - these steps should be conducted on each Akask Provider/Kubernetes worker nodes that host GPU resources unless stated otherwise within the step

- [Basic GPU Resource Verifications](#basic-gpu-resource-verifications)
- [Examine Linux Kernel Logs for GPU Resource Errors and Mismatches](#examine-linux-kernel-logs-for-gpu-resource-errors-and-mismatches)
- [Ensure Correct Version/Presence of NVIDIA Device Plugin](#ensure-correct-versionpresence-of-nvidia-device-plugin)
- [NVIDIA Fabric Manager](#nvidia-fabric-manager)

### Basic GPU Resource Verifications

- Conduct the steps in this section for basic verification and to ensure the host has access to GPU resources

#### Prep/Package Installs

```
apt update && apt -y install python3-venv

python3 -m venv /venv
source /venv/bin/activate
pip install torch numpy
```

#### Confirm GPU Resources Available on Host

> _**NOTE**_ - example verification steps were conducted on a host with a single NVIDIA T4 GPU resource. Your output will be different based on the type and number of GPU resources on the host.

```
nvidia-smi -L
```

##### Example/Expected Output

```
# nvidia-smi -L

GPU 0: Tesla T4 (UUID: GPU-faa48437-7587-4bc1-c772-8bd099dba462)
```

#### Confirm CUDA Install & Version

```
python3 -c "import torch;print(torch.version.cuda)"
```

##### Example/Expected Output

```
# python3 -c "import torch;print(torch.version.cuda)"

11.7
```

#### Confirm CUDA GPU Support iis Available for Hosted GPU Resources

```
python3 -c "import torch; print(torch.cuda.is_available())"
```

##### Example/Expected Output

```
# python3 -c "import torch; print(torch.cuda.is_available())"

True
```

### Examine Linux Kernel Logs for GPU Resource Errors and Mismatches

```
dmesg -T | grep -Ei 'nvidia|nvml|cuda|mismatch'
```

##### Example/Expected Output

> _**NOTE**_ - example output is from a healthy host which loaded NVIDIA drivers successfully and has no version mismatches. Your output may look very different if there are issues within the host.

```
# dmesg -T | grep -Ei 'nvidia|nvml|cuda|mismatch'

[Thu Sep 28 19:29:02 2023] nvidia: loading out-of-tree module taints kernel.
[Thu Sep 28 19:29:02 2023] nvidia: module license 'NVIDIA' taints kernel.
[Thu Sep 28 19:29:02 2023] nvidia-nvlink: Nvlink Core is being initialized, major device number 237
[Thu Sep 28 19:29:02 2023] NVRM: loading NVIDIA UNIX x86_64 Kernel Module  535.104.05  Sat Aug 19 01:15:15 UTC 2023
[Thu Sep 28 19:29:02 2023] nvidia-modeset: Loading NVIDIA Kernel Mode Setting Driver for UNIX platforms  535.104.05  Sat Aug 19 00:59:57 UTC 2023
[Thu Sep 28 19:29:02 2023] [drm] [nvidia-drm] [GPU ID 0x00000004] Loading driver
[Thu Sep 28 19:29:03 2023] audit: type=1400 audit(1695929343.571:3): apparmor="STATUS" operation="profile_load" profile="unconfined" name="nvidia_modprobe" pid=300 comm="apparmor_parser"
[Thu Sep 28 19:29:03 2023] audit: type=1400 audit(1695929343.571:4): apparmor="STATUS" operation="profile_load" profile="unconfined" name="nvidia_modprobe//kmod" pid=300 comm="apparmor_parser"
[Thu Sep 28 19:29:04 2023] [drm] Initialized nvidia-drm 0.0.0 20160202 for 0000:00:04.0 on minor 0
[Thu Sep 28 19:29:05 2023] nvidia_uvm: module uses symbols nvUvmInterfaceDisableAccessCntr from proprietary module nvidia, inheriting taint.
[Thu Sep 28 19:29:05 2023] nvidia-uvm: Loaded the UVM driver, major device number 235.
```

### Ensure Correct Version/Presence of NVIDIA Device Plugin

> _**NOTE**_ - conduct this verification step on the Kubernetes control plane node on which Helm was installed during your Akash Provider build

```
helm -n nvidia-device-plugin list
```

##### Example/Expected Output

```
# helm -n nvidia-device-plugin list

NAME    NAMESPACE               REVISION    UPDATED                                    STATUS      CHART                          APP VERSION
nvdp    nvidia-device-plugin    1           2023-09-23 14:30:34.18183027 +0200 CEST    deployed    nvidia-device-plugin-0.14.1    0.14.1
```

### NVIDIA Fabric Manager

- In some circumstances it has been found that the NVIDIA Fabric Manager needs to be installed on worker nodes hosting GPU resources (e.g. non-PCIe GPU configurations such as those using SXM form factors)
- If the output of the `torch.cuda.is_available()` command - covered in prior section in this doc - is an error condition, consider installing the NVIDIA Fabric Manager to resolve issue
- Frequently encountered error message encounter when issue exists:\
  \
  `torch.cuda.is_available() function: Error 802: system not yet initialized (Triggered internally at ../c10/cuda/CUDAFunctions.cpp:109.)`
- Further details on the NVIDIA Fabric Manager are available [here](https://forums.developer.nvidia.com/t/error-802-system-not-yet-initialized-cuda-11-3/234955)

> _**NOTE**_ - replace `525` in the following command with the NVIDIA driver version used on your host

> _**NOTE**_ - you may need to wait for about 2-3 minutes for the nvidia fabricmanager to initialize

```
apt-get install nvidia-fabricmanager-525
systemctl start nvidia-fabricmanager
systemctl enable nvidia-fabricmanager
```

- **nvidia-fabricmanager** package version mismatch


Occasionally, the Ubuntu repositories may not provide the correct version of the **nvidia-fabricmanager** package. This can result in the `Error 802: system not yet initialized` error on SXM NVIDIA GPUs.

A common symptom of this issue is that **nvidia-fabricmanager** fails to start properly:

```
# systemctl status nvidia-fabricmanager
Nov 05 13:55:26 node1 systemd[1]: Starting NVIDIA fabric manager service...
Nov 05 13:55:26 node1 nv-fabricmanager[104230]: fabric manager NVIDIA GPU driver interface version 550.127.05 don't match with driver version 550.120. Please update with matching NVIDIA driver package.
Nov 05 13:55:26 node1 systemd[1]: nvidia-fabricmanager.service: Control process exited, code=exited, status=1/FAILURE
```

To resolve this issue, you’ll need to use the official NVIDIA repository. Here's how to add it:

> _**NOTE**_ - replace `2204` with your Ubuntu version (e.g. `2404` for Ubuntu noble release)

> _**NOTE**_ - Running `apt dist-upgrade` with the official NVIDIA repo bumps the `nvidia` packages along with the `nvidia-fabricmanager`, without version mismatch issue.

```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/3bf863cc.pub
apt-key add 3bf863cc.pub

echo "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/ /" > /etc/apt/sources.list.d/nvidia-official-repo.list
apt update
apt dist-upgrade
apt autoremove
```

> `dpkg -l | grep nvidia` -- make sure to remove any version you don't expect
> and reboot

