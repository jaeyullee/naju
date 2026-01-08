# fluent-bit helm 설치

## 1. 이미지 준비
```
$ oc new-project fluent
$ podman pull fluent/fluent-bit:2.2.2
$ podman tag 62b8aa51bb69 bastion.ocp416.test:5000/fluent/fluent-bit:2.2.2
```

## 2. scc 설정
```
$ wget https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-openshift-security-context-constraints.yaml
$ oc apply -f fluent-bit-openshift-security-context-constraints.yaml
$ oc adm policy add-scc-to-user fluentd -z default -n fluent
$ oc create sa fluent-bit -n fluent
$ oc adm policy add-scc-to-user fluentd -z fluent-bit -n fluent
$ oc adm policy add-scc-to-user privileged -z fluent-bit -n fluent
```

## 3. elastic 연결정보 secret 설정
```
$ oc get secret elasticsearch-cluster-es-http-certs-public -n elastic-system -o json |   jq 'del(.metadata.creationTimestamp, .metadata.labels, .metadata.resourceVersion, .metadata.uid, .metadata.namespace)' |  oc apply -n fluent -f -
$ oc create secret generic fluent-bit-es-creds -n fluent --from-literal=password="$(oc get secret elasticsearch-cluster-es-elastic-user -n elastic-system -o go-template='{{.data.elastic|base64decode}}')"
```

## 4. helm 설치 및 fluent 레포 구성
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

$ helm repo add fluent https://fluent.github.io/helm-charts
$ helm repo update
$ helm search repo fluent/fluent-bit --versions
$ helm pull fluent/fluent-bit --untar --version 0.43.0
```

## 5. fluent 설정 변경 후 워크로드 배포
```
$ vi fluent-bit/values.yaml
image:
  repository: bastion.ocp416.test:5000/fluent/fluent-bit        # 원래 public 이미지 주소인데 변경
  tag: 2.2.2        # 원래 비어있는데 추가
  digest:
  pullPolicy: Always
serviceAccount:
  create: false            # 원래 true 인데 변경
  annotations: {}
  name: fluent-bit         # 원래 비어있는데 추가
openShift:
  enabled: true         # 원래 false 인데 변경
  securityContextConstraints:
    # Create SCC for Fluent-bit and allow use it
    create: false         # 원래 true 인데 변경
    existingName: "privileged"         # 원래 비어있는데 추가
securityContext:
  privileged: true         # 원래 비어있는데 추가
env:         # 원래 비어있는데 추가
  - name: ES_PASS
    valueFrom:
      secretKeyRef:
        name: fluent-bit-es-creds
        key: password
extraVolumes:         # 내용이 비어있으니 추가
  - name: es-ca
    secret:
      secretName: elasticsearch-cluster-es-http-certs-public
extraVolumeMounts:         # 내용이 비어있으니 추가
  - name: es-ca
    mountPath: /fluent-bit/etc/es-ca
config:
  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
        Buffer_Chunk_Size 64k         # default값 변경을 위해 튜닝
        Buffer_Max_Size 64k         # default값 변경을 위해 튜닝
  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On
        Buffer_Size 64k         # default값 변경을 위해 튜닝
  outputs: |
    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch-cluster-es-http.elastic.svc
        Port            9200
        tls             On
        tls.verify      On
        tls.ca_file     /fluent-bit/etc/es-ca/ca.crt
        Logstash_Format On
        Logstash_Prefix fluentd
        index           fluentd
        Suppress_Type_Name On
        http_user       elastic
        http_passwd     ${ES_PASS}
		Buffer_Size     512k    # 명시적 표시를 위해 추가
secrets:         # 없으니 추가
  - name: es-creds
    secretKeyRef:
      name: fluent-bit-es-creds
      key: password

$ helm install fluent-bit fluent/fluent-bit -n fluent -f values.yaml
```



## 참고. (fluent-bit 이미지 커스텀)
```
$ vi Containerfile
FROM docker.io/fluent/fluent-bit:2.2.2 AS builder
FROM alpine:3.18
RUN apk add --no-cache bash libc6-compat curl
COPY --from=builder / /
RUN addgroup -S fluent-bit && adduser -S -G fluent-bit fluent-bit
WORKDIR /fluent-bit
USER fluent-bit
ENTRYPOINT ["/fluent-bit/bin/fluent-bit"]
CMD ["-c", "/fluent-bit/etc/fluent-bit.conf"]

$ podman build -t bastion.ocp416.test:5000/fluent/fluent-bit:test
$ podman push bastion.ocp416.test:5000/fluent/fluent-bit:test
```
