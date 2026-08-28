# Install a dynamic volume provisioner to automatically create PersistentVolumes (PVs) when you create a PersistentVolumeClaim (PVC).

For a Kubeadm cluster on AWS EC2, choose one of the two approaches below:

## Option 1: Local Path Provisioner (Recommended for Labs)

The **Rancher Local Path Provisioner** is a single-command setup that dynamically provisions hostPath storage on worker nodes.

**1. Deploy the provisioner**

```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

**2. Set it as the default StorageClass**

```
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**3. Verify**

```
kubectl get sc
```

*Output will show local-path (default).*

## Option 2: AWS EBS CSI Driver (Production-Grade for AWS)

The **AWS EBS CSI Driver** dynamically creates real AWS EBS volumes when PVCs are requested.

**1. Prerequisites on AWS EC2**

Ensure the IAM Role attached to your EC2 worker nodes has the AWS-managed policy attached:

- AmazonEBSCSIDriverPolicy

**2. Install the AWS EBS CSI Driver**

```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.30"
```

**3. Create an EBS StorageClass**

```
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
spec:
  provisioner: ebs.csi.aws.com
  volumeBindingMode: WaitForFirstConsumer
  parameters:
    type: gp3
EOF
```

**Verification Test**

Deploy a test PVC to confirm dynamic provisioning is active:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc-test
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
```

Check the status:

```
kubectl get pvc dynamic-pvc-test
```

*(If using local-path or WaitForFirstConsumer, it will bind as soon as a Pod mounts it).*


















