## Setup and testing on-premises

In order to use our NetApp ONTAP cluster as persistent storage we need to deploy a VMWare VM or some other host which will be a worker k3s node. 

- So the first step would be to deploy a simple k3s node in our VM.
```
curl -sfL https://get.k3s.io | sh -
# Check for Ready node, takes maybe 30 seconds
k3s kubectl get node

```
- I will create a logical resource space for Trident inside k3s:
```kubectl create namespace trident```
- Next step will be download and install Trident in trident namespace. The latest Trident version when this guide was done was 21.04.1 but check this as Trident is always evolving.
```
wget https://github.com/NetApp/trident/releases/download/v21.04.1/trident-installer-21.04.1.tar.gz

tar -xf trident-installer-21.04.1.tar.gz

cd trident-installer

./tridentctl install -n trident
```

If the above process failed 