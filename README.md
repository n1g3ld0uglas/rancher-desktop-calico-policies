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

We provided a deployment file for creating your test application
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Confirm your test application is running:
```
kubectl get pods -n storefront
```

<img width="962" alt="Screenshot 2022-05-06 at 11 27 10" src="https://user-images.githubusercontent.com/82048393/167114891-87fed43a-87bd-4bd8-a67e-d3f8f4d55f82.png">

Due to the ephemeral nature of Kubernetes, the IP address of a pod is not long lived. <br/>
As a result, it makes more sense for us to target pods based on a consistent label schema (not based on IP address):

<img width="1135" alt="Screenshot 2022-05-06 at 11 52 01" src="https://user-images.githubusercontent.com/82048393/167118571-fda5c06e-224e-4a16-a0c0-e22ec82e3697.png">

To see the label schema associated with your storefront pods, run the below command:
```
kubectl get pods -n storefront --show-labels
```


## Creating your first network policies

As a best practice, we will implement a zone-based architecture via Calico's Networking & Security Policies <br/>
Using a zone-based firewall approach allows us to apply the said security policies to the security zones instead of the pods <br/>
Then, the labelled pods are set as members of the different zones - if they are not in a correct zone, the packets will be dropped.

![container-firewall](https://user-images.githubusercontent.com/82048393/167121017-0e9a68c9-0c50-4063-b211-cfb3c843f866.png)


#### Demilitarized Zone (DMZ) Policy

The following example allows ```Ingress``` traffic from the public internet CIDR net ```18.0.0.0/16``` <br/>
All other Ingress-related traffic will be ```Denied``` - this includes traffic sent to the DMZ from pods within the cluster. <br/>
<br/>
It's important to note that the ```DMZ``` labelled pods can ```Egress``` out to the pods within the ```Trusted``` zone, or if it has a label of ```app=logging``` <br/>
All other outbound traffic from ```DMZ``` zone will be dropped as part of this zero-trust initiative.

```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-desktop-calico-policies/main/dmz.yaml
```
<img width="1135" alt="Screenshot 2022-05-06 at 11 49 04" src="https://user-images.githubusercontent.com/82048393/167117911-83a788bf-c0d5-4433-abd9-dcea98eac8d5.png">

#### Trusted Zone Policy

```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-desktop-calico-policies/main/trusted.yaml
```

<img width="1167" alt="Screenshot 2022-05-06 at 11 59 23" src="https://user-images.githubusercontent.com/82048393/167119339-b4fbd596-11c9-4e94-b368-b593293bf056.png">

#### Restricted Zone Policy

```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-desktop-calico-policies/main/restricted.yaml
```

<img width="1179" alt="Screenshot 2022-05-06 at 12 01 28" src="https://user-images.githubusercontent.com/82048393/167119567-8a967705-f45f-465a-8680-a598f4610b3b.png">

#### Default-Deny Policy

And finally, to absolutely ensure zero-trust workload security is implemented for the storefront namespace, we create a default-deny policy <br/>
Default deny policy ensures pods without policy (or incorrect policy) are not allowed traffic until appropriate network policy is defined. <br/>
https://projectcalico.docs.tigera.io/security/kubernetes-default-deny


```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-desktop-calico-policies/main/default-deny.yaml
```

<img width="1186" alt="Screenshot 2022-05-06 at 12 06 18" src="https://user-images.githubusercontent.com/82048393/167120572-d87298cb-7024-4765-a2e9-1c823b0ce1a5.png">


## Optional: Connecting to Calico Cloud:
You can test Calico Cloud for FREE for 14 days via ```calicocloud.io``` <br/>
https://docs.calicocloud.io/get-started/connect/install-cluster

![Screenshot 2022-05-06 at 12 33 15](https://user-images.githubusercontent.com/82048393/167124630-41c5c828-6acb-4d27-b61f-19605f1a1389.png)

After you give a name to your cluster, you will run a single line install script:

<img width="1193" alt="Screenshot 2022-05-06 at 12 34 35" src="https://user-images.githubusercontent.com/82048393/167124657-8793d472-193d-44b5-9bfe-5b9a50ee801e.png">

You can follow the progress of the installation via:

```
kubectl get pods -A -w
```
<img width="1193" alt="Screenshot 2022-05-06 at 12 34 46" src="https://user-images.githubusercontent.com/82048393/167124800-cb1c5732-1037-47f3-8447-123ca19beb8b.png">

The entire process can take between 2-4 mins to complete:

<img width="1193" alt="Screenshot 2022-05-06 at 12 35 23" src="https://user-images.githubusercontent.com/82048393/167124852-9981ceb4-02a4-4b88-8e69-e96424f024a6.png">

The process is completed when you can see all pods in a ```Ready``` state

<img width="1193" alt="Screenshot 2022-05-06 at 12 38 54" src="https://user-images.githubusercontent.com/82048393/167124985-6040dfab-c74e-4946-b731-29a9ba975966.png">

You can now see the allowed and denied traffic per policy in the Calico Cloud web user interface:

![Screenshot 2022-05-06 at 12 39 37](https://user-images.githubusercontent.com/82048393/167125019-8b7e4268-598f-4f65-acdc-f177c4148ca1.png)

## Creating higher-level security rules in Calico Cloud:

Since we are now making use of the enterprise features, we can create our first tier - ```security``` <br/>
Policies in the ```security``` tier are read in order of precedence before policies within the ```default``` tier:

```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-desktop-calico-policies/main/security-tier.yaml
```

<img width="1199" alt="Screenshot 2022-05-06 at 12 49 10" src="https://user-images.githubusercontent.com/82048393/167126161-f0e2b3f9-0f4e-4ad9-a1e8-461cb46e65aa.png">


