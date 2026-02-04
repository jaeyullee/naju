# 1. Template 배포 (PV Dynamic Provisioning)
> NFS CSI Driver 설치되어있다고 가정  
```
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  annotations:
    description: Apache Ignite 3.x Cluster (Requires Pre-created PVs)
    iconClass: icon-redis
    openshift.io/display-name: Apache Ignite Cluster
    openshift.io/provider-display-name: KSCADA Team
    tags: database,ignite,middleware
  name: ignite-cluster-template
  namespace: ignite-test
objects:
- apiVersion: v1
  data:
    ignite-config.conf: |
      ignite {
        network {
          port: 3344
          listenAddresses: ["0.0.0.0"]
          nodeFinder {
            netClusterNodes: [
              "${APP_NAME}-0.${SERVICE_NAME}:3344",
              "${APP_NAME}-1.${SERVICE_NAME}:3344",
              "${APP_NAME}-2.${SERVICE_NAME}:3344"
            ]
          }
        }
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
  kind: ConfigMap
  metadata:
    name: ignite-config
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: ${APP_NAME}
    name: ${SERVICE_NAME}
  spec:
    clusterIP: None
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
    selector:
      app: ${APP_NAME}
- apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: ${APP_NAME}
  spec:
    persistentVolumeClaimRetentionPolicy:
      whenDeleted: Retain
      whenScaled: Retain
    podManagementPolicy: OrderedReady
    replicas: 3
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: ${APP_NAME}
    serviceName: ${SERVICE_NAME}
    template:
      metadata:
        labels:
          app: ${APP_NAME}
      spec:
        containers:
        - args:
          - --node-name
          - $(POD_NAME)
          - --config-path
          - /opt/ignite/custom/ignite-config.conf
          env:
          - name: IGNITE3_EXTRA_JVM_ARGS
            value: -Djava.net.preferIPv4Stack=true
          - name: POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: JVM_MIN_MEM
            value: ${JVM_HEAP}
          - name: JVM_MAX_MEM
            value: ${JVM_HEAP}
          - name: IGNITE_WORK_DIR
            value: /opt/ignite/work
          - name: JAVA_OPTS
            value: |
              -XX:+UseG1GC -Xms${JVM_HEAP} -Xmx${JVM_HEAP} -XX:MaxGCPauseMillis=100 -XX:InitiatingHeapOccupancyPercent=45 -XX:G1HeapRegionSize=16M -XX:+ParallelRefProcEnabled -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=30 -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:MaxDirectMemorySize=4G -Xlog:gc*:file=/opt/ignite/work/gc.log:time,level -Dignite.system.criticalWorkers.maxAllowedLagMillis=20000 -Dignite.network.connectTimeout=30000 -Dignite.network.idleTimeout=60000
          image: ${IGNITE_IMAGE}
          imagePullPolicy: IfNotPresent
          name: ignite
          ports:
          - containerPort: 3344
            name: cluster
            protocol: TCP
          - containerPort: 10300
            name: rest
            protocol: TCP
          - containerPort: 10800
            name: client
            protocol: TCP
          resources:
            limits:
              cpu: ${CPU_LIMIT}
              memory: ${MEMORY_LIMIT}
            requests:
              cpu: ${CPU_REQUEST}
              memory: ${MEMORY_REQUEST}
          volumeMounts:
          - mountPath: /opt/ignite/custom
            name: ignite-custom-vol
          - mountPath: /opt/ignite/work
            name: data
        initContainers:
        - command:
          - sh
          - -c
          - mkdir -p /opt/ignite/custom && cp /tmp/config-map/* /opt/ignite/custom/
          image: ${IGNITE_IMAGE}
          name: init-config
          volumeMounts:
          - mountPath: /tmp/config-map
            name: ignite-config-source
          - mountPath: /opt/ignite/custom
            name: ignite-custom-vol
        terminationGracePeriodSeconds: 60
        volumes:
        - configMap:
            name: ignite-config
          name: ignite-config-source
        - emptyDir: {}
          name: ignite-custom-vol
    updateStrategy:
      rollingUpdate:
        partition: 0
      type: RollingUpdate
    volumeClaimTemplates:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: ${VOLUME_CAPACITY}
        storageClassName: "nfs-csi"
        volumeMode: Filesystem
parameters:
- description: Must match 'cluster-name' label on pre-created PVs
  displayName: Application Name
  name: APP_NAME
  required: true
  value: ignite-node
- displayName: Headless Service Name
  name: SERVICE_NAME
  required: true
  value: ignite-headless
- description: Should match the capacity of pre-created PVs
  displayName: Volume Capacity
  name: VOLUME_CAPACITY
  required: true
  value: 10Gi
- displayName: Ignite Image
  name: IGNITE_IMAGE
  required: true
  value: nexus.kscada.kdneri.com:5002/app/ignite:3.1.0
- displayName: CPU Request
  name: CPU_REQUEST
  value: "2"
- displayName: CPU Limit
  name: CPU_LIMIT
  value: "2"
- displayName: Memory Request
  name: MEMORY_REQUEST
  value: 4Gi
- displayName: Memory Limit
  name: MEMORY_LIMIT
  value: 4Gi
- displayName: JVM Heap Size
  name: JVM_HEAP
  required: true
  value: 2g
```

