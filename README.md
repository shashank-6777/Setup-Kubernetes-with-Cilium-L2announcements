# Create a Kubernetes Cluster on Macbook Air M3(16/512) with UTM

Kubernetes itself is unopinionated, which is why there are so many flavors, with different networking options, different storage plugins and more. Each managed Kubernetes provider will also have something unique to its backing cloud platform. These can be provisioned in just a few clicks.

We’ll be creating a three-node cluster with one control plane node and two nodes for our workloads. We recommend at least 3G of memory for each node, with access to 2 CPU cores.


## Preparing the Servers

For this cluster we’ll be using Ubuntu 24.04 Linux, We’ll mostly be following the official Kubernetes installation guide so if you get stuck in any of these steps, feel free to review the original documentation which is much more thorough and explains some edge cases.

The default Ubuntu configuration adds some swap space for us, so our first step is to remove that, given that Kubernetes doesn’t work (well) with swap. All commands as root or sudo going forward:

```bash
$ swapoff -a
$ rm /swap.img
```
For this change to persist across reboots, we’ll also need to remove the swap line in /etc/fstab.

Next, for networking between pods, containers and everything else in our cluster, we’ll need to enable IP forwarding. Uncomment or add the following line in your /etc/sysctl.conf file:

```bash
net.ipv4.ip_forward = 1
```
Apply the changes to the running server with:
```bash
$ sysctl -p
```
We’ll also need a container runtime installed on each Kubernetes node. We’ll be using containerd in our cluster, though other options are also supported. Install containerd on each node and generate a default config:

```bash
$ apt update && apt install -y containerd
$ mkdir /etc/containerd
$ containerd config default > /etc/containerd/config.toml
```
Some features in Kubernetes require cgroup v2 and it’s recommended to use the systemd cgroup driver. To enable that, open the /etc/containerd/config.toml file, find SystemdCgroup and set its value to true. You can do this as a one-liner with sed:
```bash
$ sed -i -e "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
```
Restart containerd after making changes:
```bash
$ systemctl restart containerd
```
Finally, make sure all your nodes have a unique hostname, MAC address and product_uuid. Shouldn’t be a problem if you’ve configured each individual node, but if you’ve used some clone/snapshot feature in your hypervisor, you might need to take additional steps to fix these.

For convenience, I’ve also added the three hostnames to the /etc/hosts file on each node, as well as the laptop which I will be using to work with the cluster. Mine are in a private network, so I’ve used private IPs, but if you’re provisioning your nodes with a cloud provider, you’ll probably be using public ones instead:

```bash
192.168.0.20 kube-master01
192.168.0.21 kube-worker01
192.168.0.22 kube-worker02
```
Now that the prep work is done, let’s move on to installing some Kubernetes tools.

Installing kubeadm and kubelet
A Kubernetes kubelet is a node agent that runs on each server and does a lot of the heavy lifting. The kubeadm utility allows us to create or join a Kubernetes cluster. We’ll need both these utilities on every node in our cluster.

For Ubuntu (and most Debian-based distributions) you can use apt to install these:
```bash
$ apt install -y apt-transport-https ca-certificates curl gpg
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
 
$ apt update && apt install -y kubelet kubeadm
$ apt-mark hold kubelet kubeadm
```

Again, these should be installed on every node in the cluster. It is recommended to hold these packages with apt-mark hold or equivalent, so you can be explicit about your cluster updates.

Next, let’s bootstrap our control plane.

# Bootstrapping the Cluster

Pick one of the nodes to be your control plane node. This is where the control plane containers will reside (Kubernetes API server, etcd and others) and most other workloads will scheduled across the two remaining nodes. On the control plane node bootstrap your cluster using:

```bash
$ kubeadm init
```

You’ll see a lot of things happening here and it might take a few minutes to complete. There are some preflight checks that kubeadm runs and may abort the installation. The error messages are usually descriptive enough to understand what needs to be fixed (if swap is detected for example).

If you’ve made a mistake with kubeadm init don’t worry, you can easily kubeadm reset and try again!

