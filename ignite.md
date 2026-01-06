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

# -- GridGain [modules](https://www.gridgain.com/docs/latest/developers-guide/setup#enabling-modules) to be enabled
optionLibs: ignite-kubernetes,ignite-rest-http,ignite-log4j2,ignite-schedule,ignite-json    ## control-center-agent 제거, ignite-json 추가

# -- Number of GridGain cluster replicas
replicaCount: 3                ## 1 -> 3으로 수정

configMaps:                    ## {}에서 아래 내용으로 수정
  default-config:
    name: default-config.xml
    path: /opt/ignite/apache-ignite/config/default-config.xml
    content: |
      <?xml version="1.0" encoding="UTF-8"?>
      <beans xmlns="http://www.springframework.org/schema/beans"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://www.springframework.org/schema/beans
                                 http://www.springframework.org/schema/beans/spring-beans.xsd">

          <bean class="org.apache.ignite.configuration.IgniteConfiguration">
              <property name="discoverySpi">
                  <bean class="org.apache.ignite.spi.discovery.tcp.TcpDiscoverySpi">
                      <property name="ipFinder">
                          <bean class="org.apache.ignite.spi.discovery.tcp.ipfinder.kubernetes.TcpDiscoveryKubernetesIpFinder">
                              <property name="namespace" value="{{ .Release.Namespace }}"/>
                              <property name="serviceName" value="{{ include "gridgain.fullname" . }}-headless"/>
                          </bean>
                      </property>
                  </bean>
              </property>

              <property name="dataStorageConfiguration">
                  <bean class="org.apache.ignite.configuration.DataStorageConfiguration">
                      <property name="defaultDataRegionConfiguration">
                          <bean class="org.apache.ignite.configuration.DataRegionConfiguration">
                              <property name="persistenceEnabled" value="true"/>
                          </bean>
                      </property>
                      </bean>
              </property>
          </bean>
      </beans>

# configMapsFromFile:    ## 이 단락 전체 주석처리
#   default-config:
#     filename: default-config.xml
#     path: /opt/gridgain/config/default-config.xml

livenessProbe:
  enabled: true
  httpGet:
    path: /ignite?cmd=version
    port: 8080
    initialDelaySeconds: 30     ## 5초 -> 30초 이상으로 변경
  periodSeconds: 10
  failureThreshold: 3

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
  claimRef:
    namespace: ignite-cluster
    name: persistence-my-ignite-0
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
  claimRef:
    namespace: ignite-cluster
    name: persistence-my-ignite-1
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
  claimRef:
    namespace: ignite-cluster
    name: persistence-my-ignite-2
  volumeMode: Filesystem

$ oc apply -f pv-ignite-1
$ oc apply -f pv-ignite-2
$ oc apply -f pv-ignite-3
```

## 6. 배포
```
$ oc new-project ignite-cluster
$ oc create sa my-ignite-sa -n ignite-cluster
$ oc adm policy add-scc-to-user anyuid -z my-ignite-sa -n ignite-cluster

$ helm install my-ignite my-privage-repo/gridgain -n ignite-cluster
$ oc get pods -n ignite-cluster
```
> 혹시 ssc 문제가 발생한다면 아래 명령어 수행
```
$ oc set serviceaccount statefulset/my-ignite my-ignite-sa -n ignite-cluster
$ oc get pods -n ignite-cluster
```

## 7. 클러스터 상태 점검
```
$ oc logs my-ignite-0 -n ignite-cluster | grep "Topology snapshot"
```
> ... Topology snapshot [ver=3, locNode=..., servers=3, clients=0, state=ACTIVE, ...]        ## servers=3 이면 정상

```
$ oc exec -it my-ignite-0 -n ignite-cluster -- /opt/ignite/apache-ignite/bin/control.sh --baseline
```
> Cluster state: INACTIVE        ## INACTIVE 라면 수동으로 ACTIVATE를 해줘야함
> Current topology version: 3
```
$ oc exec -it my-ignite-0 -n ignite-cluster -- /opt/ignite/apache-ignite/bin/control.sh --activate
```
> Cluster state: ACTIVE          ## ACTIVE 됐으면 정상

```
$ oc exec -it my-ignite-0 -n ignite-cluster -- /opt/ignite/apache-ignite/bin/sqlline.sh -u jdbc:ignite:thin://127.0.0.1/
Enter username for jdbc:ignite:thin://127.0.0.1/:
Enter password for jdbc:ignite:thin://127.0.0.1/:
0: jdbc:ignite:thin://127.0.0.1/> CREATE TABLE TEST_DATA (ID INT PRIMARY KEY, NAME VARCHAR);
0: jdbc:ignite:thin://127.0.0.1/> INSERT INTO TEST_DATA (ID, NAME) VALUES (1, 'Ignite Works!');
0: jdbc:ignite:thin://127.0.0.1/> INSERT INTO TEST_DATA (ID, NAME) VALUES (2, 'Cluster Check');
0: jdbc:ignite:thin://127.0.0.1/> !quit
```
> 파드가 보안상 루트(Root) 권한이 없어서 파일 쓰기를 거절당해서 java 에러가 발생하지만 sql 자체는 정상수행 됩니다.
```
$ oc exec -it my-ignite-2 -n ignite-cluster -- /opt/ignite/apache-ignite/bin/sqlline.sh -u jdbc:ignite:thin://127.0.0.1/
Enter username for jdbc:ignite:thin://127.0.0.1/:
Enter password for jdbc:ignite:thin://127.0.0.1/:
0: jdbc:ignite:thin://127.0.0.1/> SELECT * FROM TEST_DATA;
```
> 테이블 조회되면 정상상
