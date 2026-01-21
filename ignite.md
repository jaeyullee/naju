> 고객이 AS-IS 에서 테스트 중인 내용 기반으로 수정하여 작성하였습니다.

# 1. Ignite 이미지 재빌드
> OCP에서는 파드 배포시 해당 네임스페이스에 할당된 범위의 임의의 사용자 ID (ex. 1000580000)를 할당하기 때문에 파일시스템 접근권한 에러가 발생합니다.  <br/>
> 이를 해결하기 위해 root(0) 그룹 ID 권한을 부여하는 방안을 권장합니다. [[참고]](https://docs.redhat.com/ko/documentation/openshift_container_platform/4.20/html/images/creating-images#use-uid_create-images)
```
$ vi Dockerfile
FROM apacheignite/ignite:3.1.0
USER root
RUN useradd -u 1000 -m -s /bin/bash ignite && \
    chgrp -R 0 /opt/ignite && \
    chmod -R g=u /opt/ignite && \
    chown -R ignite:0 /opt/ignite
USER ignite
```
```
$ podman build -t my-ignite:3.1.0 .
$ podman run --rm -it -u 12223 my-ignite:3.1.0
$ podman tag my-ignite:3.1.0 quay.xxx.xxx.xxx/app/ignite:3.1.0
$ podman push quay.xxx.xxx.xxx/app/ignite:3.1.0
```

# 2. NFS 준비
> 777 권한으로 테스트 진행합니다.
```
$ mkdir -p /data/nfs-manual/ignite/{pv1,pv2,pv3}
$ chmod 777 -R /data/nfs-manual/ignite
```
```
$ vi /etc/exports
...
/data/nfs-manual/ignite/pv1 192.168.10.0/24(rw,sync,no_root_squash)
/data/nfs-manual/ignite/pv2 192.168.10.0/24(rw,sync,no_root_squash)
/data/nfs-manual/ignite/pv3 192.168.10.0/24(rw,sync,no_root_squash)
```
```
$ exportfs -arv
$ showmount -e
```

# 3. Ignite 배포
```
$ vi ignite-pvs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ignite-pv-1
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    namespace: kscada-main-mw-ignite
    name: data-ignite-node-0
  nfs:
    path: /data/nfs-manual/ignite/pv1
    server: xx.xx.xx.26
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ignite-pv-2
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    namespace: kscada-main-mw-ignite
    name: data-ignite-node-1
  nfs:
    path: /data/nfs-manual/ignite/pv2
    server: xx.xx.xx.26
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ignite-pv-3
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    namespace: kscada-main-mw-ignite
    name: data-ignite-node-2
  nfs:
    path: /data/nfs-manual/ignite/pv3
    server: xx.xx.xx.26
```
```
$ vi ignite-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ignite-config
  namespace: kscada-main-mw-ignite
data:
  ignite-config.conf: |
    ignite {
      network {
        port: 3344
        listenAddresses: ["0.0.0.0"]
        nodeFinder {
          # 3개 노드의 주소를 모두 적어줍니다.
          netClusterNodes: [
            "ignite-node-0.ignite-headless:3344",
            "ignite-node-1.ignite-headless:3344",
            "ignite-node-2.ignite-headless:3344"
          ]
        }
      }
      # (기존 스토리지 설정 유지)
      storage {
        profiles = [
          { name = "aimem", engine = "aimem" },
          { name = "aipersist", engine = "aipersist" },
          { name = "rocksdb", engine = "rocksdb" },
          { name = "default", engine = "rocksdb" }
        ]
      }
      rest { port: 10300 }
      clientConnector { port: 10800 }
    }
```
```
$ vi ignite-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: ignite-headless
  namespace: kscada-mw-ignite
  labels:
    app: ignite-node
spec:
  clusterIP: None
  selector:
    app: ignite-node
  ports:
    - name: cluster
      port: 3344
      targetPort: 3344
    - name: rest
      port: 10300
      targetPort: 10300
    - name: client
      port: 10800
      targetPort: 10800

```
```
$ vi ignite-sts.yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: ignite-node
  namespace: kscada-main-mw-ignite
spec:
  serviceName: ignite-headless
  replicas: 3
  revisionHistoryLimit: 10
  podManagementPolicy: OrderedReady
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  selector:
    matchLabels:
      app: ignite-node
  template:
    metadata:
      labels:
        app: ignite-node
    spec:
      terminationGracePeriodSeconds: 60
      initContainers:
        - name: init-config
          image: 'nexus.kscada.kdneri.com:5002/app/ignite:3.1.3'
          command: ['sh', '-c', 'mkdir -p /opt/ignite/custom && cp /tmp/config-map/* /opt/ignite/custom/']
          volumeMounts:
            - name: ignite-config-source
              mountPath: /tmp/config-map
            - name: ignite-custom-vol
              mountPath: /opt/ignite/custom
      containers:
        - name: ignite
          image: 'quay.xxx.xxx.xxx:5000/app/ignite:3.1.0'
          imagePullPolicy: IfNotPresent
          args:
            - '--node-name'
            - $(POD_NAME)
            - "--config-path"
            - "/opt/ignite/custom/ignite-config.conf"
          ports:
            - name: cluster
              containerPort: 3344
              protocol: TCP
            - name: rest
              containerPort: 10300
              protocol: TCP
            - name: client
              containerPort: 10800
              protocol: TCP
          resources:
            limits:
              cpu: '8'
              memory: 32Gi
            requests:
              cpu: '8'
              memory: 32Gi
          env:
            - name: IGNITE3_EXTRA_JVM_ARGS
              value: >-
                -Djava.net.preferIPv4Stack=true
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: JVM_MIN_MEM
              value: 10g
            - name: JVM_MAX_MEM
              value: 10g
            - name: IGNITE_WORK_DIR
              value: /opt/ignite/work
            - name: JAVA_OPTS
              value: |
                -XX:+UseG1GC -Xms10g -Xmx10g -XX:MaxGCPauseMillis=100 -XX:InitiatingHeapOccupancyPercent=45 -XX:G1HeapRegionSize=16M -XX:+ParallelRefProcEnabled -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=30 -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:MaxDirectMemorySize=4G -Xlog:gc*:file=/opt/ignite/work/gc.log:time,level -Dignite.system.criticalWorkers.maxAllowedLagMillis=20000 -Dignite.network.connectTimeout=30000 -Dignite.network.idleTimeout=60000
          volumeMounts:
            - name: ignite-custom-vol
              mountPath: /opt/ignite/custom
              subPath: ignite-config.conf
            - name: data
              mountPath: /opt/ignite/work
      volumes:
        - name: ignite-config-source
          configMap:
            name: ignite-config
        - name: ignite-custom-vol
          emptyDir: {}
  volumeClaimTemplates:
    - kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: ''
        volumeMode: Filesystem
```
> 컨테이너를 32GB나 잡았는데 JVM(10GB) + Direct(4GB) = 총 14GB 정도만 명시적으로 쓰고 있습니다. <br/>
> 다음과 같은 튜닝을 추천합니다. <br/>

> 튜닝방안1) JVM 10g, Direct 4g, Stack 512k <br/>
> - 컨테이너 리소스 24GB, JVM_MIN_MEM: 10g, JVM_MAX_MEM: 10g, JAVA_OPTS: -XX:+UseG1GC -Xms10g -Xmx10g -Xss512k -XX:MaxGCPauseMillis=100 -XX:InitiatingHeapOccupancyPercent=45 -XX:G1HeapRegionSize=16M -XX:+ParallelRefProcEnabled -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=30 -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:MaxDirectMemorySize=4G -Xlog:gc*:file=/opt/ignite/work/gc.log:time,level -Dignite.system.criticalWorkers.maxAllowedLagMillis=20000 -Dignite.network.connectTimeout=30000 -Dignite.network.idleTimeout=60000

