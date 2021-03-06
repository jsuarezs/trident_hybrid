# Setup and testing on-premises

In order to use our NetApp ONTAP system as persistent storage we need to deploy a VMWare VM or some other host which will be a worker k3s node. 

- So the first step should be deploy a simple k3s node in our VM.
```
curl -sfL https://get.k3s.io | sh -
# Check for Ready node, takes maybe 30 seconds
k3s kubectl get node

root@worker1-virtual-machine:~# kubectl get nodes
NAME                      STATUS   ROLES                  AGE   VERSION
worker1-virtual-machine   Ready    control-plane,master   11d   v1.21.1+k3s1

```
- I will create a logical resource space for Trident inside k3s:
```kubectl create namespace trident```
- Next step will be download and install Trident in trident namespace. The latest Trident version when this guide was done was 21.04.1 but check this as Trident is always evolving.
```
wget https://github.com/NetApp/trident/releases/download/v21.04.1/trident-installer-21.04.1.tar.gz

tar -xf trident-installer-21.04.1.tar.gz

cd trident-installer

./tridentctl install -n trident

root@worker1-virtual-machine:~# kubectl get pods -n trident
NAME                           READY   STATUS    RESTARTS   AGE
trident-csi-6466f477c6-nxn49   6/6     Running   12         11d
trident-csi-fln7v              2/2     Running   4          11d
```

If the above process failed check if you have enough compute resources to deploy Trident as sometimes going with little memory configurations will fail.

## Create Trident storage backends on-premises using NetApp ONTAP system

As container orchestration systems like Kubernetes doesn't speak bout storage backends, Trident is taking advantage of the market standard Container Storage Interface (CSI) to provision persistent storage to stateful applications.

Trident supports NAS and SAN backend protocols. In this case NAS is configured to serve data using NFS protocol.

- So the first step would be to create the NAS backend on-premises using ontap-nas driver instead of ontap-san.

```
cd trident-installer

./tridentctl create backend --filename backend-nas.json -n trident

root@worker1-virtual-machine:/home/worker1/trident-installer# ./tridentctl get backend -n trident
+---------------+----------------+--------------------------------------+--------+---------+
|     NAME      | STORAGE DRIVER |                 UUID                 | STATE  | VOLUMES |
+---------------+----------------+--------------------------------------+--------+---------+
| BackendForNAS | ontap-nas      | e49a08fb-995d-4641-a127-8c06700c1962 | online |       2 |
+---------------+----------------+--------------------------------------+--------+---------+
```
Take notes that I'm also using NetApp storage efficiencies setting ```spaceReserve to none``` and one dedicated Storage Virtual Machine (SVM) was also deployed for Trident taking advantages of NetApp Multi-Tenancy (SMT) for security reasons.

The user ```openshift``` is also dedicated to Trident's SVM with the rights to create volumes in only this SVM and ```dataLIF``` is the IP servicing data in this SVM.

## Create Trident storage classes

When a user creates a PVC that refers to a Trident-based StorageClass, Trident will provision a new volume using the corresponding storage class and register a new PV for that volume.

- So the step should be create the StorageClass.

```
kubectl apply -f sc-nas.yaml

root@worker1-virtual-machine:~# kubectl get sc
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nas                    csi.trident.netapp.io   Retain          Immediate              true                   11d
```
Take notes that it's mandatory to specify the provisioner and in this case it's also mentioned to reference this StorageClass with the ontap-nas backend created before.

Then we're ready to provision our first volume using Trident, in this case in my application namespace. Take notes that I'm also setting a Snapshot policy to create point-in-time copies using my default policy.
```
kubectl create ns app
kubectl apply -f vol-nas.yaml -n app
```

## Create PV using Trident

Now we can check that PVC was already created and the associated PV in ONTAP cluster. This volume will be Snapmirrored to NetApp CVO in AWS.

```
root@worker1-virtual-machine:/home/worker1/trident-installer# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM      STORAGECLASS   REASON   AGE
pvc-34a9dcc0-e4cd-4d81-9248-8ad5f5bfd19e   2Gi        RWO            Retain           Bound    app/vol2   nas                     32d


clusterlab::> vol show -vserver SVM_OpenShift 
Vserver   Volume       Aggregate    State      Type       Size  Available Used%
--------- ------------ ------------ ---------- ---- ---------- ---------- -----
SVM_OpenShift SVM_OpenShift_root aggr_nodo01_DATOS online RW 1GB  970.8MB    0%
SVM_OpenShift trident_pvc_34a9dcc0_e4cd_4d81_9248_8ad5f5bfd19e aggr_nodo02_DATOS online RW 2GB 2.00GB  0%
```

## How to create On-Demand PVC snapshots

Beginning with the 20.01 release of Trident, it is now possible to use the beta Volume Snapshot feature to create snapshots of PVs at the Kubernetes layer. 

These snapshots can be used to maintain point-in-time copies of volumes that have been created by Trident and can also be used to schedule the creation of additional volumes (clones). This feature is available for Kubernetes 1.17 and above.

Before creating a Volume Snapshot, a VolumeSnapshotClass must be set up -> ```volumesnapshotclass.yaml``` and create the Snapshot using ```kubectl create -f snap.yaml```.

This esentially will create a Snapshot fot the volume called ```vol1``` and rename the Snapshot to ```pvc1-snap```.



