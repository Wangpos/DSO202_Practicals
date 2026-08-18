#  DSO202 - Practical 01 Report
## Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

### Stage 1 - Creating the Three-Node Cluster
___
1/ Cluster Configuration<br>
Created the cluster configuration file by copying the multi-node specifications from `DSO202_Practical1_Manifests.md` into `cluster/kind-cluster.yaml`.

What is there in the configuration file:
the code configures a local Kubernetes cluster named **dso202** using **Kind**.

It sets up:

* **Networking:** Localhost access (`127.0.0.1`), plus custom IP ranges for Pods and Services.
* **3 Nodes:** 1 control plane node and 2 worker nodes running Kubernetes `v1.36.1`.
* **Port Forwarding:** Exposes host port `30080` to access internal cluster services.

2/ Cluster Creation
Executed the kind command to create the 3-node Kubernetes cluster using the prepared configuration file which is there in manifest as shown below:

![alt text](<../evidence/img/Creating the Three-Node Cluster.png>)

So the cluster is created successfully and also checked whether it exist or not iin Second command line, which exist as `dso202`. I also checked the nodes in the cluster in third command line, which shows the 3 nodes in the cluster.

After that the current context is configured to use the newly created cluster, which is shown below:

![alt text](<../evidence/img/kubectl configing current context.png>)

So the creation of the cluster is successful and the current context is configured to use the newly created cluster.

### Stage 2 - Inspecting the Cluster and Its Components
____



