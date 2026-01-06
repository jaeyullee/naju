## 1. GridGrain 레포 추가
```
$ helm repo add gridgain https://gridgain.github.io/helm-charts/
$ helm repo update
```

## 2. 차트 다운로드 및 압축 해제 (최신 버전 기준)
> --untar 옵션을 쓰면 압축이 풀린 디렉토리 상태로 다운로드됩니다.
```
$ helm pull gridgain/gridgain --untar
```

## 3. values.yaml, Chart.yaml 수정 후 차트 패키징
```
$ vi gridgain/Chart.yaml
apiVersion: v2
appVersion: 8.9.11
description: GridGain is a Unified Real-Time Data Platform by the original creators
  of Apache Ignite. It enables a simplified and optimized data architecture for enterprises
  that require extreme speed, massive scale, and high availability from their data
  ecosystem.
home: https://www.gridgain.com/
maintainers:
- name: GridGain Systems, Inc. All Rights Reserved
  url: https://www.gridgain.com/
name: ignite      ## gridgain -> ignite 변경
sources:
- https://github.com/gridgain/helm-charts/tree/main/charts/gridgain
version: 1.0.4
```
```
$ vi gridgain/values.yaml
image:
  registry: docker.io
  repository: apacheignite/ignite    ## 수정. 기존 이미지 쓰면 라이선스 문제 있음
  tag: 2.17.0                  ## 수정. 3.x버전은 헬름차트 호환x
  pullPolicy: IfNotPresent

livenessProbe:
  enabled: true
  httpGet:
    path: /ignite?cmd=version
    port: 8080
    initialDelaySeconds: 30     ## 5초 -> 30초 이상으로 변경
  periodSeconds: 10
  failureThreshold: 3

# -- GridGain [modules](https://www.gridgain.com/docs/latest/developers-guide/setup#enabling-modules) to be enabled
optionLibs: ignite-kubernetes,ignite-rest-http,ignite-log4j2,ignite-schedule    ## control-center-agent 제거

# -- Number of GridGain cluster replicas
replicaCount: 3                ## 1 -> 3으로 수정

serviceAccount:
  create: false
  name: "my-ignite-sa"

# ... 나머지 설정 유지 혹은 수정

$ helm package gridgain/
```
> 결과: ignite-x.x.x.tgz 파일 생성됨

## 4. ChartMuseum으로 업로드 (curl 사용)
> <chart-museum-server-ip> 부분을 실제 IP로 변경하세요.
```
$ curl --data-binary "@ignite-1.0.4.tgz" http://192.168.10.40:8089/api/charts
$ helm repo update
$ helm search repo my-private-repo/
```

## 5. PV 생성 및 ITMS/IDMS 생성
```
$ mkdir -p /nfs/ignite/{pv1,pv2,pv3}
$ chmod 777 -R /nfs/ignite
$ vi /etc/exports
/nfs/ignite/pv1 *(no_root_squash,rw)
/nfs/ignite/pv2 *(no_root_squash,rw)
/nfs/ignite/pv3 *(no_root_squash,rw)
$ exportfs -r
$ showmount -e
```
```
$ vi pv-ignite-1
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-ignite-1
spec:
  capacity:
    storage: 8Gi
  nfs:
    server: 192.168.10.40
    path: /nfs/ignite/pv1
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  volumeMode: Filesystem

$ vi pv-ignite-2
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-ignite-2
spec:
  capacity:
    storage: 8Gi
  nfs:
    server: 192.168.10.40
    path: /nfs/ignite/pv2
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  volumeMode: Filesystem

$ vi pv-ignite-3
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-ignite-3
spec:
  capacity:
    storage: 8Gi
  nfs:
    server: 192.168.10.40
    path: /nfs/ignite/pv3
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  volumeMode: Filesystem

$ oc apply -f pv-ignite-1
$ oc apply -f pv-ignite-2
$ oc apply -f pv-ignite-3
```

## 6. 배포
```
$ oc new-project ignite-cluster
$ oc adm policy add-scc-to-user anyuid -z default -n ignite-cluster

$ helm install my-ignite my-privage-repo/gridgain -n ignite-cluster
$ oc get pods -n ignite-cluster
```
