# Create a Kubernetes Cluster on Macbook Air M3(16/512) with UTM

Since Kubernetes is an unbiased platform, there are numerous variations available, featuring varying networking configurations, storage plugins, and additional features. Every managed Kubernetes provider will also have a different feature for the cloud platform that supports it. These only require a few clicks to furnish.

With one node serving as the control plane and two nodes serving our workloads, we will be building a three-node cluster. For every node, we advise having at least 3G of memory and access to 2 CPU cores.

## Setting Up the Servers

We'll be utilizing Ubuntu 24.04 Linux on this cluster. Since Kubernetes doesn't operate (well) with swap, our first step is to remove the swap space that is added by default in the Ubuntu configuration. From now on, run all commands as root or sudo:

```bash
$ swapoff -a
$ rm /swap.img
```
In order for this modification to endure over reboots, we must further eliminate the swap line from /etc/fstab.

IP forwarding must then be enabled in order to facilitate networking between pods, containers, and every other component of our cluster. In your /etc/sysctl.conf file, remove the following statement or add the following:

```bash
net.ipv4.ip_forward = 1
```
Apply the changes to the running server with:
```bash
$ sysctl -p
```
On every Kubernetes node, a container runtime installation is also required. In our cluster, we will be utilizing containerd, however there are other possibilities as well. Install containerd and create a default configuration on every node:

```bash
$ apt update && apt install -y containerd
$ mkdir /etc/containerd
$ containerd config default > /etc/containerd/config.toml
```
It is advised to utilize the systemd cgroup driver and certain Kubernetes capabilities require cgroup v2. Open the /etc/containerd/config.toml file, locate SystemdCgroup, and change its value to true to enable that. With sed, you may accomplish this as a one-liner:
```bash
$ sed -i -e "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
```
Restart containerd after making changes:
```bash
$ systemctl restart containerd
```
Lastly, confirm that the hostname, MAC address, and product_uuid of each node are distinct. If you have configured each individual node, there shouldn't be any issues; however, if your hypervisor has a clone/snapshot feature, you may need to take further measures to resolve them.

For convenience, I’ve also added the three hostnames to the /etc/hosts file on each node, as well as the Macbook which I will be using to work with the cluster. Mine are in a private network, so I’ve used private IPs, although you'll most likely be utilizing public ones if you're deploying your nodes using a cloud provider:

```bash
192.168.0.20 kube-master01
192.168.0.21 kube-worker01
192.168.0.22 kube-worker02
```
After completing the necessary preparations, let's install the Kubernetes tools.

# Installing kubeadm and kubelet
A Kubernetes kubelet is a node agent that runs on each server and does a lot of the heavy lifting. A Kubernetes cluster can be started or joined using the kubeadm tool. On each node in our cluster, we will require both of these functions.

For Ubuntu (and most Debian-based distributions) you can use apt to install these:
```bash
$ apt install -y apt-transport-https ca-certificates curl gpg
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
 
$ apt update && apt install -y kubelet kubeadm
$ apt-mark hold kubelet kubeadm
```

Once more, each cluster node needs to have these installed. Holding certain packages with apt-mark hold or its equivalent is advised so that you can be specific about cluster updates.

Let's bootstrap our control plane next.

# Establishing the Cluster from Scratch

Select a node to serve as your control plane node. The majority of other workloads will be distributed among the two remaining nodes, and this is where the control plane containers (Kubernetes API server, etcd, and others) will be housed. You can bootstrap your cluster on the control plane node by using:

```bash
$ kubeadm init
```

There are a lot of things going on here, and it may take some time to finish. Kubeadm performs some preflight checks and has the ability to terminate the installation. When swap is discovered, for example, the error messages are typically sufficiently detailed to determine what has to be addressed.

If you’ve made a mistake with kubeadm init don’t worry, you can easily kubeadm reset and try again!

# Getting inside the Cluster

Next, set up kubectl on the Mac, which will be used to connect to the cluster from outside. From now on, we'll refer to this as the management host. Windows, macOS, and Linux can all run Kubectl. It's the main command line tool you'll use to communicate with your Kubernetes cluster.

The configuration file generated on your control plane node (/etc/kubernetes/admin.conf) should be copied into your local configuration file, usually located under ~/.kube/config (but there are other alternatives as well), after installing kubectl.

