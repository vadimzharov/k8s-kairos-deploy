## Testing Kairos as an OS for k8s cluster


In my homelab, I maintain a couple of Kubernetes (k8s) clusters to run various workloads and experiment with different k8s tools, features, and technologies. The choice of the operating system (OS) to deploy these k8s clusters is a critical element for me. One of my primary criteria for selecting the OS is its ability to facilitate easy reinstallation of my k8s clusters. This means that the OS must offer a straightforward method for resetting itself to its initial state, enabling me to reinstall k8s from scratch. Ideally, this process should not necessitate the deployment of additional tools in the environment, such as PXE boot.

I've experimented with various OS options like CentOS, Ubuntu, Talos, among others, in my quest to find the perfect fit for my requirements (although I haven't found the perfect one yet). Recently, I came across a new OS specifically designed for running containers and (edge) k8s clusters - Kairos OS. What sets Kairos OS apart is its immutability and the fact that it operates as a single container image. Users can choose from various Linux operating systems like Ubuntu or Fedora as their base OS, and Kairos then ensures immutability, easy management, and readiness for running k8s workloads.

Out of the box, Kairos OS offers the ability to install a k3s cluster in minutes in any environment. You simply download the OS, specify your desire for a k3s installation in the configuration file, and then boot it on any virtual machine or bare-metal server to enjoy its benefits. However, in my homelab, I prefer using k8s instead of k3s. Therefore, the OS must have Kubernetes binaries and a container runtime. By default, Kairos doesn't include these components, but it does provide users with the capability to build their own images derived from scratch, using base container images from different Linux distributions.

To thoroughly test Kairos, I decided to attempt the installation of a k8s cluster based on this OS and subsequently upgrade it. For my Container Network Interface (CNI), I opted to use Cilium (specifically, Cilium Cluster Mesh) to interconnect my k8s clusters and balance workloads between them. As the underlying platform, I chose the KVM hypervisor.

Briefly, here are the steps I followed to install k8s cluster based on Kairos OS on KVM hypervisor:

At first, I need to build my own Kairos container image which must have packages required for k8s to run - **kubelet**, **kubeadm**, **kubectl** and container runtime (**containerd** in my case). Since Kairos can be based on any popular Linux distro, I decided to build it on Ubuntu 22.02, just because my other k8s clusters use this OS.

Then, I need to create ISO image - to install Kairos OS on a VM I'm creating on KVM hypervisor. Here I have a choice - bake the configuration file with Kairos ISO image - so the OS will configure itself without any interaction from a user - or I can build base image and then provide configuration file via SSH or WebUI.
I decided to test both options - for my k8s master I'm going to bootstrap ISO, provide configuration file via WebUI and then initiate k8s cluster, but for my k8s node I will bake configuration file inside the ISO image - so once I bootstrap any VM using this ISO - it will become part of my k8s cluster almost without any manual steps (but due to security reason I want to manually approve CSR from fresh k8s node.)

After cluster installed, I want to upgrade it (k8s packages) and then reset it and re-install it using only SSH access (without accessing host console/mounting ISO).


### Building Kairos OS container image.

The guide how to build custom Kairos image is here - https://kairos.io/docs/reference/build-from-scratch/.

To create my custom Kairos OS container image I need to install k8s binaries (version **1.28.0**) and tune OS according to the requirements from Cilium CNI. At this stage, if you want to install any additional software or configure OS - you can edit Dockerfile to suit you needs.

I named the Dockerfile as [Dockerfile.00](Dockerfile.00) - for this image I'm installing k8s binaries version 1.28.0.
During the build I'm also configuring sysctl, kernel modules (for Cilium to work) and provide containerd configuration file (all required files are in `assets/` directory).

