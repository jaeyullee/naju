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

# 3. Kibana
## 3-1. kibana 배포
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

## 3-2. kibana - elasticsearch 간 연동에 실패하는 경우 (인증 실패)
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

## 3-3. kibana 정상 기동 확인
> Kibana is now available: 서비스 정상 기동 완료. <br/>
> plugins-service ... fleet is disabled: 불필요한 Fleet 기능 차단 확인. <br/>
> ENOTFOUND artifacts.security.elastic.co: 폐쇄망이라 발생하는 업데이트 체크 실패 로그 (무시 가능 확인).

## 3-4. index 템플릿 생성 및 보관주기 설정


# 4. Collector
## 4-1. clusterLoggingForwarder 배포
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

# 5. 로그 인덱스 설정 및 조회
## 5-1. 인덱스 템플릿 생성
> 1. Kibana 콘솔 로그인
> 2. 왼쪽 사이드바 메뉴(줄 3개 아이콘) > Management > Stack Management
> 3. 왼쪽 Data > Index Management
> 4. Index Templates > Create template
> 5. Name : infra-logs-template / index patterns : infra-logs 입력 후 Next
> 6. Next
> 7. Index settings에 아래처럼 입력 후 Next
```
{
  "index": {
    "lifecycle": {
      "name": "infra-logs-policy",
      "rollover_alias": "infra-logs"
    },
    "mode": "standard"
  }
}
```
> 8. 끝까지 Next 후 Save template
> 9. 왼쪽 사이드바 메뉴(줄 3개 아이콘) > Management > Dev Tools
> 10. Shell 에서 아래처럼 입력 후 Ctrl+Enter 하여 {"acknowledged": true, ...} 결과 확인
```
PUT %3Cinfra-logs-%7Bnow%2Fd%7D-000001%3E
{
  "aliases": {
    "infra-logs": {
      "is_write_index": true
    }
  }
}
```

> 기타 로그 타입들에 대한 설정은 생략합니다.

## 5-2. 인덱스 정책 생성
> 1. Kibana 콘솔 로그인
> 2. 왼쪽 사이드바 메뉴(줄 3개 아이콘) > Management > Stack Management
> 3. 왼쪽 Data > Index Lifecycle Policies
> 4. Create policy
> 5. Hot phase > Advanced settings 선택하여 아래와같이 설정
>    * Rollover > Use recommended defaults 비활성화
>        * Maximum primary shard size : 50 gigabytes (변경)
>        * Maximum age : 1 days (변경)
> 6. Warm phase 활성화 > Advanced settings 선택하여 아래와같이 설정
>    * Shirink > Shirink index 활성화
>    * Force merge > Force merge data 활성화
>        * Number of segments : 1 (설정)
> 7. Warm phase 오른쪽에 (Keep data in this phase forever ♾️🗑️) 라고 되어있는 부분에서 🗑️ 를 선택하여 (Delete data after this phase ♾️🗑️) 로 변경
> 8. Delete phase 아래와 같이 설정
>    * Move data into phase when : 14 days

> ❗위 값은 기본적인 운영상황에서 추천하는 설정값입니다. elasticsearch에 할당된 로그저장소 크기 및 수집로그양에 따라 적절한 튜닝이 필요합니다.

## 5-3. kibana 대시보드 데이터뷰 생성
> 1. Kibana 콘솔 로그인
> 2. 왼쪽 사이드바 메뉴(줄 3개 아이콘) > Management > Stack Management
> 3. 왼쪽메뉴 Kibana > Data Views
> 4. Create data view
> 5. Name : 원하는대로 / Index pattern : 오른쪽의 인덱스 목록보고 적절히 결정 / Timestamp field : @timestamp 입력 후 Save data view to Kibana

## 5-4. kibana 대시보드 로그 조회
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
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k \
  -XPOST "https://localhost:9200/_security/user/kibanaadmin" \
  -H "Content-Type: application/json" \
  -d '{
    "password" : "redhat1!",
    "roles" : [ "kibana_admin", "viewer" ],
    "full_name" : "Kibana Admin User"
  }'
```
* 삭제
```
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k -XDELETE "https://localhost:9200/_security/user/admin"
```
* 조회
```
$ oc exec -it ocp-es-node-1-0 -n ocp-es -- curl -u "elastic:[ES_PW]" -k -XGET "https://localhost:9200/_security/user"
```

> 유저 role 리스트 <br/>

|**역할구분**|**역할 이름**|**권한 범위**|**용도**|
|:---|:---|:---|:---|
|운영/관리|cluster_admin|클러스터의 모든 관리 권한 (노드 설정, 스냅샷, 샤드 재배치 등)|인프라 운영자용|
|&nbsp;|cluster_monitor|클러스터 상태 조회 및 노드 통계 확인 (수정 불가)|모니터링 대시보드 연결용|
|&nbsp;|security_admin|사용자 생성, 삭제, 권한 부여 등 보안 관련 모든 설정|보안 담당자용|
|데이터처리|all|모든 인덱스에 대해 모든 작업(CRUD) 가능|데이터 총괄 관리자|
|&nbsp;|editor|인덱스 생성/삭제 및 데이터 읽기/쓰기 가능|개발자 또는 DBA|
|&nbsp;|viewer|데이터 읽기(Search, Get)만 가능 (수정 불가)|일반 사용자 (로그 조회용)|
|&nbsp;|read|특정 인덱스의 데이터 읽기 권한|가장 좁은 범위의 조회 전용|
|Kibana 전용|역할 이름|권한 범위|용도|
|&nbsp;|kibana_admin|Kibana의 모든 기능 사용 (대시보드 생성, 설정 변경 등)|Kibana 관리자|
|&nbsp;|kibana_user|Kibana 메뉴 접속 및 대시보드 확인 가능|일반 분석가|
|&nbsp;|monitoring_user|Kibana 내의 Stack Monitoring 메뉴 확인 가능|시스템 관제|

> [[참고문서]](https://www.elastic.co/docs/api/doc/elasticsearch)
