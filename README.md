# Kubernetes Cluster Setup with Docker, NFS, and MPI

This guide provides comprehensive instructions for setting up a Kubernetes cluster on two nodes using Docker and NFS for shared storage. 

It covers the installation of Docker, Kubernetes components, the configuration of an NFS server, and concludes with a practical exercise in running an MPI (Message Passing Interface) application across the cluster. 

This setup is particularly useful for those looking to explore distributed computing and parallel processing with MPI in a Kubernetes environment.

## Prerequisites

- Two Ubuntu machines (nodes) with sudo privileges.
- An active internet connection on both nodes.

## 1. Install Docker and Docker Compose

Docker must be installed on both nodes. If Docker is already installed, you can skip this section.

For additional details on setting up Docker, you can refer to [this repository](https://github.com/guswns531/mpi-docker).

## 2. Install Kubernetes Components

The following steps are for installing Kubernetes version 1.27.

**These steps must be performed on both nodes.**

### 2.1 Install Required Packages
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### 2.2 Add Kubernetes APT Repository
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

### 2.3 Install Kubernetes Components
```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2.4 Disable Swap
To ensure Kubernetes functions correctly, swap must be disabled. The second command modifies the node configuration, so if you plan to remove Kubernetes before restarting, you can skip it.
```bash
swapoff -a
```
** Attention ** 
```bash
# To disable swap on startup, modify /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 3. Install cri-dockerd

cri-dockerd is required to use Docker as the container runtime for Kubernetes since Kubernetes deprecated direct Docker support starting with version 1.20.

**Install cri-dockerd on both nodes.**

Refer to [this blog](https://www.mirantis.com/blog/how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim) for additional details.

### 3.1 Download and Install cri-dockerd
```bash
sudo su
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd-0.3.14.amd64.tgz
tar -xf cri-dockerd-0.3.14.amd64.tgz
sudo mv ./cri-dockerd/cri-dockerd /usr/local/bin/ 
rm -rf ./cri-dockerd

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
systemctl status cri-docker.socket
```

## 4. Initialize Kubernetes Cluster

Before proceeding, confirm the IP addresses of both nodes, as you will need them in later steps.

### 4.0 Check and Remember Node IPs

1. **Check the IP address on each node**:
   On **Node1** (Master Node):
   ```bash
   hostname -I
   ```
   Take note of the IP address; this will be referred to as **Node1_IP**.

   On **Node2** (Worker Node):
   ```bash
   hostname -I
   ```
   Take note of the IP address; this will be referred to as **Node2_IP**.

Now, proceed with the steps below on **Node1 (Master Node)**.

### 4.1 Initialize Kubernetes Cluster

Use the `--cri-socket unix:///var/run/cri-dockerd.sock` option to specify cri-dockerd, and the `--pod-network-cidr=192.168.0.0/16` option for the Calico CNI that will be installed later.

```bash
sudo kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr=192.168.0.0/16
```

If successful, you should see:

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join Node1_IP --token <token> \
        --discovery-token-ca-cert-hash <hash>
```

**Note:** Save the `join` command with the token and other details, as it will be needed to register the worker nodes.

### 4.2 Configure kubectl

This step should be done under a regular user account, not as root.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the setup:

```bash
kubectl get nodes
```

## 5. Join Worker Node to the Cluster

Run the following command on the **worker node** (Node2) to join it to the cluster.

**Note:** Ensure to include `--cri-socket unix:///var/run/cri-dockerd.sock` as you did during the master node setup.

```bash
kubeadm join Node1_IP --cri-socket unix:///var/run/cri-dockerd.sock --token <token> --discovery-token-ca-cert-hash <hash>
```

Example:

```bash
kubeadm join 10.10.10.10:6443 --cri-socket unix:///var/run/cri-dockerd.sock --token kxsoro.qn88d098x5ifyw20 \
        --discovery-token-ca-cert-hash sha256:4fec907958f294e349f2a86f5044c8f1fc9c01998dc19f6457751b83fc2cf9bd 
```

If there are issues, ensure that the firewall is not blocking the connection.

## 6. Install Calico Network Plugin

Install the Calico network plugin on the **master node**.

Refer to the [Calico documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart) for more details.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml
watch kubectl get pods -n calico-system
```

After confirming that the pods are running, exit the watch command with `Ctrl + C`.

## 7. Verify Node Status

On the **master node**, verify the status of the nodes:
```bash
kubectl get nodes
```

Both nodes should be listed as "Ready":
```bash
NAME   STATUS   ROLES           AGE    VERSION
node1   Ready    control-plane   114s   v1.27.16
node2   Ready    <none>          89s    v1.27.16
```

You can further verify by running a test pod:
```bash
kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml
kubectl get pods
kubectl delete pod nginx
```

Here’s the revised version with instructions on configuring the NFS server to include `Node1_IP` and `Node2_IP`:

---

## 8. Set Up NFS for Shared Storage

### 8.1 Install NFS Server Container on the Master Node

We'll set up an NFS server on the master node using a Docker container from [erichough/nfs-server](https://hub.docker.com/r/erichough/nfs-server/).

1. **Create and configure the NFS directory**:
    ```bash
    sudo mkdir -p /nfs-export/mpi-dir/
    sudo chown -R $USER:$USER /nfs-export
    sudo chmod a+w /nfs-export/mpi-dir/
    
    sudo modprobe nfs nfsd rpcsec_gss_krb5
    ```

2. **Run the NFS server container**:

    Before running the NFS server container, ensure you include the IP addresses of both nodes (`Node1_IP` and `Node2_IP`). Modify the `NFS_EXPORT_0` environment variable accordingly. 

    ```bash
    docker run --name nfs-server --rm -d \
    -v /nfs-export:/nfs-export \
    -e NFS_EXPORT_0='/nfs-export $YOUR_SUBSET(rw,root_squash,no_subtree_check,async,fsid=0)' \
    --privileged \
    -p 2049:2049 \
    erichough/nfs-server
    ```
    
    For example, if `Node1_IP` is `10.10.15.51` and `Node2_IP` is `10.10.15.52` and your nodes are within a specific subnet (e.g., `10.10.15.0/24`), you can specify that subnet:
    ```bash
    docker run --name nfs-server --rm -d \
    -v /nfs-export:/nfs-export \
    -e NFS_EXPORT_0='/nfs-export 10.10.15.0/24(rw,root_squash,no_subtree_check,async,fsid=0)' \
    --privileged \
    -p 2049:2049 \
    erichough/nfs-server
    ```

To test access from the worker node:

1. **On Node2** (Worker Node):
   ```bash
   mkdir /mpi-dir
   sudo mount -v -t nfs4 Node1_IP:/mpi-dir /mpi-dir
   echo "Test" > /mpi-dir/testfile
   ```

   This process may take some time.

2. **Unmount and Clean Up**:
   ```bash
   umount /mpi-dir
   rm -rf /mpi-dir
   ```

## 9. Kubernetes Setup

### 9.1 Set Up Persistent Volumes

Create a Persistent Volume (PV) and a Persistent Volume Claim (PVC) to manage shared storage:

1. **Create a `nfs-pv-pvc.yaml` file**:
    Replace `<NODE1_IP>` with the IP address of your master node.
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfs-pv
    spec:
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteMany
      mountOptions:
        - nfsvers=4.1
      nfs:
        path: /mpi-dir
        server: <NODE1_IP>
      persistentVolumeReclaimPolicy: Retain
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-pvc
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 10Gi
      volumeName: nfs-pv
    ```

2. **Apply the configuration**:
    ```bash
    kubectl apply -f nfs-pv-pvc.yaml 
    ```

### 9.2 Deploy the Application

1. **Deploy an application using the shared storage**:
   We'll use the `guswns531/mpi-docker:v1` image, which you created in [this repository](https://github.com/guswns531/mpi-docker).

   Before deploying, make sure to obtain the hostnames of your nodes using the following command:
   ```bash
   kubectl get nodes
   ```
   Use the hostnames retrieved from this command in the `nodeSelector` fields below.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mpi-deployment-1
      labels:
        app: mpi
    spec:
      replicas: 2  # Number of pod replicas to be created for this deployment
      selector:
        matchLabels:
          app: mpi
      template:
        metadata:
          labels:
            app: mpi  
        spec:
          containers:
          - name: mpi-node
            image: guswns531/mpi-docker:v1  # Custom Docker image used for MPI computations
            volumeMounts:
            - name: workspace
              mountPath: /mpi-dir  # Mounts the NFS shared directory inside the container
            ports:
            - containerPort: 22  # SSH port used for MPI communication between nodes
          volumes:
          - name: workspace
            persistentVolumeClaim:
              claimName: nfs-pvc  # Binds the NFS Persistent Volume Claim to this deployment
          nodeSelector:
            kubernetes.io/hostname: <Node1_Hostname>  # Replace with the hostname of Node1 (master)
          tolerations:
          - key: "node-role.kubernetes.io/control-plane"
            operator: "Exists"
            effect: "NoSchedule"  # Allows scheduling on control-plane nodes despite NoSchedule taint
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mpi-deployment-2
      labels:
        app: mpi
    spec:
      replicas: 2  # Number of pod replicas to be created for this deployment
      selector:
        matchLabels:
          app: mpi  
      template:
        metadata:
          labels:
            app: mpi  
        spec:
          containers:
          - name: mpi-node
            image: guswns531/mpi-docker:v1  # Custom Docker image used for MPI computations
            volumeMounts:
            - name: workspace
              mountPath: /mpi-dir  # Mounts the NFS shared directory inside the container
            ports:
            - containerPort: 22  # SSH port used for MPI communication between nodes
          volumes:
          - name: workspace
            persistentVolumeClaim:
              claimName: nfs-pvc  # Binds the NFS Persistent Volume Claim to this deployment
          nodeSelector:
            kubernetes.io/hostname: <Node2_Hostname>  # Replace with the hostname of Node2 (worker node)
    ```

Replace `<Node1_Hostname>` and `<Node2_Hostname>` with the actual hostnames obtained from `kubectl get nodes`. This will ensure the deployments are correctly scheduled on the appropriate nodes.

2. **Apply the deployment**:
   ```bash
   kubectl apply -f mpi-deployment.yaml 
   ```

3. **Verify the setup**:
   ```bash
   kubectl get pods -o wide
   ```
   You should see the pods running across the nodes with access to shared storage.
   ```
   NAME                                READY   STATUS    RESTARTS   AGE   IP                NODE   NOMINATED NODE   READINESS GATES
   mpi-deployment-1-648bcf9c87-7fkfz   1/1     Running   0          8s    192.168.181.17    ns01   <none>           <none>
   mpi-deployment-1-648bcf9c87-xxp2w   1/1     Running   0          8s    192.168.181.18    ns01   <none>           <none>
   mpi-deployment-2-cd8fbcf5-bkdv7     1/1     Running   0          8s    192.168.205.214   ns02   <none>           <none>
   mpi-deployment-2-cd8fbcf5-whclg     1/1     Running   0          8s    192.168.205.215   ns02   <none>           <none>
   ```
   We will use the pod IPs listed above for the next steps. Be sure to use these IPs when accessing the pods via SSH and for testing the MPI application.

### 9.3 Test MPI Application

1. **Create a hostfile**:
   ```bash
   kubectl get pods -o wide --no-headers | awk '{print $6" slots=1"}' > /nfs-export/mpi-dir/hosts
   cat /nfs-export/mpi-dir/hosts
   ```
   The hostfile should look like this:
   ```
   192.168.181.17 slots=1
   192.168.181.18 slots=1
   192.168.205.214 slots=1
   192.168.205.215 slots=1
   ```

2. **Write the `hello_mpi.c` program**:
   ```c
   #include <mpi.h>
   #include <stdio.h>
   #include <unistd.h>

   int main(int argc, char** argv) {
       MPI_Init(&argc, &argv);

       int world_rank;
       MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

       int world_size;
       MPI_Comm_size(MPI_COMM_WORLD, &world_size);

       char hostname[256];
       gethostname(hostname, 256);
       
       printf("Hello from rank %d out of %d processors on %s\n", world_rank, world_size, hostname);

       MPI_Finalize();
       return 0;
   }
   ```

   Copy the file to the shared NFS directory:
   ```bash
   cp hello_mpi.c /nfs-export/mpi-dir
   ```

3. **Compile and run the MPI application**:
   
   First, select one of the pod IPs obtained from the earlier steps. For example, if you choose `192.168.181.17` as your pod IP, proceed with the following steps:

   ```bash
   ssh mpi@192.168.181.17
   # password: mpi
   ```

   Once logged in, compile the MPI application:

   ```bash
   mpicc -o /mpi-dir/hello_mpi /mpi-dir/hello_mpi.c
   ```

   Then, run the MPI application using the hostfile located in the shared NFS directory:

   ```bash
   mpirun -np 4 -hostfile /mpi-dir/hosts /mpi-dir/hello_mpi
   ```

   Expected output:

   ```bash
   Hello from rank 0 out of 4 processors on mpi-deployment-1-648bcf9c87-7fkfz
   Hello from rank 1 out of 4 processors on mpi-deployment-1-648bcf9c87-xxp2w
   Hello from rank 2 out of 4 processors on mpi-deployment-2-cd8fbcf5-bkdv7
   Hello from rank 3 out of 4 processors on mpi-deployment-2-cd8fbcf5-whclg
   ```

This version gives clear guidance on selecting a pod IP and then proceeding with the MPI application compilation and execution, ensuring users understand each step.

## 10. Reset Kubernetes (Optional)

If you need to reset the Kubernetes setup on both nodes:
```bash
kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock
docker stop nfs-server
```
