# Kubernetes

## Contents

1. [Kubernetes](#what-is-kubernetes)
2. [Why Kubernetes](#why-kubernetes)
2. [Simple concepts in Kubernetes](#concepts-related-to-kubernetes)
3. [Deployement of Kubernetes Cluster](#deploy-a-kubernetes-cluster)
    - [Accessing the Kubernetes Network](#access-kubernetes-network)
4. [Deploy a Pod / App into the Cluster](#deploy-an-app-into-the-cluster)
    - [Accessing the Deployed App](#time-to-access-the-deployed-app)
5. [Kubernetes Debug](#kubernetes-debug)
6. [Usage instructions](#usage-instructions)

## What is Kubernetes? 

Kubernetes (also known as "k8s") is an open-source platform used for managing containerized workloads and services. 


> It was originally designed by Google and is now maintained by the Cloud Native Computing Foundation (CNCF). Kubernetes provides a flexible and scalable architecture for automating the deployment, scaling, and management of containerized applications across distributed environments. 


With Kubernetes, developers and IT teams can easily manage and orchestrate containerized applications, while ensuring high availability, fault tolerance, and scalability. It also provides a rich set of features such as service discovery, load balancing, and automatic rollouts and rollbacks, making it a popular choice for modern cloud-native applications.

*To put it in simpler terms;*

Kubernetes is a tool that helps manage and run applications made up of many smaller parts, called containers. It makes it easier for developers and IT teams to automate the deployment, scaling, and management of these applications across different computers and servers. 

With Kubernetes, you can ensure that your application is always available, even if some parts of it fail. It also has many useful features, like automatic load balancing and updating of your application. Overall, Kubernetes makes it easier to build and run complex applications.

---

## Why Kubernetes?

We use Kubernetes to simplify the deployment and management of complex applications made up of many smaller parts. It provides a platform to automate tasks such as scaling, monitoring, and updating our applications, making it easier to manage them as they grow and evolve over time.

**Some Key Benefits:**

1. Automates the deployment, scaling, and management of containerized applications
Provides high availability and resilience to application failures
2. Offers self-healing capabilities to automatically recover from application and infrastructure failures
3. Supports declarative configuration to define desired state and ensure consistency
4. Enables horizontal scaling to handle changes in demand and traffic
5. Offers a wide range of plugins and extensions for storage, networking, and security
6. Supports multiple cloud providers and on-premises data centers
7. Provides a rich ecosystem of tools and resources for development and operations

---

## Concepts related to Kubernetes

> Before beginning the real game, let's go through some concepts related to Kubernetes.

**1. Nodes:** These are the physical or virtual machines where containers are deployed and run.

**2. Pods:** Pods are the smallest unit of deployment in Kubernetes, and they are used to host one or more containers.

**3. Services:** Services are used to provide a consistent way to access a set of pods, which can be load balanced across multiple nodes.

**4. Deployments:** Deployments are used to manage the rollout and scaling of application updates.

**5. Replication Controllers:** Replication Controllers ensure that a specified number of replicas of a pod are running at any given time.

**6. ConfigMaps:** ConfigMaps are used to store configuration data that can be used by pods or deployments.

**7. Secrets:** Secrets are used to store sensitive information, such as passwords or API keys, that can be accessed by pods or deployments.

**8. Volumes:** Volumes are used to provide persistent storage to pods.

**9. Namespaces:** Namespaces are used to logically separate different environments or projects within a Kubernetes cluster.

**10. Labels and Selectors:** Labels are used to attach metadata to objects, such as pods or services, while selectors are used to filter and find objects based on their labels.

---

## Deploy a Kubernetes Cluster

> To deploy a Kubernetes cluster, you will require some basic knowledge of Linux and its related tools. 

Setting up a real cluster is a challenging task that demands significant effort. However, for the purpose, we will deploy a test cluster on a single Ubuntu 64-bit server, which could either be a cloud VM or a Vagrant box. 

To do this, you can use the following command:

`wget -qO- http://git.io/veKlu | sudo sh` 

This command will use kube-init to deploy the cluster, which is based on the latest version of Kubernetes and can be updated with new releases.

This is the code for `Kube-init.sh`

```
UBUNTU_DISTRO=`uname -a | grep Ubuntu`
ARCHI=`uname -m`

if [ ! "$UBUNTU_DISTRO" ]; then
  echo "=> kube-init requires a Ubuntu distro (64 bit)"
  exit 1
fi

if [ $ARCHI != "x86_64" ]; then
  echo "=> kube-init requires a 64 bit Ubuntu distro"
  exit 1
fi

# Install Docker 
wget -qO- https://get.docker.com/ | sh

## ETCD
docker run \
    -d \
    --net=host \
    quay.io/coreos/etcd:v2.0.9 \
        --addr=127.0.0.1:4001 \
        --bind-addr=0.0.0.0:4001 \
        --data-dir=/var/etcd/data

## HyperKube apiserver
# here we set the address to `0.0.0.0`
# that's because we need to bind the api server with all IPs.
# So, in kube-dns `kube2sky` can connect to api server via
# docker's IP bridge IP 
# otherwise, we need to go through a lot of security related stuff
docker run \
    --net=host \
    -d \
    -v /var/run/docker.sock:/var/run/docker.sock\
    meteorhacks/hyperkube \
      /hyperkube apiserver \
      --portal_net=10.0.0.1/24 \
      --address=0.0.0.0 \
      --etcd_servers=http://127.0.0.1:4001 \
      --cluster_name=kubernetes \
      --v=2

## HyperKube controller-manager
docker run \
    --net=host \
    -d \
    -v /var/run/docker.sock:/var/run/docker.sock\
    meteorhacks/hyperkube \
      /hyperkube controller-manager \
      --master=127.0.0.1:8080 \
      --machines=127.0.0.1 \
      --sync_nodes=true \
      --v=2

## HyperKube scheduler
docker run \
    --net=host \
    -d \
    -v /var/run/docker.sock:/var/run/docker.sock\
    meteorhacks/hyperkube \
      /hyperkube scheduler \
      --master=127.0.0.1:8080 \
      --v=2

## HyperKube kubelet
docker run \
    --net=host \
    -d \
    -v /var/run/docker.sock:/var/run/docker.sock\
    meteorhacks/hyperkube \
      /hyperkube kubelet \
        --api_servers=http://127.0.0.1:8080 \
        --v=2 \
        --address=0.0.0.0 \
        --hostname_override=127.0.0.1 \
        --cluster_dns=10.0.0.10 \
        --cluster_domain="kubernetes.local" \
        --config=/etc/kubernetes/manifests

## Proxy which changes IP Tables Rules
docker run \
    -d \
    --net=host \
    --privileged \
    meteorhacks/hyperkube \
      /hyperkube proxy \
        --master=http://127.0.0.1:8080 \
        --v=2

## kubectl
docker run --rm -v /usr/local/bin:/_bin meteorhacks/hyperkube /bin/bash -c "cp /kubectl /_bin"
chmod +x /usr/local/bin/kubectl

## Add DNS Support
cat <<EOF > /tmp/kube-dns-rc.yaml
apiVersion: v1beta3
kind: ReplicationController
metadata:
  labels:
    k8s-app: kube-dns
  name: kube-dns
  namespace: default
spec:
  replicas: 1
  selector:
    k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      dnsPolicy: "Default"
      containers:
      - args: [
          "-listen-client-urls=http://0.0.0.0:2379,http://0.0.0.0:4001",
          "-initial-cluster-token=skydns-etcd",
          "-advertise-client-urls=http://127.0.0.1:4001"
        ]
        image: quay.io/coreos/etcd:v2.0.3
        name: etcd
      - args: [
          "-domain=kubernetes.local.",
          # we are using docker's default bridge IP here
          # that's the only way we can connect inside from the pod
          "--kube_master_url=http://172.17.42.1:8080"
        ]
        image: gcr.io/google_containers/kube2sky:1.9
        name: kube2sky
      - args: [
          "-machines=http://localhost:4001",
          "-addr=0.0.0.0:53",
          "-domain=kubernetes.local.",
        ]
        image: gcr.io/google_containers/skydns:2015-03-11-001
        name: skydns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
EOF

cat <<EOF > /tmp/kube-dns-service.yaml
apiVersion: v1beta3
kind: Service
metadata:
  labels:
    k8s-app: kube-dns
  name: kube-dns
  namespace: default
spec:
  portalIP: 10.0.0.10
  ports:
  - port: 53
    protocol: UDP
    targetPort: 53
  selector:
    k8s-app: kube-dns
EOF

waitFor() {
    cmd=$1
    while [ 1 ]; do
        ok=$(eval $cmd)
        if [ "$ok" ]; then
            break
        fi
        sleep 1
    done
}

echo
echo "=> Waiting for ETCD. (takes upto 2-5 minute)"
waitFor "wget -qO- http://127.0.0.1:4001/version | grep etcd"
echo "=>  ETCD is now online."

echo
echo "=> Waiting for Kubernates API. (takes upto 2-5 minute)"
waitFor "wget -qO- http://127.0.0.1:8080/version | grep major"
echo "=>  Kubernates API is now online."

echo
echo "=> Setting up DNS confgurations"

kubectl create -f /tmp/kube-dns-rc.yaml
kubectl create -f /tmp/kube-dns-service.yaml

rm /tmp/kube-dns-rc.yaml /tmp/kube-dns-service.yaml

echo
echo "=> Waiting for DNS confgurations (takes upto 2-5 minute)"
waitFor 'kubectl get pod -l k8s-app=kube-dns | grep Running | grep "3/3"'
echo "=>  DNS confguration completed."

## Done
echo
echo "------------------------------------------------------------------"
echo "=> Installed a Standalone Kubernates Cluster!"
echo "->  type "kubectl" to start playing with the cluster"
echo "->  to learn about Kubernetes, visit here: http://goo.gl/jmxn2W"
echo "------------------------------------------------------------------"
echo

```

---

### Access Kubernetes Network

Kubernetes generates an internal network within the cluster where each pod and service has a unique IP address. This feature enables us to access these IPs from our local machine. 

To accomplish this, we need to establish a SOCKS proxy connection with our server. 

You can create this connection by executing the following command in a separate terminal:

`ssh -D 8082 your-user@your-server-ip`

This command will help you login to this server and create a localhost on `8082`

As soon as that is done, we must set up our browser to use the newly created SOCKS proxy. On a Mac, I'll teach you how to accomplish this for both Chrome and Firefox. The instructions for Linux and Windows are nearly identical.

For [Mozilla Firefox](https://www.youtube.com/watch?v=QPSYy0vZjw4) - (Recommended. This will just add the proxy to Firefox.)


For [Google Chrome](https://www.youtube.com/watch?v=ASugh1lSmps) - (This will add the proxy to your whole machine.)

---

## Deploy an App into the Cluster

Now we will deploy a pod / app into the cluster. The app is a [Telescope app] (http://www.telescopeapp.org/) that is an open-source app. 

We will first create a file called telescope-pod.json inside the server. 
It will look like this;

```
{
  "metadata": {
    "name": "telescope-pod"
  },
  "kind": "Pod",
  "apiVersion": "v1beta3",
  "spec": {
    "containers": [
      {
        "name": "mongo",
        "image": "mongo",
        "command": ["mongod", "--smallfiles", "--nojournal"],
        "ports": [{
          "containerPort": 27017
        }]
      },
      {
        "name": "telescope",
        "image": "meteorhacks/telescope",
        "ports": [{
          "containerPort": 80
        }],
        "env": [
          {"name": "ROOT_URL", "value": "http://mydomain.com"},
          {"name": "MONGO_URL", "value": "mongodb://localhost:27017/app"}
        ]
      }
    ]
  }
}
```

There are two containers in this file: MongoDB and Telescope app. The MongoDB URL was passed to the telescope app through an environment variable as `mongodb:/localhost:27017/app`.

We can now access the MongoDB container port from within the telescope app. Ordinarily, we would not be able to do this, however it is allowed within a pod since a single pod shares resources with other containers. The network is one such resource.

Run the following command to launch the pod:
`kubectl create -f telescope-pod.json`

Now if you want, you can also check the status of your pod / app running;
`kubectl get pod`

At first, it'll display the status of our pod as 'Pending'. This is due to Kubernetes downloading telescope and Mongo pictures. After a few moments, you should notice the state 'Running'.

---

### Time to access the deployed app

Let's run `kubectl get pod`

![](https://camo.githubusercontent.com/928e21dbb63b1788a70880db4c9ae2333f35b99b3558e763b775c1bf1bfb4d59/68747470733a2f2f636c6475702e636f6d2f6e7a74537968377437452e706e67)

You can check the IP address that has been assigned to your pod here. That is your pod's IP address. Copy and paste it into the SOCKS proxy-configured browser. You may now use our telescope app.

![](https://camo.githubusercontent.com/b2eeaf385a4369f08dae9769bdb329d9f164720eb9bdb1d53f87ffde16215c47/68747470733a2f2f636c6475702e636f6d2f70476e336a6b346531302e706e67)

---


# Kubernetes Debug

> Debugging Kubernetes involves a variety of steps and techniques, including examining logs, checking the status of running pods and nodes, and troubleshooting networking issues. 

For connecting to and debugging a currently running `target` container in a currently existent pod in a Kubernetes cluster, use `kubectl-debug`. The target container may have a shell and busybox utils, providing some debug capabilities, or it may be very basic, lacking even a shell, making real-time troubleshooting/debugging impossible. `kubectl-debug` is meant to address this issue.

**This is a high-level overview of the process:**

**- Check the logs:** Kubernetes logs can provide valuable insight into what went wrong with a pod or node. You can view the logs for a pod by running `kubectl logs <pod-name>` or for a node by running `journalctl -u kubelet`.

**- Check the pod status:** You can check the status of pods in Kubernetes by running `kubectl get pods`. This will show you the status of all pods running in the cluster.

**- Check the node status:** You can check the status of nodes in Kubernetes by running `kubectl get nodes`. This will show you the status of all nodes in the cluster.

**- Troubleshoot networking issues:** Kubernetes networking can be complex, so it's important to be familiar with the various networking components, including services, endpoints, and network policies. You can troubleshoot networking issues by using `kubectl describe` commands and examining the logs.

**- Use debugging tools:** Kubernetes provides several debugging tools, including `kubectl exec` for running commands inside a container, `kubectl port-forward` for forwarding a port from a pod to your local machine, and `kubectl debug` for attaching a debugger to a running container.

** How does it work?**

1. The following is how the user invokes `kubectl-debug`: `kubectl-debug --namespace NAMESPACE POD_NAME -c TARGET_CONTAINER_NAME`

2. `kubectl-debug` uses the same interface as kubectl to connect with the cluster and instructs kubernetes to start a new 'debug-agent' container on the same node as the 'target' container.

3. The debug-agent pod's debug-agent process connects directly to containerd (or dockerd if applicable) on the host running the 'target' container and requests the launch of a new 'debug' container with the same pid, network, user, and ipc namespaces as the target container.

4. In summary, 'kubectl-debug' launches the 'debug-agent' container, and 'debug-agent' launches the 'debug' pod/container.

5. The 'debug-agent' pod routes the 'debug' container's terminal output to the 'kubectl-debug' executable, allowing you to communicate directly with the shell running in the 'debug' container. You may now utilise the troubleshooting tools accessible in the debug container (BASH, cURL, tcpdump, and so on) without requiring these utilities to be present in the target container image.

---

### Usage instructions

``` 
# kubectl 1.12.0 or higher

# print the help
kubectl-debug -h

# start the debug container in the same namespace, and cgroup etc as container 'TARGET_CONTAINER_NAME' in
#  pod 'POD_NAME' in namespace 'NAMESPACE'
kubectl-debug --namespace NAMESPACE POD_NAME -c TARGET_CONTAINER_NAME

# in case of your pod stuck in `CrashLoopBackoff` state and cannot be connected to,
# you can fork a new pod and diagnose the problem in the forked pod
kubectl-debug --namespace NAMESPACE POD_NAME -c CONTAINER_NAME --fork

# In 'fork' mode, if you want the copied pod to retain the labels of the original pod, you can use 
# the --fork-pod-retain-labels parameter (comma separated, no spaces). If not set (default), this parameter 
# is empty and so any labels of the original pod are not retained, and the labels of the copied pods are empty.
# Example of fork mode:
kubectl-debug --namespace NAMESPACE POD_NAME -c CONTAINER_NAME --fork --fork-pod-retain-labels=<labelKeyA>,<labelKeyB>,<labelKeyC>

# in order to interact with the debug-agent pod on a node which doesn't have a public IP or direct access 
# (firewall and other reasons) to access, port-forward mode is enabled by default. If you don't want 
# port-forward mode, you can use --port-forward false to turn off it. I don't know why you'd want to do 
# this, but you can if you want.
kubectl-debug --port-forward=false --namespace NAMESPACE POD_NAME -c CONTAINER_NAME

# you can choose a different debug container image. By default, nicolaka/netshoot:latest will be 
# used but you can specify anything you like
kubectl-debug --namespace NAMESPACE POD_NAME -c CONTAINER_NAME --image nicolaka/netshoot:latest 

# you can set the debug-agent pod's resource limits/requests, for example:
# default is not set
kubectl-debug --namespace NAMESPACE POD_NAME -c CONTAINER_NAME --agent-pod-cpu-requests=250m --agent-pod-cpu-limits=500m --agent-pod-memory-requests=200Mi --agent-pod-memory-limits=500Mi

# use primary docker registry, set registry kubernetes secret to pull image
# the default registry-secret-name is kubectl-debug-registry-secret, the default namespace is default
# please set the secret data source as {Username: <username>, Password: <password>}
kubectl-debug --namespace NAMESPACE POD_NAME --image nicolaka/netshoot:latest --registry-secret-name <k8s_secret_name> --registry-secret-namespace <namespace>

# in addition to passing cli arguments, you can use a config file if you would like to 
# non-default values for various things.
kubectl-debug --configfile /PATH/FILENAME --namespace NAMESPACE POD_NAME -c TARGET_CONTAINER_NAME
 ```

