## 1. private helm repo 생성
```
$ podman run -d --name chartmuseum --rm -it -p 8089:8080 -v /opt/helm-repo:/charts -e DEBUG=true -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts chartmuseum/chartmuseum:latest
$ helm repo add my-private-repo http://192.168.10.40:8089
$ helm repo update
```

## 2. 헬름 차트 다운로드 및 설정변경
```
$ helm pull bitnami/zookeeper --untar
$ vi zookeeper/values.yaml
global:
  security:
    allowInsecureImages: true  ## 수정
image:
  registry: docker.io          ## idms/itms를 쓰려는경우 registry/repository를 나눠서 설정하는 것을 권장
  repository: bitnamilegacy/zookeeper
  tag: 3.9.3-debian-12-r22
  pullPolicy: IfNotPresent
replicaCount: 3                ## 수정
...
persistence:
  storageClass: ""             ## 비워두거나 "-"로 설정하여 동적 프로비저닝 방지 및 수동 PV 바인딩 유도

$ helm package zookeeper
```
> 결과: zookeeper-x.x.x.tgz 파일 생성됨

## 3. ChartMuseum으로 업로드
```
$ curl --data-binary "@zookeeper-13.8.7.tgz" http://192.168.10.40:8089/api/charts
$ helm repo update
$ helm search repo my-private-repo/zookeeper
```

## 4. pv 생성
```
$ mkdir -p /nfs/zookeeper/{pv1,pv2,pv3}
$ chmod 777 -R /nfs/zookeeper
$ vi /etc/exports
/nfs/zookeeper/pv1 *(no_root_squash,rw)
/nfs/zookeeper/pv2 *(no_root_squash,rw)
/nfs/zookeeper/pv3 *(no_root_squash,rw)
$ exportfs -r
$ showmount -e
```
```
$ vi idms-zookeeper.yaml
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: bitnami-zookeeper-mirror
spec:
  imageDigestMirrors:
    - source: docker.io/bitnamilegacy/zookeeper
      mirrors:
        - bastion-nexus.kscada.kdneri.com:5002/zookeeper/zookeeper

$ vi itms-zookeeper.yaml
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: bitnami-zookeeper-tag-mirror
spec:
  imageTagMirrors:
    - source: docker.io/bitnamilegacy/zookeeper
      mirrors:
        - bastion-nexus.kscada.kdneri.com:5002/zookeeper/zookeeper

$ oc apply -f idms-zookeeper.yaml
$ oc apply -f itms-zookeeper.yaml
```



## 6. helm 차트로 배포
```
$ oc new-project zookeeper
$ helm install my-zookeeper my-private-repo/zookeeper
```