```
FROM quay.io/kairos/core-ubuntu-22-lts:latest as base
RUN mkdir -p /run/lock
RUN touch /usr/libexec/.keep

RUN mkdir -p /var/cache/apt/archives/partial

RUN curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmour -o /etc/apt/trusted.gpg.d/kubernetes-xenial.gpg
RUN apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main" -y

RUN apt-get update && apt-get install -y --no-install-recommends \
    containerd \
    runc \
    kubelet=1.28.0-00 \
    kubeadm=1.28.0-00 \
    kubectl=1.28.0-00 \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /etc/containerd/

COPY assets/90-override.conf /etc/sysctl.d/90-override.conf
COPY assets/netfilter.conf /etc/modules-load.d/netfilter.conf
COPY assets/config.toml /etc/containerd/config.toml

# Copy the Kairos framework files. We use master builds here for fedora. See https://quay.io/repository/kairos/framework?tab=tags for a list
COPY --from=quay.io/kairos/framework:master_ubuntu-22-lts / /

# Set the Kairos arguments in os-release file to identify your Kairos image
FROM quay.io/kairos/osbuilder-tools:latest as osbuilder
RUN zypper install -y gettext
RUN mkdir /workspace
COPY --from=base /etc/os-release /workspace/os-release
# You should change the following values according to your own versioning and other details
RUN OS_NAME=kairos-vadim-ubuntu \
  OS_VERSION=v1.0.0 \
  OS_ID="kairos" \
  BUG_REPORT_URL="https://github.com/vadimzharov/k8s-kairos/issues" \
  HOME_URL="https://github.com/vadimzharov/k8s-kairos" \
  OS_REPO="quay.io/vadimzharov/kairos" \
  OS_LABEL="latest" \
  GITHUB_REPO="vadimzharov/k8s-kairos" \
  VARIANT="core" \
  FLAVOR="ubuntu" \
  /update-os-release.sh

FROM base
COPY --from=osbuilder /workspace/os-release /etc/os-release

# Activate Kairos services
RUN systemctl enable cos-setup-reconcile.timer && \
          systemctl enable cos-setup-fs.service && \
          systemctl enable cos-setup-boot.service && \
          systemctl enable cos-setup-network.service

## Generate initrd
RUN kernel=$(ls /boot/vmlinuz-* | head -n1) && \
            ln -sf "${kernel#/boot/}" /boot/vmlinuz
RUN kernel=$(ls /lib/modules | head -n1) && \
            dracut -v -N -f "/boot/initrd-${kernel}" "${kernel}" && \
            ln -sf "initrd-${kernel}" /boot/initrd && depmod -a "${kernel}"
RUN rm -rf /boot/initramfs-*

```

To build the base Kairos container image execute docker build command:
```
docker build -t quay.io/vadimzharov/kairos:v1.28.0 -f Dockerfile.00
docker push quay.io/vadimzharov/kairos:v1.28.0
```

Once these commands have completed, you'll have a container image that serves as the base for building an ISO image. This image can be used to install Kairos on a virtual machine (VM) or to upgrade/modify nodes that are already running Kairos.

To create an ISO image from the Kairos container image, you can use the special tool called AuroraBoot, which is available at https://github.com/kairos-io/AuroraBoot. AuroraBoot takes the base image and can bake a cloud-init configuration file into it, if needed. This ISO image can then be used to install Kairos on a VM or bare-metal machine.
If you include a cloud-init configuration file during the ISO image creation process, the OS will boot and configure itself automatically using the provided cloud-init file, requiring no user interaction. Alternatively, if the image is built without a cloud-init configuration file, users can provide this file(s) by accessing SSH or through the WebUI.

As I mentioned above - at this stage I decided to build ISO without providing cloud-init file - to try how it can be provided via WebUI.

To build ISO execute:
```
mkdir build/
docker run -v "$PWD"/build:/tmp/auroraboot -v /var/run/docker.sock:/var/run/docker.sock --rm -ti quay.io/kairos/auroraboot:v0.2.5 --set container_image=quay.io/vadimzharov/kairos:v1.28.0 --set "disable_http_server=true" --set "disable_netboot=true" --set "state_dir=/tmp/auroraboot" --debug
```
After build completes, the ISO image will be available in the `build/iso` directory, file name is **kairos.iso**.