#### To install kubectl on mac using brew
```bash
brew install kubectl

```
Check your cluster nodes with:
```bash
shashank@Mac ~ % kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   26h   v1.30.5
worker01   Ready    worker          26h   v1.30.5
worker02   Ready    worker          26h   v1.30.5
shashank@Mac ~ % 
```
The three nodes you added to the cluster ought should be listed for you to observe. If you’re using more than one network interface, use kubectl get nodes -o wide to get further details and validate the right IP address is displayed for each node.

You’ll also note that the status of each node is NotReady. That’s because we have not configured networking in our cluster. Let’s do that next.

# Adding a Networking Plugin
Since Kubernetes is agnostic towards networking as well, a vanilla Kubernetes cluster comes with a dozen or so networking plugins/addons installed instead of networking installed by default.

One that we'll use is named Cilium. You can use the Cilium CLI tool to install Cilium within your Kubernetes cluster after installing it on your system.

#### To install cilium-cli on mac using brew
```bash
brew install cilium-cli
```

To communicate with your Kubernetes cluster, the Cilium CLI will also require knowledge of its configuration, which can be found in the KUBECONFIG environment variable. When bootstrapping the cluster on the control plane, you can set this to the produced /etc/kubernetes/admin.conf before executing cilium install:

#### To install helm on mac using brew
```bash
brew install helm
```
```bash
helm install cilium cilium/cilium --version 1.16.1 \
   --namespace cilium-system \
   --set l2announcements.enabled=true \
   --set k8sClientRateLimit.qps=100 \
   --set k8sClientRateLimit.burst=200 \
   --set kubeProxyReplacement=true \
   --set k8sServiceHost=${API_SERVER_IP} \
   --set k8sServicePort=${API_SERVER_PORT} \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true \
   --set prometheus.enabled=true \
   --set operator.prometheus.enabled=true \
   --set hubble.enabled=true \
   --set hubble.metrics.enableOpenMetrics=true \
   --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```
It’ll take a few minutes to setup the networking plugin. Once successful, you should see some new pods running, and an overall OK status for Cilium:

```bash
shashank@Mac ~ % cilium status -n cilium-system
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

Deployment             hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium-envoy       Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Deployment             hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium-envoy       Running: 3
                       cilium             Running: 3
                       hubble-ui          Running: 1
                       hubble-relay       Running: 1
                       cilium-operator    Running: 2
Cluster Pods:          13/13 managed by Cilium
Helm chart version:    
Image versions         cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 3
                       cilium             quay.io/cilium/cilium:v1.16.1@sha256:0b4a3ab41a4760d86b7fc945b8783747ba27f29dac30dd434d94f2c9e3679f39: 3
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.1@sha256:e2e9313eb7caf64b0061d9da0efbdad59c6c461f6ca1752768942bfeda0796c6: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.1@sha256:0e0eed917653441fded4e7cdb096b7be6a3bddded5a2dd10812a27b1fc6ed95b: 1
                       hubble-relay       quay.io/cilium/hubble-relay:v1.16.1@sha256:2e1b4c739a676ae187d4c2bfc45c3e865bda2567cc0320a90cb666657fcfcc35: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.1@sha256:3bc7e7a43bc4a4d8989cb7936c5d96675dd2d02c306adf925ce0a7c35aa27dc4: 2
shashank@Mac ~ % 
```
Let’s take a look at our nodes from outside the cluster via kubectl:

```bash
shashank@Mac ~ % kubectl get nodes -o wide     
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
master01   Ready    control-plane   26h   v1.30.5   192.168.0.20   <none>        Ubuntu 24.04.1 LTS   6.8.0-45-generic   containerd://1.7.12
worker01   Ready    worker          26h   v1.30.5   192.168.0.21   <none>        Ubuntu 24.04.1 LTS   6.8.0-45-generic   containerd://1.7.12
worker02   Ready    worker          26h   v1.30.5   192.168.0.22   <none>        Ubuntu 24.04.1 LTS   6.8.0-45-generic   containerd://1.7.12
shashank@Mac ~ % 
```

The node statuses ought to now show Ready if everything is proceeding as planned. In addition to seeing several pods operating in the cilium-system namespace with proper IP addresses, the IP addresses should be correct:

