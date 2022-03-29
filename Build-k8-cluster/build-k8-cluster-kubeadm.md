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

      ```shell
      sudo kubectl get node --kubeconfig /etc/kubernetes/admin.conf
      ```

    But we cannot pass --kubeconfig flag and sudo everytime to run commands. 

    To get rid of them we can place the kubeconfig file in the path where kubectl looks for it i.e., ***$HOME/.kube/config*** and change the ownership of that file to current user.

    Now we can kubectl commands without sudo and --kubeconfig flag

      ```shell
      kubectl get node
      ```

### 1.6 Namespaces

- In Kubernetes, namespaces provides a mechanism for isolating groups of resources within a single cluster. Names of resources need to be unique within a namespace, but not across namespaces. Namespace-based scoping is applicable only for namespaced objects (e.g. Deployments, Services, etc) and not for cluster-wide objects (e.g. StorageClass, Nodes, PersistentVolumes, etc).

  > Why we need Namespaces?
  >  - Resources grouped in namespace
  >  - Conflicts: Many teams, same application
  > - Resource sharing: staging and development
  >  - Access and Resource limits on namespaces

- By default we get four namespaces kube-system, kube-public,kube-node-lease, default. 

  ```shell
  kubectl get namespace
  ```

- kubectl fetches all pods from **default** namespace by default. if we want to fetch any other namespace details

  ```shell
  kubectl get ns -n kube-system
  ```

### 1.7 Container Network Interface(CNI)

  - For Pod to Pod communication, there is no builtin solution provided by kubernetes. Instead it creates an Interface. so whatever plugin implements this interface can be used in k8 cluster.

    > Requirements for CNI plugins
    > - Every pod gets its own unique IP addr 
    > - Pods on same node can communicate each other usig that IP addr
    > - Pods on different nodes can communicate each other usig that IP addr without NAT(Network Address Translation)

  - There are many CNI plugins available in market, we use **weavenet** 

  - Pods will get their own private network and its own IP address rane or CIDR block for each node. so pods within node can communicate with that IP address

    > IP Address range or CIDR block provided by CNI plugins should not overlap with cluster VPC IP Address range or CIDR block 

  - CNI plugins like **weavenet** are deployed as pods in all nodes. They form a group and can directly talk to each other. so pods from different node can communicate with this plugins.

  #### Installation

  - To install weavenet , its a single line command. It will download the manifest file required to deploy it as pod.

    ```shell
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    ```

  - Instead we can download manifest in our cluster and apply manually. to download it use below command


    ```shell
    wget "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')" -o weave.yml
    ```

  - you can customize this file, like changing default CIDR block or changing port it runs on. After you customize your cahnges. Now we can install CNI pligin

    ```shell
    kubectl apply -f <file_path>.yaml
    ```

    Now our master node and core-DNS pod should be in ready state, so check statsu of then

    ```shell
    kubectl get node

    kubectl get pod -n kube-system
    ```

  - To check detailed information about any pod

    ```shell
    kubectl describe <name-of-pod> -n <namespace-name>
    ```
  
  - To get all list of pods with details like IP Address and which node it is running

    ```shell
    kubectl get node -n <namespace> -o wide
    ```

### 1.8 Join Worker nodes to cluster

  Now everything is ready, but we have one node cluster. lets join worker nodes to cluster.

  - Note that we already installed containerd, kubeadm, kubelet, kubectl on our two nodes. To join worker nodes to cluster, we need to run a command which was generated while running ***kubeadm init*** command

  - If we don't have that we can get that command by executing below command in master

    ```shell
    kubeadm token create --print-join-command

    output: kubeadm join <ipaddress:port> --token <token> --discovery-token-ca-cert-hash <hash>
    ```

  - Run above command generated in the output on our worker nodes. This will join worker nodes with cluster (use sudo)

    ```shell
    sudo kubeadm join <ipaddress:port> --token <token> --discovery-token-ca-cert-hash <hash>
    ```

  - In Background, kubelet will schedule pods if needed without our involvement. Check status of node and pods in master

    ```shell
    kubectl get node -o wide

    kubectl get pod -A -o wide
    ```

    > kube-proxy and weavenet pods will be scheduled automatically in worker nodes as they are daemon sets. Daemon sets will schedule a pod in each single node in a cluster.

  - so after this the weave-net in all nodes should communicate to each other. lets check that out

    ```shell
    kubectl get pod -A -o wide | grep weave-net

    kubectl logs <weave-net-pod-name> -n kube-system -c weave
    ```

  - you can see some errors, as weave-net is unable to connect to other worker nodes and master node in port **6783**. This happens because port **6783** is not opened on any of our nodes. To rectify it just open this port. refer 1.3 for opening port in AWS instances.

  - Now check the logs in all nodes

    ```shell
    kubectl get pod -A -o wide | grep weave-net

    kubectl logs <weave-net-pod-name> -n kube-system -c weave
    ```

  - To check the status of weave-net and whether they discovered each other

    ```shell
    kubectl get pod -n kube-system -o wide | grep weave-net

    kubectl exec -n kube-system <weave-pod-name> -c weave -- /home/weave/weave --local status
    ```

### 1.9 Deploy Test Application

  - Now everything is ready and weave-net also communicating with each other. we can test by deploying some applicationns

    ```shell
    kubectl run <pod-name> --image=<image-name>

    kubectl get pod -o wide
    ```

    Check where it is scheduled, IP address of pod e.t.c.,
