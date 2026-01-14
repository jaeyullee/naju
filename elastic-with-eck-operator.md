> 해당 문서는 Cluster Logging 오퍼레이터와 ECK 오퍼레이터가 이미 미러링되어 준비된 환경을 기준으로 작성합니다.

# 1. Cluster Logging 오퍼레이터 및 ECK 오퍼레이터 설치
> OpenShift 콘솔의 GUI 화면을 이용하여 Operator 설치

# 2. Elasticsearch
## 2-1. elasticsearch 컨테이너 이미지 반입
> ECK 오퍼레이터 버전에 따라 elasticsearch 이미지 버전 결정
> nexus의 도커 레지스트리에 elasticsearch 이미지를 푸시할 경로는 ocp-operators-mirror/eck로 가정.
```
$ podman pull docker.elastic.io/elasticsearch/elasticsearch:9.2.0
$ podman tag docker.elastic.io/elasticsearch/elasticsearch:9.2.0 ocp-registry.xxx.xxx.xxx:5000/ocp-operators-mirror/elasticsearch/elasticsearch:9.2.0
$ podman push ocp-registry.xxx.xxx.xxx:5000/ocp-operators-mirror/elasticsearch/elasticsearch:9.2.0
$ curl -u <id>:<pw> https://ocp-registry.xxx.xxx.xxx:5000/v2/_catalog
```

## 2-2. elasticsearch 컨테이너 이미지 활용을 위한 미러링
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

## 2-3. elasticsearch cluster 배포
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

## 2-4. Elasticsearch Cluster 상태 확인
```
$ oc get elasticsearch -n ocp-es
$ PASSWORD=$(oc get secret ocp-es-elastic-user -n ocp-es -o go-template='{{.data.elastic | base64decode}}')
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:$PASSWORD" -k "https://localhost:9200/_cluster/health?pretty"
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:$PASSWORD" -k "https://localhost:9200/_cat/nodes?v"
$ oc logs -f ocp-es-node-1-0 -n ocp-es
```

# 3. Collector
## 3-1. clusterLoggingForwarder 배포포
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

# 4. Kibana
## 4-1. kibana 배포
```
$ vi kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: ocp-es
spec:
  version: 9.2.0
  count: 2
  image: ocp-registry.xxx.xxx.xxx:5000/oss/kibana/kibana:9.2.0
  elasticsearchRef:
    name: ocp
  config:
    server.maxPayload: 26214400
    logging.root.level: info
    i18n.locale: "ko-KR"
    telemetry.enabled: false
    telemetry.optIn: false
    server.ssl.enabled: false
    xpack.fleet.enabled: false
    xpack.fleet.registryUrl: "http://127.0.0.1"
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  podTemplate:
    spec:
      tolerations:
      - key: "role"
        operator: "Equal"
        value: "infra"
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  kibana.k8s.elastic.co/name: kibana
              topologyKey: kubernetes.io/hostname
      containers:
      - name: kibana
        resources:
          limits:
            memory: 2Gi
            cpu: 2
          requests:
            memory: 2Gi
            cpu: 1
```
```
$ vi kibana-route.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: kibana
  namespace: ocp-es
spec:
  host: kibana-ocp-es.apps.xxx.xxx.xxx
  to:
    kind: Service
    name: kibana-kb-http
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```
```
$ oc create -f kibana.yaml
$ oc create -f kibana-route.yaml
```

## 4-2. kibana - elasticsearch 간 연동에 실패하는 경우 (인증 실패)
```
$ oc extract -n ocp-es secret/ocp-es-elastic-user --to=-
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k \
    -XPOST "https://localhost:9200/_security/service/elastic/kibana/credential/token/ocp-custom-token?pretty"
```
> 생성된 토큰 값 복사!
```
$ oc create secret generic kibana-manual-token -n ocp-es \
  --from-literal=token=[복사한_토큰_값] \
  --dry-run=client -o yaml | oc apply -f -
```
```
$ oc edit kibana kibana -n ocp-es
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: ocp-es
spec:
  version: 9.2.0
  count: 2
  image: ocp-registry.xxx.xxx.xxx:5000/oss/kibana/kibana:9.2.0
  elasticsearchRef:
    name: ocp
  config:
    server.maxPayload: 26214400
    logging.root.level: info
    i18n.locale: "ko-KR"
    telemetry.enabled: false
    telemetry.optIn: false
    server.ssl.enabled: false
    xpack.fleet.enabled: false
    xpack.fleet.registryUrl: "http://127.0.0.1"
    elasticsearch.serviceAccountToken: "${MANUAL_TOKEN}"    ## 추가
    elasticsearch.ssl.verificationMode: "none"              ## 추가
...
  podTemplate:
    spec:
...
      containers:
      - name: kibana
...
        env:                              ## 이하내용 추가
        - name: MANUAL_TOKEN
          valueFrom:
            secretKeyRef:
              name: kibana-manual-token
              key: token
```

