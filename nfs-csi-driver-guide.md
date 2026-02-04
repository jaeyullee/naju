## 1. NFS 서버 준비
```
mkdir -p /export/k8s
chmod 777 /export/k8s
echo "/export/k8s *(rw,sync,no_root_squash)" >> /etc/exports
exportfs -rav
systemctl enable --now nfs-server
```

## 2. Kubernetes에 NFS CSI Driver 설치
```
$ helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```
```
$ podman pull registry.k8s.io/sig-storage/nfsplugin:v4.13.0
$ podman pull registry.k8s.io/sig-storage/csi-provisioner:v6.1.0
$ podman pull registry.k8s.io/sig-storage/csi-resizer:v2.0.0
$ podman pull registry.k8s.io/sig-storage/csi-snapshotter:v8.4.0
$ podman pull registry.k8s.io/sig-storage/livenessprobe:v2.17.0
$ podman pull registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.15.0
$ podman pull registry.k8s.io/sig-storage/snapshot-controller:v8.4.0

$ podman tag registry.k8s.io/sig-storage/nfsplugin:v4.13.0 nexus.kscada.kdneri.com:5002/sig-storage/nfsplugin:v4.13.0
$ podman tag registry.k8s.io/sig-storage/csi-provisioner:v6.1.0 nexus.kscada.kdneri.com:5002/sig-storage/csi-provisioner:v6.1.0
$ podman tag registry.k8s.io/sig-storage/csi-resizer:v2.0.0 nexus.kscada.kdneri.com:5002/sig-storage/csi-resizer:v2.0.0
$ podman tag registry.k8s.io/sig-storage/csi-snapshotter:v8.4.0 nexus.kscada.kdneri.com:5002/sig-storage/csi-snapshotter:v8.4.0
$ podman tag registry.k8s.io/sig-storage/livenessprobe:v2.17.0 nexus.kscada.kdneri.com:5002/sig-storage/livenessprobe:v2.17.0
$ podman tag registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.15.0 nexus.kscada.kdneri.com:5002/sig-storage/csi-node-driver-registrar:v2.15.0
$ podman tag registry.k8s.io/sig-storage/snapshot-controller:v8.4.0 nexus.kscada.kdneri.com:5002/sig-storage/snapshot-controller:v8.4.0

$ podman push nexus.kscada.kdneri.com:5002/sig-storage/nfsplugin:v4.13.0
$ podman push nexus.kscada.kdneri.com:5002/sig-storage/csi-provisioner:v6.1.0
$ podman push nexus.kscada.kdneri.com:5002/sig-storage/csi-resizer:v2.0.0
$ podman push nexus.kscada.kdneri.com:5002/sig-storage/csi-snapshotter:v8.4.0
$ podman push nexus.kscada.kdneri.com:5002/sig-storage/livenessprobe:v2.17.0
$ podman push nexus.kscada.kdneri.com:5002/sig-storage/csi-node-driver-registrar:v2.15.0
$ podman push nexus.kscada.kdneri.com:5002/sig-storage/snapshot-controller:v8.4.0
```
```
$ vi nfs-itms.yaml
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  annotations:
  name: nfs-csi-mirror
spec:
  imageTagMirrors:
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/snapshot-controller
    source: registry.k8s.io/sig-storage/snapshot-controller
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/nfsplugin
    source: registry.k8s.io/sig-storage/nfsplugin
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/csi-provisioner
    source: registry.k8s.io/sig-storage/csi-provisioner
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/csi-resizer
    source: registry.k8s.io/sig-storage/csi-resizer
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/csi-snapshotter
    source: registry.k8s.io/sig-storage/csi-snapshotter
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/livenessprobe
    source: registry.k8s.io/sig-storage/livenessprobe
  - mirrors:
    - nexus.kscada.kdneri.com:5002/sig-storage/csi-node-driver-registrar
    source: registry.k8s.io/sig-storage/csi-node-driver-registrar
```
```
$ oc create -f nfs-itms.yaml
```
```
$ helm pull csi-driver-nfs/csi-driver-nfs --untar

$ vi csi-driver-nfs/values.yaml
...
## StorageClass resource example: 부분 수정
storageClass:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  parameters:
    server: 192.168.10.40
    share: /nfs/nfs-csi
  reclaimPolicy: Delete
  volumeBindingMode: Immediate
  mountOptions:
    - nfsvers=4.1
...
```
```
$ helm package csi-driver-nfs/

$ curl --data-binary "@csi-driver-nfs-4.13.0.tgz" http://192.168.10.40:8089/api/charts
$ helm repo update
$ helm search repo my-private-repo/
```

```
$ oc new-project nfs-csi
$ oc adm policy add-scc-to-user privileged -z csi-nfs-node-sa -n nfs-csi
$ oc adm policy add-scc-to-user privileged -z csi-nfs-controller-sa -n nfs-csi

$ helm install csi-driver-nfs my-private-repo/csi-driver-nfs \
 --namespace nfs-csi --version 4.13.0
```

## 3. StorageClass 작성
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.0.0.5      # RHEL NFS 서버 IP
  share: /export/k8s    # export 경로
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
```

## 4. PVC 생성
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 5Gi
```
