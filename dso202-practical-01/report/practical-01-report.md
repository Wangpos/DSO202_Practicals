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
This stage is important because we can know how to directly query, inspect and verify those running processes and components in the cluster. It is also important to know how to check the status of the cluster and its components.

1/ One of the methods is checking where the control plane is as shown below:

![alt text](<../evidence/img/Step 1. Ask the cluster where its control plane is..png>)

So as we can see the information about the control plane where it is running at `http://127.0.0.1:55489` and the coreDNS is running at `http://127.0.0.1:55489/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy`.

2/ Listing the nodes in the cluster and checking their status as shown below:

![alt text](<../evidence/img/Step 2. List the nodes.png>)

Form the above image, we can conclude that only the control plane node is assigned the role.

The worker nodes are assigned `<none>` for the roles as kind assigned no roles for the worker nodes. It gets its role when we deploy workloads to the cluster. 

3/ Now lets use `-o wide` option in the command line to get more information about the nodes in the cluster as shown below:

![alt text](<../evidence/img/Step 3. Add columns to the same query.png>)

Can also be used for getting pod information as well.

4/ This step uses `kubectl describe` to inspect `worker-node-1`, verifying its custom metadata labels, available hardware resources (CPU/memory), and currently running Pods as shown below:

![alt text](<../evidence/img/Step 4. Read one node in detail and locate the labels applied by Listing 1..png>)

Lets see a clean list of lables that is being assigned to worker-node-1 as shown below:

![alt text](<../evidence/img/Step5 retriving node detailes.png>)

5/ Getting the namesapaces in the cluster and checking their status as shown below:

![alt text](<../evidence/img/Step 5. List the namespaces that exist before any work is done..png>)

6/ Listing out the components of control plane which runs as pod in kube-system as shown below:

![alt text](<../evidence/img/Step 6. List the control-plane components. They run as Pods in kube system.png>)

**Three Observations that are made are:**

* **Control Plane Components:** `etcd`, `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` run only once on the control-plane node to manage the cluster.
* **Cluster-Wide Networking:** `kube-proxy` and `kindnet` run on all three nodes to handle networking everywhere across the cluster.
* **Core DNS Service:** `coredns` runs twice on the control-plane node because it has special permission to run there, unlike ordinary application Pods.