> 튜닝방안2) JVM 16g, Direct 8g, Stack 512k <br/>
> - 컨테이너 리소스 32GB, JVM_MIN_MEM: 16g, JVM_MAX_MEM: 16g, JAVA_OPTS: -XX:+UseG1GC -Xms16g -Xmx16g -Xss512k -XX:MaxGCPauseMillis=100 -XX:InitiatingHeapOccupancyPercent=45 -XX:G1HeapRegionSize=16M -XX:+ParallelRefProcEnabled -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=30 -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:MaxDirectMemorySize=8G -Xlog:gc*:file=/opt/ignite/work/gc.log:time,level -Dignite.system.criticalWorkers.maxAllowedLagMillis=20000 -Dignite.network.connectTimeout=30000 -Dignite.network.idleTimeout=60000
```
$ oc new-project kscada-main-mw-ignite
$ oc create -f ignite-pvs.yaml
$ oc create -f ignite-cm.yaml
$ oc create -f ignite-headless-svc.yaml
$ oc create -f ignite-sts.yaml
```

# 4. Ignite 클러스터 초기화
```
$ oc exec -it ignite-node-0 -n kscada-main-mw-ignite -- ignite3 cluster init   --url=http://127.0.0.1:10300   --name=kscada-cluster   --metastorage-group=ignite-node-0,ignite-node-1,ignite-node-2
```
> 버전에 따라 다를 수 있으므로, ignite3 cluster init --help 를 이용해 flags를 미리 확인해야 합니다.
> 안될 경우 포트 살아있는지 확인하는 방법
```
$ oc exec -it ignite-node-0 -n ignite-test -- grep "283C" /proc/net/tcp
$ oc exec ignite-node-0 -n ignite-test -- perl -e 'use IO::Socket::INET; $s = IO::Socket::INET->new(PeerAddr => "127.0.0.1", PeerPort => 10300, Proto => "tcp", Timeout => 2); if($s) { print "SUCCESS: Port 10300 is responding\n"; } else { print "FAILED: $!\n"; }'
```

# 5. Ignite 클러스터 상태 점검
```
$ oc exec -it ignite-node-0 -n kscada-mw-ignite -- \
  /opt/ignite/apache-ignite/bin/ignite3 cluster status
```
> Cluster status: <br/>
>   Name: my-cluster        <-- 아까 init 할 때 지은 이름 <br/>
>   State: ACTIVE           <-- [중요] ACTIVE 상태여야 함 <br/>
>   Health: HEALTHY         <-- HEALTHY 여야 함
```
$ oc exec -it ignite-node-0 -n kscada-mw-ignite -- \
  /opt/ignite/apache-ignite/bin/ignite3 cluster topology logical
```
> Logical topology: <br/>
>   ignite-node-0 (id: ... , address: 10.128.x.x:3344) <br/>
>   ignite-node-1 (id: ... , address: 10.128.x.x:3344) <br/>
>   ignite-node-2 (id: ... , address: 10.128.x.x:3344) <br/>
>    <br/>
> Total: 3 nodes   <-- [중요] 반드시 3개가 보여야 성공!
```
$ oc exec -it ignite-node-0 -n kscada-mw-ignite -- \
  curl -s http://localhost:10300/management/v1/cluster/state
```
> { <br/>
>   "clusterTag": { ... }, <br/>
>   "igniteVersion": "3.1.0", <br/>
>   "name": "my-cluster", <br/>
>   "state": "ACTIVE"      <-- 여기가 핵심 <br/>
> }
```
$ oc logs ignite-node-0 -n kscada-mw-ignite | grep "Topology"
```
> ... Topology snapshot: [nodes=3] ...
