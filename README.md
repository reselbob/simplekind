# simplekind
A project that demonstrates how used KinD and the MetalLB LoadBalancer to create a load balanced Kubernetes cluster.

**CREDIT:** This project builds off of the work done by 
[Venkat Nagappan](vhttps://twitter.com/VenkatNagappan) on his YouTube video found [here](https://youtu.be/zNbqxPRTjFg).

## Setting Up Kind under KataCoda

**Step 1:** Go to the Ubuntu Playground on Katacoda

`https://katacoda.com/courses/ubuntu/playground`

**Step 2:** Install `kubectl`

```
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

```

**Step 3:** Check to make sure kubectl is running

`kubectl version --client`

You'll get output similar to the following:

```
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
```

**Step 4:** Install [KinD](https://kind.sigs.k8s.io/)

```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64

chmod +x ./kind

mv ./kind /usr/local/bin/kind

```

**Step 5:** Create the configuration file using `vi` that tells `KinD` how to create the cluster

`vi kind-config.yaml`

Add the following to the file, `kind-config.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker

```

**Step 6:** 

Create the cluster in `KinD`

`kind create cluster --config kind-config.yaml`

You'll get output similar to the following:

```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋

```

**Step 7:** Install the tool, `sipcalc`. You'll need it to determine the range of IP addresses that the [MetalLB](https://metallb.universe.tf/) load balancer will use.

`sudo apt install sipcalc -y`

**Step 8:** Install [MetalLB](https://metallb.universe.tf/)

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/namespace.yaml

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/metallb.yaml

kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

```

**Step 9:** Find out the IP addresses being used by the nodes in the K8S cluster:

`kubectl get nodes -o wide`

You'll get output similar to the following:

```
NAME                 STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                                     KERNEL-VERSION      CONTAINER-RUNTIME
kind-control-plane   Ready    master   5m23s   v1.19.1   172.19.0.4    <none>        Ubuntu Groovy Gorilla (development branch)   4.4.0-185-generic   containerd://1.4.0
kind-worker          Ready    <none>   4m42s   v1.19.1   172.19.0.3    <none>        Ubuntu Groovy Gorilla (development branch)   4.4.0-185-generic   containerd://1.4.0
kind-worker2         Ready    <none>   4m41s   v1.19.1   172.19.0.2    <none>        Ubuntu Groovy Gorilla (development branch)   4.4.0-185-generic   containerd://1.4.0

```

**Step 10:** Find the [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) of the bridge into tthe cluster

`ip a s`

You'll get output similar to the following

```
.
.
.
4: br-956cdadd3d33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:8b:33:c3:fb brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-956cdadd3d33
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::42:8bff:fe33:c3fb/64 scope link
       valid_lft forever preferred_lft forever
    inet6 fe80::1/64 scope link
       valid_lft forever preferred_lft forever
.
.
.
```

**Step 11:** Use the `sipcalc` tool find the range of IP addresses that MetalLB can use.

`sipcalc 172.19.0.1/16`


**WHERE**

`172.19.0.1/16` is the CIDR discovered using `ip a s`

You'll get put similar to the following:

```
-[ipv4 : 172.19.0.1/16] - 0

[CIDR]
Host address            - 172.19.0.1
Host address (decimal)  - 2886926337
Host address (hex)      - AC130001
Network address         - 172.19.0.0
Network mask            - 255.255.0.0
Network mask (bits)     - 16
Network mask (hex)      - FFFF0000
Broadcast address       - 172.19.255.255
Cisco wildcard          - 0.0.255.255
Addresses in network    - 65536
Network range           - 172.19.0.0 - 172.19.255.255
Usable range            - 172.19.0.1 - 172.19.255.254

```

**Step 12:** Create the ConfigMap for MetalLB using `vi`

`vi metallb-config.yaml`

Add the following content to `metallb-config.yaml`

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.19.255.1-172.19.255.250

```

**WHERE**

`172.19.255.1-172.19.255.250` are the IP addreses determined from the steps above.

**Step 13:** Apply the ConfigMap to the K8S cluster

`kubectl apply -f metallb-config.yaml`

**Step 14:** Create a test K8S deployment and bind it to a service

```
kubectl create deploy nginx --image nginx

kubectl expose deploy nginx --port 80 --type LoadBalancer

```

**Step 15:** Confirm that the `nginx` service is bound to an external IP address

`kubectl get services`

You'll get out put similar to the following:

```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP        11m
nginx        LoadBalancer   10.96.148.110   172.19.255.1   80:32326/TCP   9s
```

Notice that the nginx service publishes an `EXTERNAL-IP`.

**Step 16:** Run the `curl` against the `EXTERNAL-IP`, for example:

`curl 172.19.255.1`

You'll get output similar to the following:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
