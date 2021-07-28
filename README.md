# Working with NetApp Trident in Hybrid Cloud scenario
Learn how to deploy and use Trident on-premises and then move your Persistent Volumes (PV) smoothly to AWS using NetApp technologies.

This will provide persistence storage for statetul containers applications among of other features provided by NetApp Cloud Manager.

## Architecture


- In this scenario I'm using the following components for the solution:
    - One NetApp ONTAP cluster running on-premises with version 9.8P5.
    - One VMWare Ubuntu 20.04 64-bit VM working as a k3s worker node running on ESXi 6.7.
    - One t2.medium EC2 running Ubuntu 18.04 Bionic working as a k3s worker node on AWS.
    - One NetApp Cloud Volumes ONTAP (CVO) of your choice, could be a single-node or HA configuration running on AWS.
  
### Reference image showing NetApp ONTAP on-premises cluster replicating to Cloud Volumes ONTAP on AWS using NetApp SnapMirror

![cvo_snapmirror_aws](https://user-images.githubusercontent.com/59535705/127310359-0a1f99c5-6310-4688-b24d-79be4c5fb580.png)

### Reference image showing Cloud Volumes ONTAP in HA configuration on AWS connected to Elastic Container Services


![AWS_EKS](https://user-images.githubusercontent.com/59535705/127309092-f3e69e53-6a93-49c4-b7ba-50e51151974f.jpeg)


## Setup and testing

My reommendation is going around Trident resources before deploying it so you will understand all the concepts before going steps further -> [Trident docs](https://netapp-trident.readthedocs.io/en/stable-v21.04/)

Click [here](0_worker1-k3s/README_onprem.md) to follow the steps in order to deploy and install Trident using a NetApp ONTAP on-premises cluster as persistent storage. And [here](1_worker2-ec2-k3s/README_cloud.md) for cloud deployment with Cloud Volumes ONTAP (CVO).