## 4-3. kibana 정상 기동 확인
> Kibana is now available: 서비스 정상 기동 완료. <br/>
> plugins-service ... fleet is disabled: 불필요한 Fleet 기능 차단 확인. <br/>
> ENOTFOUND artifacts.security.elastic.co: 폐쇄망이라 발생하는 업데이트 체크 실패 로그 (무시 가능 확인).


## 4-4. kibana 대시보드 데이터뷰 생성
> 1. Kibana 콘솔 로그인
> 2. 왼쪽 사이드바 메뉴(줄 3개 아이콘) > Management > Stack Management
> 3. 왼쪽메뉴 하단의 Kibana > Data Views
> 4. Create data view
> 5. Name : 원하는대로 / Index pattern : 오른쪽의 인덱스 목록보고 적절히 결정 / Timestamp field : @timestamp 입력 후 Save data view to Kibana

## 4-5. kibana 대시보드 로그 조회
> 1. Kibana 콘솔 로그인
> 2. 왼쪽 사이드바 메뉴(줄 3개 아이콘) > Analytics > Discover
> 3. 왼쪽 상단 Dava View 선택창에서 9. 에서 생성한 Data View 선택
> 4. 기능 활용
>    * 시간 범위 조절 (Time Picker): 오른쪽 상단의 시계 아이콘을 클릭하여 로그를 볼 시간 범위(예: Last 15 minutes, Last 24 hours)를 설정합니다. Vector가 실시간으로 로그를 쏘고 있다면 'Last 15 minutes'로 두고 [Refresh] 버튼 옆의 화살표를 눌러 **[Start auto-refresh]**를 켜두는 것이 좋습니다.
>    * 필드 필터링 (Available Fields): 왼쪽 리스트에 log.level, message, host.name 등 Vector가 보낸 필드들이 보일 것입니다. 특정 필드 이름 옆의 [+] 아이콘을 누르면 우측 테이블에 해당 열이 추가되어 표 형태로 깔끔하게 볼 수 있습니다.
>    * 검색 및 필터 (KQL): 상단 검색창에 log.level : "error" 라고 입력하면 에러 로그만 필터링됩니다. KQL(Kibana Query Language)은 자동 완성을 지원하므로 입력하기 편리합니다.
>    * 로그 상세 보기: 리스트의 특정 행 왼쪽 화살표(>)를 누르면 해당 로그의 전체 JSON 내용과 모든 필드 값을 한눈에 확인할 수 있습니다.


# 추가
## 1) Elasticsearch 계정 생성 및 관리
> elastic 기본유저는 패스워드 변경 또는 계정 삭제가 불가능합니다.
* 생성
```
$ oc extract -n ocp-es secret/ocp-es-elastic-user --to=-
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k -XPOST "https://localhost:9200/_security/user/admin" \
   -H "Content-Type: application/json" \
   -d '{      "password" : "redhat1!",
    "roles" : [ "superuser" ],
    "full_name" : "Admin User"
  }'
```
> 유저 role 리스트
|역할 이름|권한 범위|용도|
|:---|:---|:---|
|cluster_admin|클러스터의 모든 관리 권한 (노드 설정, 스냅샷, 샤드 재배치 등)|인프라 운영자용|
|cluster_monitor|클러스터 상태 조회 및 노드 통계 확인 (수정 불가)|모니터링 대시보드 연결용|
|security_admin|사용자 생성, 삭제, 권한 부여 등 보안 관련 모든 설정|보안 담당자용|
* 삭제
```
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k -XDELETE "https://localhost:9200/_security/user/admin"
```
* 조회
```
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k -XGET "https://localhost:9200/_security/user"
```
