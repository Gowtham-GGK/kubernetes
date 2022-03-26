# Build Kubernetes Cluster from Scratch

## 1. Provision Infrastructure on AWS


### 1.1 Create Instances

  - Create instances (1 master & 2 workers) in AWS with below
    - Ubuntu 
    - t2.medium

  - While creating it will ask for key-pair, if you have any already created use that or create one. Please make sure to download it because we use this to connect to instance.


### 1.2 Test SSH Connection

  - First move the prive key-pair(.pem) to standard location **~/.ssh**
  - After moving, we need to restrict that file with permission **400** (Read permission for owner only)

      ```shell
      chmod 400 ~/.ssh/<key-name>.pem
      ```
  - Now we can connect to instances which are created on AWS. For that get ip public ip address of all the instances. use the following command to connect 

      ```shell
      ssh -i ~/.ssh/<key-name>.pem ubuntu@<ip-addr>
      ```

### 1.3 Install Kubeadm Toolbox

  #### Prerequisites

  - A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
  - 2 GB or more of RAM per machine (any less will leave little room for your apps).
  - 2 CPUs or more.
  - Full network connectivity between all machines in the cluster (public or private network is fine).
  - Unique hostname, MAC address, and product_uuid for every node. See [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-mac-address) for more details.
  - Certain ports are open on your machines. See [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)  for more details.
    <details> 
      <p>

      - In AWS, we need to add Inbound rules to the Instance's security group.
      - While adding Inbound rules, it will ask for source give 
        - 0.0.0.0/0 for All(API server & Node port services)
        - 172.31.0.0/16 (IPv4 CIDR of your instance VPC)  for all others. why because all our pods can be accessible only within our cluster not outside cluster

      - #### Control plane

        | Port Range | Purpose | Used By |
        | ---------- | ------- | ------- |
        | 6443 | Kubernetes API server | All |
        | 2379-2380 | etcd server client API | kube-apiserver, etcd |
        | 10250 | Kubelet API | Self, Control plane |
        | 10259 | kube-scheduler | Self |
        | 10257 | kube-controller-manager	 | Self |

      - #### Worker node(s)
        
        | Port Range | Purpose | Used By |
        | ---------- | ------- | ------- |
        | 10250 | Kubelet API | Self, Control plane |
        | 30000-32767 | NodePort Services†	 | All |
    

      

      </p>
    </details>

  - Swap disabled. You **MUST** disable swap in order for the kubelet to work properly. 
    <details>
    <p>
      
      - why beacuse k8 scheduler determines best available node on which to deploy newly created pods. if memory swapping is allowd, it will lead to performance and stability issues in kubernetes. 

      > **SWAPPING** Memory swapping is a computer technology that enables an operating system to provide more memory to a running application or process than is available in physical random access memory (RAM). When the physical system memory is exhausted, the operating system can opt to make use of memory swapping techniques to get additional memory

      ```shell
      sudo swapoff -a
      ```
    </p>
    </details>

  #### Install container runtime (containerd)

  - If you don't specify a runtime, kubeadm automatically tries to detect an installed container runtime by scanning through a list of well known Unix domain sockets. The following table lists container runtimes and their associated socket paths:

    | Runtime	| Path to Unix domain socket |
    | ------- | -------------------------- |
    | Docker | /var/run/dockershim.sock |
    | containerd |	/run/containerd/containerd.sock |
    | CRI-O |	/var/run/crio/crio.sock |

  - To check pre-requisites and configure, insstall **containerd** run this [shell script ](./install-containerd.sh) on both master & worker node(s)

    > Make sure to provide execute permissions for the script to work

  #### Install kubeadm, kubelet and kubectl

  - **kubeadm**: the command to bootstrap the cluster.

  - **kubelet**: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

  - **kubectl**: the command line util to talk to your cluster.

  - Install above threee k8 components using this [shell script](./install-kubeadm-kubelet-kubectl.sh)

    > Make sure all the three having same version

  

### 1.4 Initialise Cluster with kubeadm

  - Initialise cluster by running below command only on master 

    ```shell
    sudo kubeadm init
    ```

    above command will do the below
    -  checks to validate system state before running any changes and download/pull images required to start k8 components -> **preflight phase**

    - generates ***/etc/kubernetes*** folder and self signed CA to setup identities for each component -> **certs Phase**
    - put generated certificates in ***/etc/kubernetes/pki*** -> **certs Phase**

    - make neccessary configuration files for all clients(controlmanager, kubelet, scheduler) to connect to API server ***/etc/kubernetes*** -> **kubeconfig phase**

    - configure kubelet environment ***/var/lib/kubelet/config.yaml*** and start kubelet -> **kubelet-start**

    - generates static pod manifests for API server, control manager and scheduler in ***/etc/kubernetes/manifests*** -> **controlplane phase**
    
    - installs a DNS server and kube-proxy add-on components via the API server -> **addons Phase**

### 1.5 Connect to Cluster

  - For connecting to cluster, API server is the entry point. kubeadm already generated client certificates which we can use to adminster cluster

  - kubeadm also generates **kubeconfig** file which has details like API server address, admin user client certificate and private key

  - kubectl tool is used to connect to cluster

  - For example we use below command to get status of node. we pass kubeconfig file to authenticate

      ```
      sudo kubectl get node --kubeconfig /etc/kubernetes/admin.conf
      ```

    But we cannot pass --kubeconfig flag and sudo everytime to run commands. 

    To get rid of them we can place the kubeconfig file in the path where kubectl looks for it i.e., ***$HOME/.kube/config*** and change the ownership of that file to current user.

    Now we can kubectl commands without sudo and --kubeconfig flag

      ```
      kubectl get node
      ```