Copy the image to make it available for KVM (`/var/lib/libvirt/images` in my case).

### Create k8s master

Create VM using *virt-install* and install OS. Kairos will boot but not perform any installation tasks - since cloud-init configuration file is missing.
```
virt-install --name k8s-master --memory 4096 --vcpus 2 --disk size=25 --cdrom /var/lib/libvirt/images/kairos-1-28-0.iso --os-variant ubuntu22.10
```

Kairos console after boot:

![Kairos console](/pict/01_Kairos_Console.png)

Now Kairos WebUI is available `http://<vm-ip>:8080`, where you can provide cloud-init configuration file.

Open a web browser and paste cloud-init config file to configure OS. In my case, in addition to default user credentials, since I want to install k8s - I'm adding kubeadm configuration file (which I can use to install k8s) and make `/etc/kubernetes`, `/var/lib/etcd/` and `/var/lib/kubelet` directories persistent (all other directories are ephemeral).

Kairos WebUI:

![Kairos WebUI](/pict/02_Kairos_WebUI.png)

Cloud-init configuration file to configure k8s master node OS is in [kairos_config_master.yaml](kairos_config_master.yaml) file. This file has content of [kubeadm-config.yaml](kubeadm-config.yaml) file (k8s cluster configuration, base64 encoded), my ssh public key - so I can login to the node after configuration, desired hostname to set for the node (k8s-master) and list of partitions I want to be persistent (as described above).

Paste the content of the [kairos_config_master.yaml](kairos_config_master.yaml) file to the WebUI and press install.

Kairos will configure itself and reboot. After reboot you should be able to login to the OS and check content of `/etc/kubernetes/kubeadm-config.yaml` file:
```
[vadim@fedora kairos]$ ssh kairos@192.168.124.240
The authenticity of host '192.168.124.240 (192.168.124.240)' can't be established.
ED25519 key fingerprint is SHA256:E3zo2n2YXuQfAS8jKYaGKVlzB4l8+YkC6YB4B2dKm1o.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.124.240' (ED25519) to the list of known hosts.
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.2.0-31-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

kairos@k8s-master:~$ sudo -i
root@k8s-master:~# cat /etc/kubernetes/kubeadm-config.yaml 
---
apiServer:
    extraArgs:
      authorization-mode: Node,RBAC
    timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: k8s-ubuntu
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
networking:
    dnsDomain: cluster.local
    podSubnet: 10.244.0.0/16
    serviceSubnet: 10.96.0.0/18
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: "/run/containerd/containerd.sock"
---
```

Now the system is ready to install k8s. Execute as root:

```
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --skip-phases=addon/kube-proxy
```

After command completes, wait for CSR to appear and approve it using kubeconfig file available in `/etc/kubernetes/admin.conf`:
```
root@k8s-master:~# export KUBECONFIG=/etc/kubernetes/admin.conf
root@k8s-master:~# kubectl get csr
NAME        AGE   SIGNERNAME                                    REQUESTOR                REQUESTEDDURATION   CONDITION
csr-246l4   15m   kubernetes.io/kube-apiserver-client-kubelet   system:node:k8s-master   <none>              Approved,Issued
csr-r55dt   13s   kubernetes.io/kubelet-serving                 system:node:k8s-master   <none>              Pending
root@k8s-master:~# kubectl certificate approve csr-r55dt
certificatesigningrequest.certificates.k8s.io/csr-r55dt approved
```

Now the k8s cluster has been installed but CNI is missing (this is the reason why node is in NotReady state). 

