# EFS with Amazon EKS Setup Guide

This repository provides a step-by-step guide to set up Amazon Elastic File System (EFS) with Amazon Elastic Kubernetes Service (EKS).

## Prerequisites

- AWS CLI configured with appropriate permissions
- kubectl installed and configured
- eksctl installed
- An existing EKS cluster
- AWS IAM Authenticator

## Architecture

```plaintext
EKS Cluster
├── Node Group
│   ├── Worker Node 1
│   │   └── Pod(s) ─────┐
│   └── Worker Node 2   │
│       └── Pod(s) ─────┼──► EFS File System
└── EFS CSI Driver      │
    └── Storage Class ──┘
```

## Installation Steps

### 1. Create an EFS File System

```bash
# Get VPC ID
export VPC_ID=$(aws eks describe-cluster --name your-cluster-name --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Get CIDR
export CIDR=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[].CidrBlock" --output text)

# Create Security Group
export EFS_SG=$(aws ec2 create-security-group \
    --group-name MyEFSSG \
    --description "Security Group for EFS" \
    --vpc-id $VPC_ID \
    --output text)

# Add inbound rule
aws ec2 authorize-security-group-ingress \
    --group-id $EFS_SG \
    --protocol tcp \
    --port 2049 \
    --cidr $CIDR

# Create EFS filesystem
aws efs create-file-system \
    --creation-token eks-efs \
    --performance-mode generalPurpose \
    --throughput-mode bursting \
    --tags Key=Name,Value=MyEKSEFS
```

### 2. Install EFS CSI Driver

```bash
# Add the AWS EFS CSI Driver Helm repository
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/

# Update Helm repositories
helm repo update

# Install the EFS CSI Driver
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver
```

### 3. Create Storage Class

```yaml
# storage-class.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxxx # Replace with your EFS ID
  directoryPerms: "700"
```

Apply the storage class:
```bash
kubectl apply -f storage-class.yaml
```

### 4. Create PVC and Pod

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: efs-app
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim
```

Apply the configurations:
```bash
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
```

## Verification

```bash
# Check if PVC is bound
kubectl get pvc

# Check if pod is running
kubectl get pods

# Check the contents of the file
kubectl exec efs-app -- cat /data/out.txt
```

## Cleanup

```bash
# Delete the pod and PVC
kubectl delete pod efs-app
kubectl delete pvc efs-claim

# Delete the storage class
kubectl delete sc efs-sc

# Delete the EFS filesystem
aws efs delete-file-system --file-system-id fs-xxxxxx

# Delete the security group
aws ec2 delete-security-group --group-id $EFS_SG
```

## Troubleshooting

Common issues and solutions:

1. **Mount failed**: Check security group settings
2. **CSI driver not working**: Verify IAM roles and policies
3. **PVC stuck in pending**: Check storage class configuration

## References

- [AWS EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
- [EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EFS Documentation](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)
