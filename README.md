# rancher-desktop-calico-policies

I avoided using the built-in Kubernetes component of Rancher Desktop. <br/>
I unchecked this option and switched the container runtime from ```containerd``` to ```dockerd```:

<img width="337" alt="Screenshot 2022-05-06 at 10 55 19" src="https://user-images.githubusercontent.com/82048393/167110045-700d5474-6d46-4ff2-856b-de7d3584d3cd.png">

Rancher Desktop will setup a single node VM inside your laptop/desktop <br/>
This entire process will take less than a minute to complete:

<img width="941" alt="Screenshot 2022-05-06 at 10 55 41" src="https://user-images.githubusercontent.com/82048393/167110190-8eb8968e-f93b-4278-9242-0d3332cd649a.png">

<img width="941" alt="Screenshot 2022-05-06 at 10 56 00" src="https://user-images.githubusercontent.com/82048393/167110204-ff08007b-cf31-4343-8467-8e4c936f1e46.png">

## Install KinD

KinD offers a lot of customization like using another CNI component like Calico for CNI and network policies instead of relying on the built-in one <br/>
If you want to use KinD on Rancher Desktop for Kubernetes, make sure you are always using the latest release. It’s ```v0.12.0``` at the time of writing

```
cd desktop
```

```
mkdir rancher-desktop
```

```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.12.0/kind-darwin-amd64
```


```
chmod +x ./kind
```

<img width="941" alt="Screenshot 2022-05-06 at 11 08 52" src="https://user-images.githubusercontent.com/82048393/167112056-0c9c86d6-16f1-4491-91e6-a74fb360ff93.png">



## Disabling Kindnet

To use Calico as the CNI plugin in Kind clusters, we need to do the following: <br/>
<br/>

Create a ```kind-calico.yaml``` file that contains the following:
```
vi kind-calico.yaml
```


Disable the installation of kindnet
```
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
networking:
  disableDefaultCNI: true # disable kindnet
  podSubnet: 192.168.0.0/16 # set to Calico's default subnet
```

Create your Kind cluster, passing the configuration file using the --config flag:

```
kind create cluster --config kind-calico.yaml
```

## Verify Kind Cluster

Once the cluster is up, list the pods in the ```kube-system``` namespace to verify that ```kindnet``` is not running:

```
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
kubectl get pods -n kube-system
```

```kindnet``` should be missing from the list of pods:

```Note:``` The coredns pods are in the ```pending``` state. This is expected. <br/>
They will remain in the pending state until a CNI plugin is installed.

## Install Calico CNI