# Accessing the Cluster

Next, install kubectl on the mac which you’ll use to access the cluster externally. We’ll call this the management host going forward. Kubectl is available for Linux, Windows and macOS. It’s the command line utility you’ll be using to interact with your Kubernetes cluster the most.

After installing kubectl copy the contents of the configuration file generated on your control plane node (/etc/kubernetes/admin.conf) into your local configuration file, typically under ~/.kube/config (there are other options too).

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
You should see a list of the three nodes you’ve added to the cluster. If you’re using more than one network interface, use kubectl get nodes -o wide to get more details and confirm the correct IP address is displayed for each node.

You’ll also note that the status of each node is NotReady. That’s because we have not configured networking in our cluster. Let’s do that next.

# Adding a Networking Plugin
Kubernetes is unopinionated about networking too, which is why there is no networking installed by default in a vanilla Kubernetes cluster, and a dozen or so networking plugins/addons.

We’ll be using one called Cilium and these installation instructions. Once you’ve installed the Cilium CLI you can use the utility to install Cilium into your Kubernetes cluster.

The Cilium CLI will also need to know the configuration of your Kubernetes cluster from the KUBECONFIG env var to speak to your cluster. You can set this to the generated /etc/kubernetes/admin.conf when bootstrapping the cluster on the control plane, before running cilium install:

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


#### To install cilium-cli on mac using brew
```bash
brew install cilium-cli
```

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

If everything is going according to plan, the node statuses should now display Ready. The IP addresses should be correct, and you should see a bunch of pods running in the cilium-system namespace as well, with correct IP addresses too:

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

Cilium (and load balancers in general, it seems) have two modes for announcing IPs of services. The more complex one is the BGP mode. In this mode, Cilium would announce routes to the exposed services. This needs an environment where BGP is configured. I decided to skip this approach, as my network knowledge in general isn’t that great. I’ve only got a relatively hazy idea what the BGP protocol even does.

So I settled on the simpler approach, L2 Announcements. In this approach, all Cilium nodes in the cluster take part in a leader election for each of the services which should be exposed and receive a virtual IP. The node which wins the election then answers any ARP requests asking for the MAC address of the node with the service virtual IP. The node then regularly renews a lease in Kubernetes to signal to all other nodes in the cluster that it’s still there. If a lease isn’t renewed in a certain time frame, another node takes over the ARP announcements.

One consequence of this approach is the fact that this is not true load balancing. All traffic for a given service will always arrive at one specific node. From the documentation, this is different when using the BGP approach, as that approach does provide true load balancing. But what the L2 announcements approach does provide is fail over, and this is all that I really care about for my setup, at least for now.

#### Load balancer IP pools

```bash
$ kubectl create -f ciliumLoadBalancerIPPool.yaml
```
It defines a relatively small IP range, as I don’t expect to expose too many services.

#### L2 announcement policies
The second piece of config is the configuration for which services should get an IP and which nodes should do the L2 announcements. This is done via a CiliumL2AnnouncementPolicy manifest.

```bash
$ kubectl create -f CiliumL2AnnouncementPolicy.yaml
```
This restricts the announcements to only happen from my worker nodes, not from the control plane or other available nodes.
In addition, I’m adding a serviceSelector here, so that only certain services get an IP and are announced.


## Example

With all of that config done, let’s have a look at an example. I used the existing grafana/prometheus service's and changed from ClusterIP to Loadblancer.

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
The important part here is the EXTERNAL_IP. Next, check whether there is a Kubernetes lease created by anyone , signaling that the node is announcing the service:

```bash
shashank@Mac Grafana-SVC % kubectl get -n cilium-system leases.coordination.k8s.io
NAME                                             HOLDER                AGE
cilium-l2announce-cilium-monitoring-grafana      worker02              23h
cilium-l2announce-cilium-monitoring-prometheus   worker02              22h
cilium-operator-resource-lock                    worker01-kmpq92rsqt   26h
shashank@Mac Grafana-SVC %
```



