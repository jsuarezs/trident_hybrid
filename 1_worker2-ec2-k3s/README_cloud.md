# Setup and testing on AWS - Importing from on-premises

In order to use our NetApp Cloud Volumes ONTAP (CVO) as persistent storage we need to deploy an AWS EC2 which will be a worker k3s node. 

- So the first step should be deploy a simple k3s node in our EC2 instance. These steps are the same like in on-premises so you can check it [here](/README.md).

- I will create a logical resource space for Trident inside k3s:
```kubectl create namespace trident```
- Next step will be download and install Trident in trident namespace. The latest Trident version when this guide was done was 21.04.1 but check this as Trident is always evolving.
```
wget https://github.com/NetApp/trident/releases/download/v21.04.1/trident-installer-21.04.1.tar.gz

tar -xf trident-installer-21.04.1.tar.gz

cd trident-installer

./tridentctl install -n trident

root@ip-172-30-14-251:/home/ubuntu# kubectl get pods -n trident
NAME                           READY   STATUS    RESTARTS   AGE
trident-csi-6466f477c6-8dvjm   6/6     Running   0          9d
trident-csi-zzl4k              2/2     Running   0          9d
```

If the above process failed check if you have enough compute resources to deploy Trident as sometimes going with little memory configurations will fail.

## Create Trident storage backends on AWS using Cloud Volumes ONTAP (CVO)

As container orchestration systems like Kubernetes doesn't speak bout storage backends, Trident is taking advantage of the market standard Container Storage Interface (CSI) to provision persistent storage to stateful applications.

Trident supports NAS and SAN backend protocols. In this case NAS is configured to serve data using NFS protocol.

- So the first step would be to create the NAS backend like on-premises using ontap-nas driver instead of ontap-san. Check the files whatever if you're looking for deploying single or HA NetApp CVO.

Take notes that I'm also using NetApp storage efficiencies setting ```spaceReserve to none``` and one dedicated Storage Virtual Machine (SVM) was also deployed for Trident taking advantages of NetApp Multi-Tenancy (SMT) for security reasons.

The user ```k3saws``` is also dedicated to Trident's SVM with the rights to create volumes in only this SVM and ```dataLIF``` is the IP servicing data in this SVM. The best way to get this data is from Cloud Manager, inside the CVO environment and check for the correct ```dataLIF``` to serve.

```
cd trident-installer

./tridentctl create backend --filename backend-cloud.json -n trident
```
NOTES to create the backend in NetApp HA-CVO using Trident:


In case

## Create Trident storage classes

When a user creates a PVC that refers to a Trident-based StorageClass, Trident will provision a new volume using the corresponding storage class and register a new PV for that volume.

- So the step should be create the StorageClass.

```
kubectl apply -f sc-nas-cloud.yaml
```
Take notes that it's mandatory to specify the provisioner and in this case it's also mentioned to reference this StorageClass with the ontap-nas backend created before.

Then we're ready to the final step to import the volume which was Snapmirrored from on-premise to NetApp CVO using ```Trident import```.

```
./tridentctl import volume BackendForNAS trident_pvc_34a9dcc0_e4cd_4d81_9248_8ad5f5bfd19e -f vol-nas-cloud.yaml -n trident
```