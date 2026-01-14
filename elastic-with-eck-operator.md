> 해당 문서는 Cluster Logging 오퍼레이터와 ECK 오퍼레이터가 이미 미러링되어 준비된 환경을 기준으로 작성합니다.

# 1. Cluster Logging 오퍼레이터 및 ECK 오퍼레이터 설치
> OpenShift 콘솔의 GUI 화면을 이용하여 Operator 설치

# 2. elasticsearch 컨테이너 이미지 반입
> ECK 오퍼레이터 버전에 따라 elasticsearch 이미지 버전 결정
> nexus의 도커 레지스트리에 elasticsearch 이미지를 푸시할 경로는 ocp-operators-mirror/eck로 가정.
```
$ podman pull docker.elastic.io/elasticsearch/elasticsearch:9.2.0
$ podman tag docker.elastic.io/elasticsearch/elasticsearch:9.2.0 ocp-registry.xxx.xxx.xxx:5000/ocp-operators-mirror/elasticsearch/elasticsearch:9.2.0
$ podman push ocp-registry.xxx.xxx.xxx:5000/ocp-operators-mirror/elasticsearch/elasticsearch:9.2.0
$ curl -u <id>:<pw> https://ocp-registry.xxx.xxx.xxx:5000/v2/_catalog
```

# 3. elasticsearch 컨테이너 이미지 활용을 위한 미러링
```
$ vi itms-elastic.yaml
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: elastic-mirror
spec:
  imageTagMirrors:
  - mirrors:
    - ocp-registry.xxx.xxx.xxx:5000/oss/elasticsearch
    source: docker.elastic.co/elasticsearch
```
```
$ oc apply -f itms-elastic.yaml
$ watch oc get node,mcp
```

# 4. elasticsearch cluster 배포
```
$ vi elastic-pvs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ocp-es-1
  namespace: ocp-es
  labels:
    index: "ocp-es-1"
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  nfs:
    path: /data/nfs-manual/eck/pv1
    server: xx.xx.xx.26
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ocp-es-2
  labels:
    index: "ocp-es-2"
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  nfs:
    path: /data/nfs-manual/eck/pv2
    server: xx.xx.xx.26
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ocp-es-3
  labels:
    index: "ocp-es-3"
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  nfs:
    path: /data/nfs-manual/eck/pv3
    server: xx.xx.xx.26
```
```
$ vi elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: ocp
  namespace: ocp-es
spec:
  nodeSets:
  - config:
      node.attr.attr_name: node-1
      node.roles:
      - master
      - data
      - ingest
      node.store.allow_mmap: false
    count: 1
    name: node-1
    podTemplate:
      metadata:
        labels:
          es: ocp-es
      spec:
        nodeSelector:
          node-role.kubernetes.io/logging: ""
        tolerations:
        - key: "role"
          operator: "Equal"
          value: "logging"
          effect: "NoSchedule"
        containers:
        - env:
          - name: ES_JAVA_OPTS
            value: -Xms4g -Xmx4g
          name: elasticsearch
          resources:
            limits:
              cpu: "4"
              memory: 8Gi
            requests:
              cpu: "4"
              memory: 8Gi
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: ""
        selector:
          matchLabels:
            index: "ocp-es-1"
  - config:
      node.attr.attr_name: node-2
      node.roles:
      - master
      - data
      - ingest
      node.store.allow_mmap: false
    count: 1
    name: node-2
    podTemplate:
      metadata:
        labels:
          es: ocp-es
      spec:
        nodeSelector:
          node-role.kubernetes.io/logging: ""
        tolerations:
        - key: "role"
          operator: "Equal"
          value: "logging"
          effect: "NoSchedule"
        containers:
        - env:
          - name: ES_JAVA_OPTS
            value: -Xms4g -Xmx4g
          name: elasticsearch
          resources:
            limits:
              cpu: "4"
              memory: 8Gi
            requests:
              cpu: "4"
              memory: 8Gi
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: ""
        selector:
          matchLabels:
            index: "ocp-es-2"
  - config:
      node.attr.attr_name: node-3
      node.roles:
      - master
      - data
      - ingest
      node.store.allow_mmap: false
    count: 1
    name: node-3
    podTemplate:
      metadata:
        labels:
          es: ocp-es
      spec:
        nodeSelector:
          node-role.kubernetes.io/logging: ""
        tolerations:
        - key: "role"
          operator: "Equal"
          value: "logging"
          effect: "NoSchedule"
        containers:
        - env:
          - name: ES_JAVA_OPTS
            value: -Xms4g -Xmx4g
          name: elasticsearch
          resources:
            limits:
              cpu: "4"
              memory: 8Gi
            requests:
              cpu: "4"
              memory: 8Gi
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: ""
        selector:
          matchLabels:
            index: "ocp-es-3"
  version: 9.2.0
```
```
$ oc new-project ocp-es
$ oc apply -f elastic-pvs.yaml
$ oc apply -f elasticsearch.yaml
```

# 5. Elasticsearch Cluster 상태 확인
```
$ oc get elasticsearch -n ocp-es
$ PASSWORD=$(oc get secret ocp-es-elastic-user -n ocp-es -o go-template='{{.data.elastic | base64decode}}')
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:$PASSWORD" -k "https://localhost:9200/_cluster/health?pretty"
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:$PASSWORD" -k "https://localhost:9200/_cat/nodes?v"
$ oc logs -f ocp-es-node-1-0 -n ocp-es
```

# 6. clusterLoggingForwarder 배포
```
$ oc create sa logging-collector -n openshift-logging
$ oc adm policy add-cluster-role-to-user logging-collector-logs-writer -z logging-collector -n openshift-logging
$ oc adm policy add-cluster-role-to-user collect-application-logs -z logging-collector -n openshift-logging
$ oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z logging-collector -n openshift-logging
$ oc adm policy add-cluster-role-to-user collect-audit-logs -z logging-collector -n openshift-logging
```
```
$ ELASTIC_PASSWORD=$(oc get secret ocp-es-elastic-user -n ocp-es -o jsonpath='{.data.elastic}' |base64 -d)
$ oc create secret generic ocp-es-secret --from-literal=username=elastic --from-literal=password=$ELASTIC_PASSWORD -n openshift-logging
```
> 아래 clusterlogforwarder는 infrastructure 로그만 전송하게 설정했음.
> application 으로 분류되지만 인프라 성격의 operator 로그들 전송 및 audit 로그 전송 설정 필요함.
```
$ vi clusterlogforwarder.yaml
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: infra-logforwarder-instance
  namespace: openshift-logging
spec:
  collector:
    tolerations:
    - operator: Exists
  serviceAccount:
    name: logging-collector
  outputs:
  - name: eck-elasticsearch
    type: elasticsearch
    elasticsearch:
      url: https://ocp-es-http.ocp-es.svc:9200
      index: "infra-logs"
      version: 8  # 8.x 버전 이상은 무조건 8로 통일
      authentication:
        username:
          secretName: ocp-es-secret
          key: username
        password:
          secretName: ocp-es-secret
          key: password
    tls:
      insecureSkipVerify: true
  pipelines:
  - name: infra-logs-to-eck
    inputRefs:
    - infrastructure
    outputRefs:
    - eck-elasticsearch
```
```
$ oc apply -f clusterlogforwarder.yaml
$ oc logs -f -n openshift-logging infra-logforwarder-instance-xxxxx -c collector
```





