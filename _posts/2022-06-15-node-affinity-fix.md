---
title: "Kubernetes Pod Warning: 1 node(s) had volume node affinity conflict"
author: ronald_dc
categories: [Kubernetes]
tags: [EKS, Kubernetes]
---

**0\. If you didn't find the solution in other answers...**
-----------------------------------------------------------

In our case the error happened on a AWS EKS cluster freshly provisioned with Pulumi (see [full source here](https://github.com/jonashackt/tekton-argocd-eks/blob/main/eks-deployment/index.ts)). The error drove me nuts, since I didn't change anything, just created a `PersistentVolumeClaim` [as described in the Buildpacks Tekton docs](https://buildpacks.io/docs/tools/tekton/#41-pvcs):

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: buildpacks-source-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi

```

I didn't change anything else from the default EKS configuration and also didn't add/change any `PersistentVolume` or `StorageClass` (in fact I didn't even know how to do that). As the default EKS setup seems to rely on 2 nodes, I got the error:

```
0/2 nodes are available: 2 node(s) had volume node affinity conflict.

```

Reading through [Sownak Roy's answer](https://stackoverflow.com/a/55514852/4964553) I got a first glue what to do - but didn't know *how to do it*. So for the folks interested **here are all my steps to resolve the error**:

>  **_Warning:_** This will delete all your data, make sure you have backups. 

**1\. Check EKS nodes `failure-domain.beta.kubernetes.io` labels**
------------------------------------------------------------------

As described in the section `Statefull applications` [in this post](https://vorozhko.net/120-days-of-aws-eks-kubernetes-in-staging) two nodes are provisioned on other AWS availability zones as the persistent volume (PV), which is created by applying our `PersistendVolumeClaim` described above.

To check that, you need to look into/describe your nodes with `kubectl get nodes`:

```
$ kubectl get nodes
NAME                                             STATUS   ROLES    AGE     VERSION
ip-172-31-10-186.eu-central-1.compute.internal   Ready    <none>   2d16h   v1.21.5-eks-bc4871b
ip-172-31-20-83.eu-central-1.compute.internal    Ready    <none>   2d16h   v1.21.5-eks-bc4871b

```

and then have a look at the `Label` section using `kubectl describe node <node-name>`:

```
$ kubectl describe node ip-172-77-88-99.eu-central-1.compute.internal
Name:               ip-172-77-88-99.eu-central-1.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t2.medium
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=eu-central-1
                    failure-domain.beta.kubernetes.io/zone=eu-central-1b
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-172-77-88-99.eu-central-1.compute.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=t2.medium
                    topology.kubernetes.io/region=eu-central-1
                    topology.kubernetes.io/zone=eu-central-1b
Annotations:        node.alpha.kubernetes.io/ttl: 0
...

```

In my case the node `ip-172-77-88-99.eu-central-1.compute.internal` has `failure-domain.beta.kubernetes.io/region` defined as `eu-central-1` and the az with `failure-domain.beta.kubernetes.io/zone` to `eu-central-1b`.

And the other node defines `failure-domain.beta.kubernetes.io/zone` az `eu-central-1a`:

```
$ kubectl describe nodes ip-172-31-10-186.eu-central-1.compute.internal
Name:               ip-172-31-10-186.eu-central-1.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t2.medium
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=eu-central-1
                    failure-domain.beta.kubernetes.io/zone=eu-central-1a
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-172-31-10-186.eu-central-1.compute.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=t2.medium
                    topology.kubernetes.io/region=eu-central-1
                    topology.kubernetes.io/zone=eu-central-1a
Annotations:        node.alpha.kubernetes.io/ttl: 0
...

```

**2\. Check `PersistentVolume`'s `topology.kubernetes.io` field**
-----------------------------------------------------------------

Now we should check the `PersistentVolume` automatically provisioned after we manually applied our `PersistentVolumeClaim`. Use `kubectl get pv`:

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS   REASON   AGE
pvc-93650993-6154-4bd0-bd1c-6260e7df49d3   1Gi        RWO            Delete           Bound    default/buildpacks-source-pvc   gp2                     21d

```

followed by `kubectl describe pv <pv-name>`

```
$ kubectl describe pv pvc-93650993-6154-4bd0-bd1c-6260e7df49d3
Name:              pvc-93650993-6154-4bd0-bd1c-6260e7df49d3
Labels:            topology.kubernetes.io/region=eu-central-1
                   topology.kubernetes.io/zone=eu-central-1c
Annotations:       kubernetes.io/createdby: aws-ebs-dynamic-provisioner
...

```

The `PersistentVolume` was configured with the label `topology.kubernetes.io/zone` in az `eu-central-1c`, **which makes our Pods complain about not finding their volume - since they are in a completely different az!**

**3\. Add `allowedTopologies` to `StorageClass`**
-------------------------------------------------

As [stated in the Kubernetes docs](https://kubernetes.io/docs/concepts/storage/storage-classes/#allowed-topologies) one solution to the problem is to add a `allowedTopologies` configuration to the `StorageClass`. If you already provisioned a EKS cluster like me, you need to retrieve your already defined `StorageClass` with

```
kubectl get storageclasses gp2 -o yaml

```

Save it to a file called `storage-class.yml` and add a `allowedTopologies` section that matches your node's `failure-domain.beta.kubernetes.io` labels like this:

```
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - eu-central-1a
    - eu-central-1b

```

The `allowedTopologies` configuration defines that the `failure-domain.beta.kubernetes.io/zone` of the `PersistentVolume` must be either in `eu-central-1a` or `eu-central-1b` - not `eu-central-1c`!

The full `storage-class.yml` looks like this:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
parameters:
  fsType: ext4
  type: gp2
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - eu-central-1a
    - eu-central-1b

```

Apply the enhanced `StorageClass` configuration to your EKS cluster with

```
kubectl apply -f storage-class.yml

```

**4\. Delete `PersistentVolumeClaim`, add `storageClassName: gp2` to it and re-apply it**
-----------------------------------------------------------------------------------------

In order to get things working again, we need to delete the `PersistentVolumeClaim` first.

To map the `PersistentVolumeClaim` to our previously define `StorageClass` we need to add `storageClassName: gp2` to the PersistendVolumeClaim definition in our `pvc.yml`:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: buildpacks-source-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: gp2

```

Finally re-apply the `PersistentVolumeClaim` with `kubectl apply -f pvc.yml`. This should resolve the error.