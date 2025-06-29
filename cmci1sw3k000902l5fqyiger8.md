---
title: "High-performance EKS storage with Amazon FSx for Lustre"
datePublished: Sun Jun 29 2025 19:13:36 GMT+0000 (Coordinated Universal Time)
cuid: cmci1sw3k000902l5fqyiger8
slug: high-performance-eks-storage-with-amazon-fsx-for-lustre
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1750970648761/1d1137f0-fef5-4da8-88e6-002370386445.png
tags: aws, eks, fsx-for-lustre, persistentvolumes

---

[Amazon FSx](https://docs.aws.amazon.com/en_us/fsx/) is a fully managed service that lets users launch third-party high-performance file systems on AWS. It is designed to offer reliable, scalable, and secure file storage that works with various workloads and applications. Amazon FSx provides popular and feature-rich file systems, each designed for specific needs:

* [**Amazon FSx for Windows File Server**](https://docs.aws.amazon.com/fsx/latest/WindowsGuide/what-is.html)**:** This option provides a fully managed, native Microsoft Windows file system. It is ideal for lifting and shifting Windows-based applications and workloads to the cloud. It supports the Server Message Block (SMB) protocol and integrates seamlessly with Microsoft Active Directory.
    
* [**Amazon FSx for Lustre**](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)**:** Designed for high-performance computing (HPC), machine learning, media processing, and financial modeling workloads. FSx for Lustre is a parallel file system built for speed and scalability. It can provide sub-millisecond latencies, hundreds of gigabytes per second of throughput, and millions of IOPS.
    
* [**Amazon FSx for NetApp ONTAP**](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/what-is-fsx-ontap.html)**:** This service brings the popular NetApp ONTAP file system to AWS, offering a familiar and feature-rich data management experience. It supports multiple protocols, including NFS, SMB, and iSCSI, and provides advanced features such us snapshots, cloning, and replication.
    
* [**Amazon FSx for OpenZFS**](https://docs.aws.amazon.com/fsx/latest/OpenZFSGuide/what-is-fsx.html)**:** Built on the open-source OpenZFS file system, this option provides a high-performance, cost-effective file storage solution for a wide range of Linux-based workloads.
    

So, why is Amazon FSx important for [EKS](https://docs.aws.amazon.com/eks/latest/userguide/storage.html)? While most applications running in EKS are stateless, sometimes we need to store data and keep it even after a pod's lifetime. To store data beyond a pod's lifetime, EKS uses [Persistent Volumes (PVs)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), which offer different types of storage options (we can find the complete list [here](https://docs.aws.amazon.com/eks/latest/userguide/storage.html)). One of these options is Amazon FSx for Lustre.

## Pre-requisites

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI.](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console)
    
* Install the [EKSCTL CLI](https://github.com/eksctl-io/eksctl).
    
* Install [Helm CLI](https://helm.sh/docs/intro/install/)**.**
    

## Installing the Amazon FSx CSI Driver

To allow EKS to manage the lifecycle of Amazon FSx for Lustre, we need to install the [Amazon FSx for Lustre Container Storage Interface (CSI) driver](https://github.com/kubernetes-sigs/aws-fsx-csi-driver). The driver requires IAM permissions to interact with Amazon FSx on our behalf. Run the following command:

```powershell
eksctl create iamserviceaccount --name fsx-csi-controller-sa --namespace kube-system --cluster <MY_CLUSTER_NAME> --attach-policy-arn arn:aws:iam::aws:policy/AmazonFSxFullAccess --approve --role-name AmazonEKSFSxLustreCSIDriverFullAccess --region <MY_REGION> --override-existing-serviceaccounts
```

The command above will create an AWS IAM role named `AmazonEKSFSxLustreCSIDriverFullAccess` with the `AmazonFSxFullAccess` policy attached. It will also create a Kubernetes service account called `fsx-csi-controller-sa` linked to this role. With that setup, it's time to install the driver using [Helm](https://github.com/kubernetes-sigs/aws-fsx-csi-driver/tree/master):

```powershell
helm repo add aws-fsx-csi-driver https://kubernetes-sigs.github.io/aws-fsx-csi-driver
helm repo update
helm upgrade --install aws-fsx-csi-driver --namespace kube-system aws-fsx-csi-driver/aws-fsx-csi-driver --set controller.serviceAccount.create=false --set controller.serviceAccount.name=fsx-csi-controller-sa
```

We need to note that we are setting two values to interact correctly with the resources previously created:

* `controller.serviceAccount.name`: We set the driver's service account name to match the service account we already created.
    
* `controller.serviceAccount.create`: We specify not to create a new service account during the Helm package installation.
    

Once the Helm package is installed, we can check all the pods created for the driver using the following command:

```powershell
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-fsx-csi-driver
```

## File system access control

To manage network traffic through Amazon FSx for Lustre, we need to create a security group to control the access. First, run the following command to get the cluster's security group, VPC, and subnet ID:

```powershell
aws eks describe-cluster --name <MY_CLUSTER_NAME> --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text
aws eks describe-cluster --name <MY_CLUSTER_NAME> --query 'cluster.resourcesVpcConfig.vpcId' --output text
aws eks describe-cluster --name <MY_CLUSTER_NAME> --query 'cluster.resourcesVpcConfig.subnetIds[0]' --output text
```

Create a security group in the same VPC as the cluster:

```powershell
aws ec2 create-security-group --group-name eks-fsx-lustre-sg --description "Security group for FSx Lustre" --vpc-id <MY_VPC_ID> --query 'GroupId' --output text
```

Add the following inbound rules to the security group you created above:

```powershell
aws ec2 authorize-security-group-ingress --group-id <MY_SG_ID> --protocol tcp --port 988 --source-group <MY_CLUSTER_SG_ID>
aws ec2 authorize-security-group-ingress --group-id <MY_SG_ID> --protocol tcp --port 988 --source-group <MY_SG_ID>
aws ec2 authorize-security-group-ingress --group-id <MY_SG_ID> --protocol tcp --port 1021-1023 --source-group <MY_CLUSTER_SG_ID>
aws ec2 authorize-security-group-ingress --group-id <MY_SG_ID> --protocol tcp --port 1021-1023 --source-group <MY_SG_ID>
```

Amazon FSx for Lustre typically uses TCP port `988` and the range `1018-1023`. We need to set a few inbound rules using the cluster's security group as the source group, and another set using the same security group we created (this is for internal communication between Amazon FSx for Lustre components). Further information can be found [here](https://docs.aws.amazon.com/fsx/latest/LustreGuide/limit-access-security-groups.html).

## Dynamic provisioning

A [`StorageClass`](https://kubernetes.io/docs/concepts/storage/storage-classes/) is used for [dynamic provisioning](https://github.com/kubernetes-sigs/aws-fsx-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning#edit-storageclass) of PVs. Create a `storageclass.yaml` file with the following content:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fsx-lustre-sc
provisioner: fsx.csi.aws.com
parameters:
  subnetId: "MY_SUBNET_ID"
  securityGroupIds: "MY_SG_ID"
  deploymentType: "PERSISTENT_2"
  perUnitStorageThroughput: "500"
  fileSystemTypeVersion: "2.12"
reclaimPolicy: Delete
allowVolumeExpansion: false
volumeBindingMode: Immediate
```

The `provisioner: fsx.csi.aws.com` parameter tells the cluster to use the Amazon FSx CSI driver to provision the storage. The FSx-specific parameters are as follows:

* `subnetId`: The VPC subnet used by our cluster.
    
* `securityGroupIds`: The security group created in the previous step.
    
* `deploymentType`: Amazon FSx for Lustre offers different deployment types.
    
    * `PERSISTENT_1`: Older generation, lower throughput, HDD-backed metadata
        
    * `PERSISTENT_2`: Current generation with SSD-backed metadata, better performance and reliability
        
    * `SCRATCH_1` and `SCRATCH_2`: Temporary storage with no replication, much cheaper, but data can be lost.
        
* `perUnitStorageThroughput`: Defines the baseline throughput capacity per TiB of storage. For `PERSISTENT_2`, options are `125`, `250`, `500`, or `1000` MB/s per TiB.
    
* `fileSystemTypeVersion`: Specifies the Lustre software version.
    

It's time to request storage using a [PersistentVolumeClaim (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). Create a `pvc.yaml` file with the following content:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fsx-lustre-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: fsx-lustre-sc
  resources:
    requests:
      storage: 1200Gi
```

The PVC will stay in the `Pending` status for 5-10 minutes while the Amazon FSx for Lustre file system is being set up. Once it's ready, we can use our storage. Create a `job.yaml` file with the following content:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: quick-test
spec:
  template:
    spec:
      containers:
      - name: quick-test
        image: alpine/git:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            apk add --no-cache bash bc time
            
            echo "=== Quick FSx Git Performance Check ==="
            echo "Pod: $(hostname)"
            echo "Start time: $(date)"
            echo ""
            
            cd /mnt/fsx
            rm -rf quick-test
            mkdir -p quick-test
            cd quick-test
            
            echo "Testing git clone performance..."
            time git clone --depth 1 https://github.com/torvalds/linux.git test-repo
            
            echo "Testing git operations..."
            cd test-repo
            time git status
            time git log --oneline -50
            
            echo ""
            echo "Disk usage:"
            df -h /mnt/fsx
            du -sh /mnt/fsx/quick-test
            
            echo ""
            echo "=== Quick test completed ==="
        volumeMounts:
        - name: fsx-storage
          mountPath: /mnt/fsx
      volumes:
      - name: fsx-storage
        persistentVolumeClaim:
          claimName: fsx-lustre-claim
      restartPolicy: Never
  backoffLimit: 3
```

The script's main purpose is to measure the performance of the mounted storage. It works in a few key stages:

1. Install the necessary tools (`time`, `bc`) inside the container.
    
2. Deletes and creates a test directory (`/mnt/fsx/quick-test`) on the shared storage to make sure the test can be repeated.
    
3. Measures the time it takes to perform a high-I/O task: cloning Linux kernel source code from GitHub.
    
4. Measures the time for common `Git` operations such as `git status` and `git log`.
    
5. Displays the disk space usage for both the entire mounted volume and the newly created test directory.
    

Run the following commands to deploy the resource to EKS:

```powershell
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
kubectl apply -f job.yaml
```

This integration is more than just a technical task; it's an essential approach for running data-heavy applications on EKS. By letting AWS handle the complexity of managing a high-performance file system, we can concentrate on developing our applications while still meeting strict performance and persistence needs. You can find all the code [here](https://github.com/raulnq/eks-aws-fsx-for-lustre). Thanks, and happy coding.