```bash
shashank@Mac ~ % kubectl get pods -n cilium-system
NAME                               READY   STATUS    RESTARTS      AGE
cilium-5m5xc                       1/1     Running   3 (19h ago)   25h
cilium-envoy-6lc88                 1/1     Running   3 (19h ago)   25h
cilium-envoy-cl5xb                 1/1     Running   2 (19h ago)   25h
cilium-envoy-mcpvf                 1/1     Running   2 (19h ago)   25h
cilium-operator-64767f6566-2cnm9   1/1     Running   2 (19h ago)   25h
cilium-operator-64767f6566-fszkg   1/1     Running   6 (19h ago)   25h
cilium-s2jk8                       1/1     Running   2 (19h ago)   25h
cilium-x9rcx                       1/1     Running   2 (19h ago)   25h
hubble-relay-644b68d988-29vhw      1/1     Running   3 (19h ago)   25h
hubble-ui-59bb4cb67b-stc89         2/2     Running   6 (19h ago)   25h
shashank@Mac ~ % 
```

## Cilium L2 announcements setup

There are two ways that Cilium (and load balancers in general, it appears) announces service IPs. The BGP mode is the more intricate one. In this mode, Cilium would declare routes to the exposed services. An environment with configured BGP is required for this. Since I don't know a lot about networks in general, I choose not to use this strategy. I just have a vague understanding of what the BGP protocol is used for.

I therefore chose the easier strategy of L2 Announcements. With this method, every Cilium node in the cluster participates in an election to choose a leader for every service that needs to be made public and given a virtual IP address. After winning the election, the node with the service virtual IP responds to any ARP queries requesting the node's MAC address. The node then periodically renews a lease in Kubernetes to indicate to all other nodes in the cluster that it’s still there. Another node assumes control of the ARP announcements if a lease isn't extended within a predetermined window of time.

This strategy has several consequences, one of which is that it is not genuine load balancing. One particular node will always receive all traffic for a given service. According to the documentation, this isn't the case when employing the BGP technique, which offers real load balancing. All I really care about for my system, at least for the time being, is fail over, which is something that the L2 announcements strategy does give.

#### Load balancer IP pools

```bash
$ kubectl create -f ciliumLoadBalancerIPPool.yaml
```
I don't anticipate exposing too many services, thus it defines a small IP range.

#### L2 announcement policies
The configuration for which nodes should make the L2 announcements and which services should receive IP addresses makes up the second piece of setup. A CiliumL2AnnouncementPolicy manifest is used to do this.

```bash
$ kubectl create -f CiliumL2AnnouncementPolicy.yaml
```
This limits the notifications to come from my worker nodes alone—not from the control plane or any other nodes that might be available.
To ensure that only specific services receive an IP address and are publicized, I'm also implementing a serviceSelector here.


## Example

Now that everything has been configured, let's look at an example. I switched from ClusterIP to Loadblancer using the grafana/prometheus services that were already in place.

```bash
$ kubectl apply -f grafana-svc.yaml
$ kubectl apply -f prometheus-svc.yaml
```

```bash
shashank@Mac Grafana-SVC % kubectl get svc -n cilium-monitoring
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
grafana      LoadBalancer   10.109.114.38   192.168.0.200   3000:30980/TCP   24h
prometheus   LoadBalancer   10.110.208.59   192.168.0.201   9090:32024/TCP   24h
shashank@Mac Grafana-SVC % 
shashank@Mac Grafana-SVC %
```
Here, the EXTERNAL_IP is the crucial component. Next, see if someone has created a Kubernetes lease, which indicates that the node is announcing the service:

```bash
shashank@Mac Grafana-SVC % kubectl get -n cilium-system leases.coordination.k8s.io
NAME                                             HOLDER                AGE
cilium-l2announce-cilium-monitoring-grafana      worker02              23h
cilium-l2announce-cilium-monitoring-prometheus   worker02              22h
cilium-operator-resource-lock                    worker01-kmpq92rsqt   26h
shashank@Mac Grafana-SVC %
```






## Documentation

[Kubernetes setup](https://kubernetes.io/docs/setup/)

[Cilum CNI](https://docs.cilium.io/en/stable/)




## Authors

[@shashank-linkedin](https://linkedin.com/in/shashank-sharma-137002124)