Copy kubeconfig file to local machine (I did not install helm binaries on Kairos OS), update [cilium-values.yaml](cilium-values.yaml) file with correct k8s API IP address (must be set as VM IP address) and run helm install:
```
[vadim@fedora kairos]$ export KUBECONFIG=kubeconfig
[vadim@fedora kairos]$ kubectl get pods -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-5dd5756b68-qprcl             0/1     Pending   0          20m
kube-system   coredns-5dd5756b68-rw7fm             0/1     Pending   0          20m
kube-system   etcd-k8s-master                      1/1     Running   0          20m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          20m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          20m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          20m
[vadim@fedora kairos]$ helm install cilium cilium/cilium -n kube-system -f cilium-values.yaml 
NAME: cilium
LAST DEPLOYED: Wed Sep 13 10:27:44 2023
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble.

Your release version is 1.14.1.

For any further help, visit https://docs.cilium.io/en/v1.14/gettinghelp
```

Wait for cilium pods to be in the Ready state and ensure node is in Ready state as well:
```
[vadim@fedora kairos]$ kubectl get pods -A
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   cilium-8cjlw                             1/1     Running   0          54s
kube-system   cilium-operator-5d7f984789-g2c6k         1/1     Running   0          54s
kube-system   coredns-5dd5756b68-qprcl                 1/1     Running   0          21m
kube-system   coredns-5dd5756b68-rw7fm                 1/1     Running   0          21m
kube-system   etcd-k8s-master                          1/1     Running   0          21m
kube-system   kube-apiserver-k8s-master                1/1     Running   0          21m
kube-system   kube-controller-manager-k8s-master       1/1     Running   0          21m
kube-system   kube-scheduler-k8s-master                1/1     Running   0          21m
[vadim@fedora kairos]$ kubectl get nodes
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   22m   v1.28.0

```

Reboot the node and ensure all pods are exist and in Running state. If you don't see cilium pods - it means the directory `/var/lib/etcd/` is not persistent (check cloud-init configuration file).


### Adding a new node to the k8s cluster

Next step is to add a new node to my k8s cluster, and I want to make this step fully automated - means all I want is to create VM with ISO attached (one command)- and then see the node is joining to my k8s cluster. To do this I can re-use Kairos container image I built before, but bake cloud-init configuration file inside ISO image - so the node will configure itself and execute command to join to the k8s cluster.

This cloud-init configuration file will be different comparing to configuration file I used for the master node, because I need to:
1. Generate hostname for the node (I want to use this image to provision any number of nodes);
2. Have proper *kubeconfig* file for `kubeadm join` command;
3. Execute `kubeadm join` command if node is not a part of the cluster.

#### Create user and kubeconfig file for nodes to join the k8s cluster

Let's start with **kubeconfig** file content. I will create user in my kubernetes cluster and grant this user permissions to join nodes to the cluster. In this case node provisioning is fully automated, but what I have to do - is to manually approve CSRs (generated by this user) to finally allow node to join to the cluster (this process can be automated as well, but I want control on what nodes are actually joning the cluster).

