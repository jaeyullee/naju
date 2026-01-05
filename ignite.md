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

## 3. values.yaml 수정 후 차트 패키징
```
$ vi ignite-kube/values.yaml
image:
  registry: docker.io
  repository: apache/ignite    ## 수정. 기존 이미지 쓰면 라이선스 문제 있음
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

# (선택) 서비스 어카운트 이름 고정 (나중에 권한 줄 때 편함)
serviceAccount:
  create: true
  name: "my-ignite-sa"

# ... 나머지 설정 유지 혹은 수정

$ helm package ignite-kube/
```
> 결과: ignite-kube-x.x.x.tgz 파일 생성됨

## 4. ChartMuseum으로 업로드 (curl 사용)
> <chart-museum-server-ip> 부분을 실제 IP로 변경하세요.
```
$ curl --data-binary "@ignite-kube-2.16.0.tgz" http://<chart-museum-server-ip>:8080/api/charts
$ helm repo update
$ helm search repo my-private-repo/
```

## 5. 배포
```
$ oc new-project ignite-cluster
$ oc adm policy add-scc-to-user anyuid -z default -n ignite-cluster
$ oc adm policy add-scc-to-user anyuid -z my-ignite-sa -n ignite-cluster

$ helm install my-ignite my-internal-repo/ignite-kube -n ignite-cluster
$ oc get pods -n ignite-cluster
```
