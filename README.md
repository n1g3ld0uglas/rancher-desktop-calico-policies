# rancher-desktop-calico-policies

I avoided using the built-in Kubernetes component of Rancher Desktop. <br/>
I unchecked this option and switched the container runtime from ```containerd``` to ```dockerd```:

<img width="340" alt="Screenshot 2022-05-06 at 10 51 34" src="https://user-images.githubusercontent.com/82048393/167109172-e5d82494-0a00-4002-b9f5-a00fd6c22cce.png">

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

## Install KinD