# 2. Template 배포 (PV Manual Provisioning)
> ignite-pvs.yaml 에 있는 .spec.claimRef.namespace 를 적절히 수정 필요  
```
$ cat ignite-pvs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ignite-node-pv-1
  labels:
    usage: ignite-data-vol
    cluster-name: kscada-ignite
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    name: data-ignite-node-0
    namespace: <namespace>
  nfs:
    path: /nfs/ignite/pv1
    server: xx.xx.xx.26
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ignite-node-pv-2
  labels:
    usage: ignite-data-vol
    cluster-name: kscada-ignite
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    name: data-ignite-node-1
    namespace: ignite-test
  nfs:
    path: /nfs/ignite/pv2
    server: xx.xx.xx.26
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ignite-node-pv-3
  labels:
    usage: ignite-data-vol
    cluster-name: kscada-ignite
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    name: data-ignite-node-2
    namespace: ignite-test
  nfs:
    path: /nfs/ignite/pv3
    server: xx.xx.xx.26
```
```
cat ignite-template.yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: ignite-cluster-template
  annotations:
    description: "Apache Ignite 3.x Cluster (Requires Pre-created PVs)"
    iconClass: "icon-redis"
    tags: "database,ignite,middleware"
    openshift.io/display-name: "Apache Ignite Cluster"
    openshift.io/provider-display-name: "KSCADA Team"
objects:
# ==========================================================
# 1. ConfigMap
# ==========================================================
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: ignite-config
  data:
    ignite-config.conf: |
      ignite {
        network {
          port: 3344
          listenAddresses: ["0.0.0.0"]
          nodeFinder {
            netClusterNodes: [
              "${APP_NAME}-0.${SERVICE_NAME}:3344",
              "${APP_NAME}-1.${SERVICE_NAME}:3344",
              "${APP_NAME}-2.${SERVICE_NAME}:3344"
            ]
          }
        }
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

# ==========================================================
# 2. Service
# ==========================================================
- apiVersion: v1
  kind: Service
  metadata:
    name: ${SERVICE_NAME}
    labels:
      app: ${APP_NAME}
  spec:
    clusterIP: None
    selector:
      app: ${APP_NAME}
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

# ==========================================================
# 3. StatefulSet
# ==========================================================
- kind: StatefulSet
  apiVersion: apps/v1
  metadata:
    name: ${APP_NAME}
  spec:
    serviceName: ${SERVICE_NAME}
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
        app: ${APP_NAME}
    template:
      metadata:
        labels:
          app: ${APP_NAME}
      spec:
        terminationGracePeriodSeconds: 60
        initContainers:
          - name: init-config
            image: ${IGNITE_IMAGE}
            command: ['sh', '-c', 'mkdir -p /opt/ignite/custom && cp /tmp/config-map/* /opt/ignite/custom/']
            volumeMounts:
              - name: ignite-config-source
                mountPath: /tmp/config-map
              - name: ignite-custom-vol
                mountPath: /opt/ignite/custom
        containers:
          - name: ignite
            image: ${IGNITE_IMAGE}
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
                cpu: ${CPU_LIMIT}
                memory: ${MEMORY_LIMIT}
              requests:
                cpu: ${CPU_REQUEST}
                memory: ${MEMORY_REQUEST}
            env:
              - name: IGNITE3_EXTRA_JVM_ARGS
                value: "-Djava.net.preferIPv4Stack=true"
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.name
              - name: JVM_MIN_MEM
                value: ${JVM_HEAP}
              - name: JVM_MAX_MEM
                value: ${JVM_HEAP}
              - name: IGNITE_WORK_DIR
                value: /opt/ignite/work
              - name: JAVA_OPTS
                value: |
                  -XX:+UseG1GC -Xms${JVM_HEAP} -Xmx${JVM_HEAP} -XX:MaxGCPauseMillis=100 -XX:InitiatingHeapOccupancyPercent=45 -XX:G1HeapRegionSize=16M -XX:+ParallelRefProcEnabled -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=30 -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:MaxDirectMemorySize=4G -Xlog:gc*:file=/opt/ignite/work/gc.log:time,level -Dignite.system.criticalWorkers.maxAllowedLagMillis=20000 -Dignite.network.connectTimeout=30000 -Dignite.network.idleTimeout=60000
            volumeMounts:
              - name: ignite-custom-vol
                mountPath: /opt/ignite/custom
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
              storage: ${VOLUME_CAPACITY}
          storageClassName: ''
          volumeMode: Filesystem
          selector:
            matchLabels:
              usage: ignite-data-vol
              cluster-name: ${APP_NAME}

# ==========================================================
# 4. Parameters
# ==========================================================
parameters:
  - name: APP_NAME
    displayName: "Application Name"
    description: "Must match 'cluster-name' label on pre-created PVs"
    value: "ignite-node"
    required: true

  - name: SERVICE_NAME
    displayName: "Headless Service Name"
    value: "ignite-headless"
    required: true

  - name: VOLUME_CAPACITY
    displayName: "Volume Capacity"
    description: "Should match the capacity of pre-created PVs"
    value: "100Gi"
    required: true

  - name: IGNITE_IMAGE
    displayName: "Ignite Image"
    value: "quay.xxx.xxx.xxx:8443/app/ignite:3.1.0"
    required: true

  - name: CPU_REQUEST
    displayName: "CPU Request"
    value: "8"

  - name: CPU_LIMIT
    displayName: "CPU Limit"
    value: "8"

  - name: MEMORY_REQUEST
    displayName: "Memory Request"
    value: "32Gi"

  - name: MEMORY_LIMIT
    displayName: "Memory Limit"
    value: "32Gi"

  - name: JVM_HEAP
    displayName: "JVM Heap Size"
    value: "10g"
    required: true
```
```
$ oc new-project <namespace>
$ oc create -f ignite-template.yaml
$ oc create -f ignite-pvs.yaml
```
> OCP 웹콘솔 GUI에서 템플릿으로 ignite 배포
```
$ oc exec -it ignite-node-0 -- ignite3 cluster init  --url=http://127.0.0.1:10300   --name=kscada-ignite   --metastorage-group=ignite-node-0,ignite-node-1,ignite-node-2
```
