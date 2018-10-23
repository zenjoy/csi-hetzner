A Container Storage Interface ([CSI](https://github.com/container-storage-interface/spec)) Driver for Hetzner Cloud Volumes. The CSI plugin allows you to use Hetzner Cloud Volumes with your preferred Container Orchestrator.

The Hetzner CSI plugin is only tested on Kubernetes and is highly experimental. Theoretically it
should also work on other Container Orchestrator's, such as Mesos or
Cloud Foundry.

Current limitations are:

* Volumes need to be minimum 10Gi
* Maximum 5 volumes can be attached to a single server

## Acknowledgement

The code for this driver was adapted from the [Container Storage Interface (CSI) Driver for DigitalOcean Block Storage](https://github.com/digitalocean/csi-digitalocean) and modified to work with the Hetzner Cloud API. Many thanks for their work on the CSI Driver!

## Installing to Kubernetes

**Requirements:**

* Kubernetes v1.12+
* enabling feature gates `--feature-gates=VolumeSnapshotDataSource=true,KubeletPluginsWatcher=true,CSINodeInfo=true,CSIDriverRegistry=true
`
* `--allow-privileged` flag must be set to true for both the API server and the kubelet
* (if you use Docker) the Docker daemon of the cluster nodes must allow shared mounts

#### 1. Create a secret with your Hetzner Cloud API Token:

Replace the placeholder string starting with `__REPLACE_ME__` with your own secret and
save it as `secret.yml`: 

```
apiVersion: v1
kind: Secret
metadata:
  name: hetzner
  namespace: kube-system
stringData:
  access-token: "__REPLACE_ME__"
```

and create the secret using kubectl:

```
$ kubectl create -f ./secret.yml
secret "hetzner" created
```

You should now see the hetzner secret in the `kube-system` namespace along with other secrets

```
$ kubectl -n kube-system get secrets
NAME                  TYPE                                  DATA      AGE
default-token-jskxx   kubernetes.io/service-account-token   3         18h
hetzner          Opaque                                1         18h
```

#### 2. Deploy the CSI plugin:

(instructions are from the [Kubernetes v1.12 CSI Driver guides](https://kubernetes-csi.github.io/docs/Setup.html))

Enable the CSIDriver:

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/csi-api/master/pkg/crd/testdata/csidriver.yaml --validate=false
```

If your cluster uses RBAC, create the applicable cluster roles:

```
$ kubectl create -f https://raw.githubusercontent.com/kubernetes-csi/docs/master/book/src/example/rbac/csi-provisioner-rbac.yaml
$ kubectl create -f https://raw.githubusercontent.com/kubernetes-csi/docs/master/book/src/example/rbac/csi-attacher-rbac.yaml
$ kubectl create -f https://raw.githubusercontent.com/kubernetes-csi/docs/master/book/src/example/rbac/csi-nodeplugin-rbac.yaml
$ kubectl create -f https://raw.githubusercontent.com/kubernetes-csi/docs/master/book/src/example/snapshot/csi-snapshotter-rbac.yaml
```

Last, deploy the Hetzner Cloud Volume CSI Driver:
```
$ kubectl apply -f https://raw.githubusercontent.com/zenjoy/csi-hetzner/master/deploy/kubernetes/releases/csi-hetzner-dev.yaml
```

A new storage class will be created with the name `hc-block-storage` which is
responsible for dynamic provisioning. This is set to **"default"** for dynamic
provisioning. If you're using multiple storage classes you might want to remove
the annotation from the `csi-storageclass.yaml` and re-deploy it. This is
based on the [recommended mechanism](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes) of deploying CSI drivers on Kubernetes

*Note that the deployment proposal to Kubernetes is still a work in progress and not all of the written
features are implemented. When in doubt, open an issue or ask #sig-storage in [Kubernetes Slack](http://slack.k8s.io)*

#### 3. Test and verify:

Create a PersistentVolumeClaim. This makes sure a volume is created and provisioned on your behalf:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: hc-block-storage
```

Check that a new `PersistentVolume` is created based on your claim:

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM             STORAGECLASS       REASON    AGE
pvc-0879b207-9558-11e8-b6b4-5218f75c62b9   10Gi        RWO            Delete           Bound     default/csi-pvc   hc-block-storage             3m
```

The volume is not attached to any node yet. It'll only attached to a node if a
workload (i.e: pod) is scheduled to a specific node. Now let us create a Pod
that refers to the above volume. When the Pod is created, the volume will be
attached, formatted and mounted to the specified Container:

```
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-hc-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-hc-volume
      persistentVolumeClaim:
        claimName: csi-pvc 
```

Check if the pod is running successfully:


```
$ kubectl describe pods/my-csi-app
```

Write inside the app container:

```
$ kubectl exec -ti my-csi-app /bin/sh
/ # touch /data/hello-world
/ # exit
$ kubectl exec -ti my-csi-app /bin/sh
/ # ls /data
hello-world
```