To generate kubeconfig file for the new user (let's assign username as **join**), login to k8s master node and execute:
```
kairos@k8s-master:~$ sudo -i 
root@k8s-master:~# kubeadm kubeconfig user --client-name=join > kubeconfig.join
root@k8s-master:~# 
```

Now the kubeconfig file for a new user is created as **kubeconfig.join**, but this user does not have any permissions:
```
root@k8s-master:~# KUBECONFIG=kubeconfig.join kubectl get pods -A
Error from server (Forbidden): pods is forbidden: User "join" cannot list resource "pods" in API group "" at the cluster scope
root@k8s-master:~# KUBECONFIG=kubeconfig.join kubectl get cm -A
Error from server (Forbidden): configmaps is forbidden: User "join" cannot list resource "configmaps" in API group "" at the cluster scope
```

To be able to add nodes to the cluster, the user must have permissions to read configmaps in kube-system namespace and create nodes and certificatesigningrequests on the cluster level. Let's grant these permissions:
```
root@k8s-master:~# kubectl create role k8s-join-user-role --verb=get,list --resource=configmaps -n kube-system
role.rbac.authorization.k8s.io/k8s-join-user-role created
root@k8s-master:~# kubectl create rolebinding k8s-join-rolebinding --role=k8s-join-user-role --user=join -n kube-system
rolebinding.rbac.authorization.k8s.io/k8s-join-rolebinding created
root@k8s-master:~# kubectl create clusterrole k8s-join-user-clusterrole --verb=get,list,create --resource=nodes,certificatesigningrequests.certificates.k8s.io
clusterrole.rbac.authorization.k8s.io/k8s-join-user-clusterrole created
root@k8s-master:~# kubectl create clusterrolebinding k8s-join-user-crb --clusterrole=k8s-join-user-clusterrole --user=join
clusterrolebinding.rbac.authorization.k8s.io/k8s-join-user-crb created
```

Now get content of the file **kubeconfig.join** and add it to [kairos_config_node.yaml](kairos_config_node.yaml) file to the `stages.boot.files` section (**base64 encoded**).

To properly encode the content, execute:
```
root@k8s-master:~# base64 -w0 kubeconfig.join 
```

During boot process, Kairos agent will write content of this file to `/etc/kubernetes/kubeconfig.join` and then execute command (described in the next section of the configuration file) `kubeadm join --discovery-file /etc/kubernetes/kubeconfig.join` but only if `/var/lib/kubelet/config.yaml` file does not exist - means it will not try to join node to the cluster if node already was configured and joined to the cluster.

And one more change in [kairos_config_node.yaml](kairos_config_node.yaml) file comparing to [kairos_config_master.yaml](kairos_config_master.yaml) file - node hostname is generated during OS install based on machine UID (laset 4 symbols) - means it will always be unique and I can use the same image to provision as many nodes as I want:
```
stages:
  initramfs:
    - name: "Setup hostname"
      hostname: "node-{{ trunc 4 .MachineID }}"
```

#### Create ISO image to provision new nodes for k8s cluster

Now I need to build a new ISO image with cloud-init configuration file using **AuroraBoot** tool.

Copy file [kairos_config_node.yaml](kairos_config_node.yaml) to `build/` directory and execute:
```
docker run -v "$PWD"/build:/tmp/auroraboot -v /var/run/docker.sock:/var/run/docker.sock --rm -ti quay.io/kairos/auroraboot:v0.2.5 --set container_image=quay.io/vadimzharov/kairos:v1.28.0 --set "disable_http_server=true" --set "disable_netboot=true" --set "state_dir=/tmp/auroraboot" --debug --cloud-config /tmp/auroraboot/kairos_config_node.yaml
```
I'm using the same command as previous , but adding additional argument for AuroraBoot to add *kairos_config_node.yaml* file to install the OS (`--cloud-config`).
Wait for the build to complete and copy `build/iso/kairos.iso` image as **kairos-node.iso**.

#### Provision a new node

Since this image has cloud-init configuration file, all I need is to create a new VM with this ISO file attached:

```
virt-install --name k8s-node01 --memory 4096 --vcpus 2 --disk size=25 --cdrom /var/lib/libvirt/images/kairos-node.iso --os-variant ubuntu22.10
```

New VM will boot with Kairos node ISO attached and Kairos will install itself without any interaction from a user. Node name will be generated automatically, and since kubelet configuration file is absent - node will execute `kubeadm join` command and create CSRs (and requiestor for the CSR is user I created earlier), check on k8s cluster after node created:
```
[vadim@fedora kairos]$ kubectl get csr
NAME        AGE    SIGNERNAME                                    REQUESTOR                REQUESTEDDURATION   CONDITION
csr-gkzsr   93s    kubernetes.io/kube-apiserver-client-kubelet   join                     <none>              Pending
```
Approve the bootstrap CSR:
```
[vadim@fedora kairos]$ kubectl certificate approve csr-gkzsr
certificatesigningrequest.certificates.k8s.io/csr-gkzsr approved
```

Wait for node CSR to appear and approve it (notice node name - it has random 4 symbols based on machine UID):
```
[vadim@fedora kairos]$ kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                REQUESTEDDURATION   CONDITION
csr-gkzsr   2m49s   kubernetes.io/kube-apiserver-client-kubelet   join                     <none>              Approved,Issued
csr-r2smz   9s      kubernetes.io/kubelet-serving                 system:node:node-e8d7    <none>              Pending
[vadim@fedora kairos]$ kubectl certificate approve csr-r2smz
certificatesigningrequest.certificates.k8s.io/csr-r2smz approved
```

That's it - node has been added to the cluster with minimum efforts:
```
[vadim@fedora kairos]$ kubectl get nodes
NAME         STATUS   ROLES           AGE    VERSION
k8s-master   Ready    control-plane   116m   v1.28.0
node-e8d7    Ready    <none>          105s   v1.28.0
```

To add one more node execute the same command (but change VM name for KVM):

```
virt-install --name k8s-node02 --memory 4096 --vcpus 2 --disk size=25 --cdrom /var/lib/libvirt/images/kairos-node.iso --os-variant ubuntu22.10
```

Wait for CSRs to appear and approve it (as I said before, this process can be automated if required):
```
[vadim@fedora kairos]$ kubectl get nodes
NAME         STATUS   ROLES           AGE    VERSION
k8s-master   Ready    control-plane   125m   v1.28.0
node-3a6c    Ready    <none>          101s   v1.28.0
node-e8d7    Ready    <none>          10m    v1.28.0
```


### Upgrading k8s cluster

To upgrade k8s cluster I need to upgrade k8s binaries (previously I built Kairos image using k8s **1.28.0** binaries, but now I want to upgrade to **1.28.1**). Since Kairos is immutable OS, I cannot update packages on existing OS simply by using package manager - I have to build a new image with new k8s binaries installed and then trigger Kairos upgrade process. During this upgrade process Kairos will replace existing *Active* image for the OS, but only if upgrade process finished successfully (and if there are issues during upgrade - revert changes on the previous version) - see this guide for detailed explanation - https://kairos.io/docs/architecture/container/#ab-upgrades .

To build a new base Kairos container image I created new [Dockerfile.01](Dockerfile.01) file - it is exactly the same as [Dockerfile.00](Dockerfile.00), but with k8s binaries **v1.28.1** installed.
Execute docker build command:
```
docker build -t quay.io/vadimzharov/kairos:v1.28.1 -f Dockerfile.01
docker push quay.io/vadimzharov/kairos:v1.28.1
```

Now, after image is available in my registry, I can upgrade Kairos OS on my k8s master. Login to the node, become root and check kubelet version:
```
[vadim@fedora kairos]$ ssh kairos@192.168.124.240
kairos@k8s-master:~$ sudo -i
root@k8s-master:~# kubelet --version
Kubernetes v1.28.0
```

Trigger the upgrade:
```
kairos-agent upgrade --source oci:quay.io/vadimzharov/kairos:v1.28.1
```

Kairos will upgrade active image (replace everything exept content of directoies I set as persistent in cloud-init configuration file) and reboot the node. Wait for node to come back, login and check kubelet version:
```
[vadim@fedora kairos]$ ssh kairos@192.168.124.240
kairos@k8s-master:~$ sudo -i
root@k8s-master:~# kubelet --version
Kubernetes v1.28.1
root@k8s-master:~# 
```

Also notice the output from `kubectl get nodes` command - the master is on kubernetes 1.28.1 but nodes are still on 1.28.0:
```
[vadim@fedora kairos]$ kubectl get nodes
NAME         STATUS   ROLES           AGE    VERSION
k8s-master   Ready    control-plane   140m   v1.28.1
node-3a6c    Ready    <none>          16m    v1.28.0
node-e8d7    Ready    <none>          25m    v1.28.0
```

Upgrade all other nodes by executing `kairos-agent upgrade --source oci:quay.io/vadimzharov/kairos:v1.28.1` command on each node:
```
[vadim@fedora kairos]$ kubectl get nodes 
NAME         STATUS   ROLES           AGE    VERSION
k8s-master   Ready    control-plane   150m   v1.28.1
node-3a6c    Ready    <none>          26m    v1.28.1
node-e8d7    Ready    <none>          35m    v1.28.1
```

### Resetting k8s cluster

To experiment with some k8s features I need to be able to re-install my k8s cluster without touching my bare-metal environment, and since it is my homelab - I don't have access to the console of my bare-metal machines. It means I need to be able to re-install OS without logging to the console (only by accessing it via SSH), which is possible if you configure environment to support network boot (for example). However, Kairos allows users to reset itself to initial state using remote access - see this link for details https://kairos.io/docs/reference/reset/.

Login to k8s master and become root:
```
[vadim@fedora kairos]$ ssh kairos@192.168.124.240
kairos@k8s-master:~$ sudo -i
root@k8s-master:~#
```
Execute the following commands to reset the system:
```
root@k8s-master:~# grub2-editenv /oem/grubenv set next_entry=statereset
root@k8s-master:~# reboot
```

Node will reboot and reset itself. Login to the node and validate:
```
[vadim@fedora kairos]$ ssh kairos@192.168.124.241
kairos@k8s-master:~$ sudo -i
root@k8s-master:~# ls -la /etc/kubernetes/
total 12
drwxr-xr-x 3 root root 4096 Sep 13 18:04 .
drwxr-xr-x 1 root root  360 Sep 13 18:04 ..
-rw-r--r-- 1 root root  673 Sep 13 18:04 kubeadm-config.yaml
drwxr-xr-x 2 root root 4096 Sep  6 21:05 manifests
root@k8s-master:~# ls -la /var/lib/etcd
total 4
drwxr-xr-x 2 root root 4096 Sep 13 18:04 .
drwxr-xr-x 1 root root  280 Sep 13 18:04 ..
root@k8s-master:~# kubelet --version
Kubernetes v1.28.0
root@k8s-master:~# 
```

The state has been reverted - all directories cleaned up (even if they are persistent), OS was provisioned based on initial OS container image - kubelet is on v1.28.0 version again.

Install k8s again, repeating all steps executed before:
```
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --skip-phases=addon/kube-proxy
```
Obtain kubeconfig from `/etc/kubernetes/admin.conf` and use it to approve CSR.

Then update cilium-values.yaml file with correct kubernetes master IP (address may change) - field `k8sServiceHost` and install cilium via helm:
```
[vadim@fedora kairos]$ helm install cilium cilium/cilium -n kube-system -f cilium-values.yaml 
```

Ensure all pods are running:
```
[vadim@fedora kairos]$ kubectl get pods -A
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   cilium-5vbvq                             1/1     Running   0          73s
kube-system   cilium-operator-65b94c4cc6-nbcdn         1/1     Running   0          73s
kube-system   coredns-5dd5756b68-2rcds                 1/1     Running   0          30m
kube-system   coredns-5dd5756b68-b9mqr                 1/1     Running   0          30m
kube-system   etcd-k8s-master                          1/1     Running   0          31m
kube-system   kube-apiserver-k8s-master                1/1     Running   0          31m
kube-system   kube-controller-manager-k8s-master       1/1     Running   0          31m
kube-system   kube-scheduler-k8s-master                1/1     Running   0          31m
```

Now I need to reset k8s node - to add it to the cluster. But before I need to create user `join` with `kubeconfig` file and apply roles and clusterroles by repeating steps I executed before.
On k8s master node:

```
root@k8s-master:/home/kairos# export KUBECONFIG=/etc/kubernetes/admin.conf 
root@k8s-master:/home/kairos# kubeadm kubeconfig user --client-name=join > kubeconfig.join 
root@k8s-master:/home/kairos# kubectl create role k8s-join-user-role --verb=get,list --resource=configmaps -n kube-system
role.rbac.authorization.k8s.io/k8s-join-user-role created
root@k8s-master:/home/kairos# kubectl create rolebinding k8s-join-rolebinding --role=k8s-join-user-role --user=join -n kube-system
rolebinding.rbac.authorization.k8s.io/k8s-join-rolebinding created
root@k8s-master:/home/kairos# kubectl create clusterrole k8s-join-user-clusterrole --verb=get,list,create --resource=nodes,certificatesigningrequests.certificates.k8s.io
clusterrole.rbac.authorization.k8s.io/k8s-join-user-clusterrole created
root@k8s-master:/home/kairos# kubectl create clusterrolebinding k8s-join-user-crb --clusterrole=k8s-join-user-clusterrole --user=join
clusterrolebinding.rbac.authorization.k8s.io/k8s-join-user-crb created
```

So I have my new `kubeconfig.join` and I need to provide this file to my k8s node during node bootstrap. Since I have access to the node (and it is still configured as a part of my previous k8s cluster), I can login to the node and update cloud-init configuration file - and then reset the node (Kairos will keep changes in the configuration file). The file I need to update is in the path `/oem/90_custom.yaml` - this is the default path for Kairos baked cloud-init configuration file (but kairos-agent will read and apply configuration from all files in this path).

Login to the node and update `/oem/90_custom.yaml` file - replace `stages.boot.files.[0]content` with the new base64 decoded **kubeconfig.join** file.

After that run commands to reset the system:

```
[vadim@fedora kairos]$ ssh kairos@192.168.124.77
kairos@node-c5a5:~$ sudo -i
root@node-c5a5:~# grub2-editenv /oem/grubenv set next_entry=statereset
root@node-c5a5:~# reboot
```

That's it - after reset - node will try to join to the cluster using updated kubeconfig.join file and you need to approve CSRs:
```
[vadim@fedora kairos]$ kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                REQUESTEDDURATION   CONDITION
csr-lq7b6   3m42s   kubernetes.io/kube-apiserver-client-kubelet   join                     <none>              Pending
[vadim@fedora kairos]$ kubectl certificate approve csr-lq7b6
certificatesigningrequest.certificates.k8s.io/csr-lq7b6 approved
[vadim@fedora kairos]$ kubectl get csr | grep Pend
csr-p99rj   3s      kubernetes.io/kubelet-serving                 system:node:node-5d9d    <none>              Pending
[vadim@fedora kairos]$ kubectl certificate approve csr-p99rj
certificatesigningrequest.certificates.k8s.io/csr-p99rj approved
```

And after a couple of minutes the node will become Ready. Notice - the kubelet version was also reverted to v1.28.0 - because Kairos agent reverted node to the initial container image.

```
[vadim@fedora kairos]$ kubectl get nodes
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   51m   v1.28.0
node-5d9d    Ready    <none>          83s   v1.28.0
```

## Conclusion

Kairos is a minimalistic and immutable operating system that simplifies the installation and management of Kubernetes (k8s) clusters. It offers a high degree of customization, allowing users to build their own images or customize running ones via cloud-init configuration files. These configurations can include adjustments to sysctl settings and the installation or update of required kernel modules if required. 
What I very like is Kairos flexibility. You have the freedom to choose their preferred base operating system, whether it's Ubuntu, Fedora, or another popular distribution, as well as container runtime - such as containerd or CRI-O. This versatility ensures that Kairos can seamlessly integrate into diverse k8s cluster environments.

Additionally, Kairos offers a lightweight and automated approach to re-installing your environment. This process is designed to be straightforward and does not necessitate changes to your infrastructure, eliminating the need for network boot configurations.









