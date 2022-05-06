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
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true # disable kindnet
  podSubnet: 192.168.0.0/16 # set to Calico's default subnet
```

Create your Kind cluster, passing the configuration file using the --config flag:

```
kind create cluster --config kind-calico.yaml
```

<img width="941" alt="Screenshot 2022-05-06 at 11 16 49" src="https://user-images.githubusercontent.com/82048393/167113347-f783ac08-a65e-4da2-b278-ea71d28783d0.png">



## Verify Kind Cluster

Once the cluster is up, list the pods in the ```kube-system``` namespace to verify that ```kindnet``` is not running:

```
kubectl get pods -n kube-system
```

<img width="941" alt="Screenshot 2022-05-06 at 11 18 45" src="https://user-images.githubusercontent.com/82048393/167113575-2318c196-7760-472e-9877-41b93476b614.png">


```kindnet``` should be missing from the list of pods:

```Note:``` The coredns pods are in the ```pending``` state. This is expected. <br/>
They will remain in the pending state until a CNI plugin is installed.



## Install Calico CNI

Install the Tigera Calico operator and custom resource definitions.
```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```

<img width="962" alt="Screenshot 2022-05-06 at 11 20 27" src="https://user-images.githubusercontent.com/82048393/167114358-2cf3587f-39e9-4186-8e1e-e267703ccb83.png">


Install Calico by creating the necessary custom resource.
```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```

<img width="962" alt="Screenshot 2022-05-06 at 11 20 45" src="https://user-images.githubusercontent.com/82048393/167114329-61010589-ddd3-4b1f-bafd-d00d17dac224.png">


Confirm that all of the pods are running with the following command.
```
watch kubectl get pods -n calico-system
```

Wait until each pod has the ```STATUS``` of ```Running```.

<img width="962" alt="Screenshot 2022-05-06 at 11 21 46" src="https://user-images.githubusercontent.com/82048393/167114314-313d51e5-00d4-4a01-8f9f-8e014615abce.png">


Congratulations! You now have an Rancher Desktop cluster running Calico <br/>
As expected, those coredns pods are now in a ```Running``` state after the CNI install

<img width="962" alt="Screenshot 2022-05-06 at 11 24 45" src="https://user-images.githubusercontent.com/82048393/167114574-6fa160f5-9f6d-4c85-8284-19905cd6d50c.png">


## Install a test application (Storefront)


```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Confirm your test application is running:
```
kubectl get pods -n storefront
```

<img width="962" alt="Screenshot 2022-05-06 at 11 27 10" src="https://user-images.githubusercontent.com/82048393/167114891-87fed43a-87bd-4bd8-a67e-d3f8f4d55f82.png">
