# lke-vlan
Provides an example for how to register worker nodes to a VLAN for LKE.  You can find the related guide for this repo here: https://patnordstrom.medium.com/private-networking-for-lke-clusters-and-dependent-systems-using-vlans-on-akamai-cloud-compute-1fdd5b3f09ed


excerpt

# Pre-Requisites Before Deploying

Here is a checklist of things you will need in order to complete the exercise in this article:

    Clone the GitHub repository here — https://github.com/patnordstrom/lke-vlan
    You will need a Linode account. There’s a link on this page to register and get free credits to start — https://www.linode.com/docs/products/platform/get-started/
    Once you have your account created you will need to spin up an LKE cluster with at least 2 nodes for testing — https://www.linode.com/docs/products/compute/kubernetes/get-started/
    You will also need Kubernetes CLI installed locally as well to interact with the cluster. The article above covers how to get this setup.
    The DaemonSet we are deploying to the cluster uses the Linode API so you will need a Personal Access Token (PAT) to add to the cluster as a secret — https://www.linode.com/docs/products/tools/api/guides/manage-api-tokens/. NOTE: The only permissions needed for this script is Read/Write on the Linodes object. See example below.

# Deploying the DaemonSet

Before we begin, ensure you have a running LKE cluster with at least 2 nodes (mine has 3 nodes in this example) and that your Kubernetes CLI can reach the cluster as well.

✅ Kubernetes Cluster is running
You can view your cluster within Linode Cloud Manager to check your cluster status

✅ Kubernetes CLI is connected

[pnordstrom]$ kubectl get nodes
NAME                            STATUS   ROLES    AGE   VERSION
lke177234-257574-5667fb420000   Ready    <none>   20d   v1.29.2
lke177234-257574-5fa4b98a0000   Ready    <none>   8d    v1.29.2
lke177234-257574-64a26f810000   Ready    <none>   20d   v1.29.2

Updating the Deployment YAML

Next we need to update the deployment.yaml with the following:

    REQUIRED — We need to add our Personal Access Token we generated earlier to the Secret resource. This needs to be a base64 encoding of the access token you generated earlier (see example below). NOTE: the access key below has been revoked before the publishing of this article.

[pnordstrom]$ echo -n c50cc51a2a8a65c6150f7608433d03378553670de466f0d9f905db45a6906eab | base64
YzUwY2M1MWEyYThhNjVjNjE1MGY3NjA4NDMzZDAzMzc4NTUzNjcwZGU0NjZmMGQ5ZjkwNWRiNDVhNjkwNmVhYg==

    OPTIONAL — We need to update the ConfigMap with our VLAN name and an RFC 1918 CIDR block we want to use. NOTE: The default CIDR in the deployment.yaml isn’t necessarily the best choice as the Linode Private IP space used for communication between nodes and services within a region use the CIDR 192.168.128.0/17 so using any overlap in this CIDR could cause networking conflicts. For my test deployment detailed in this article I used 10.0.10.0/24 for my vlan_cidr value.
    OPTIONAL — We need to update the container image reference in the DaemonSet to point to your image location if you built the container yourself (however, you can use the default one defined for testing without changing this if you like)

Here’s a summary of where you can make the changes to the DaemonSet YAML definition:

Deploy the DaemonSet

Once you have made the changes you can deploy the DaemonSet and you can even check the logs of one of the pods to see the key points in the script being hit and echoed.

# Apply the YAML
[pnordstrom]$ kubectl apply -f deployment.yaml 
configmap/vlan-join-controller-config created
secret/linode-api created
daemonset.apps/vlan-join-controller created

# Get the list of pods
[pnordstrom]$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
vlan-join-controller-hqc9q   1/1     Running   0          5s
vlan-join-controller-l6fpk   1/1     Running   0          5s
vlan-join-controller-tffvw   1/1     Running   0          5s

# Check the logs from one of the pods
[pnordstrom]$ kubectl logs vlan-join-controller-hqc9q 
All init variables exist, starting script
Adding IP: 10.0.10.99/24 to list of existing VLAN IPs in use
The IP chosen is: 10.0.10.190/24
Node successfully added to VLAN
Node is rebooting

Once the script has updated the pool and the nodes have rebooted the DaemonSet will simply report every minute if it is joined to the VLAN. See example below

# Check for pods after the nodes have rebooted
[pnordstrom]$ kubectl get pods
NAME                         READY   STATUS    RESTARTS        AGE
vlan-join-controller-fgshr   1/1     Running   0               6m56s
vlan-join-controller-vx8jh   1/1     Running   0               4m52s
vlan-join-controller-xrskb   1/1     Running   1 (7m42s ago)   8m22s

# Check the logs for one of the pods to see that it has reported it is connected to the VLAN
[pnordstrom]$ kubectl logs vlan-join-controller-fgshr 
All init variables exist, starting script
compute instance ID not found
Node currently exists in the VLAN lke-private-vlan
Node currently exists in the VLAN lke-private-vlan
Node currently exists in the VLAN lke-private-vlan
Node currently exists in the VLAN lke-private-vlan
Node currently exists in the VLAN lke-private-vlan
Node currently exists in the VLAN lke-private-vlan
Node currently exists in the VLAN lke-private-vlan

Since this deploys as a DaemonSet it will deploy a pod to each new node that is added to the pool as you scale out. While not demonstrated here, you can easily test this out by scaling the pool and verifying that new nodes are registered to the VLAN.
