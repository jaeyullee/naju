> servicemesh3 오퍼레이터 샘플

# 1. imageset-config.yaml 작성
```
vi imageset-config-servicemesh3.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
      - name: servicemeshoperator3
        defaultChannel: stable-3.2
        channels:
        - name: stable-3.2
          minVersion: 3.2.1
          maxVersion: 3.2.1
```

# 2. 미러링 수행
```
$ oc mirror -v2 --config=./imageset-config.yaml \
  docker://nexus.kscada.kdneri.com:5002/servicemesh-3-2-1 \
  --dir=file://./mirror-1 \
  --authfile=/root/pull-secret.json
```

# 3. catalogsource, idms 배포
> mirror-1/working-dir/cluster-resources의 cs 및 idms가 기존 배포된 오브젝트와 충돌하지 않도록 파일 이름 변경
```
$ oc apply -f ./mirror-1/working-dir/cluster-resources/cs-xxxx.yaml
$ oc apply -f ./mirror-1/working-dir/cluster-resources/idms-xxxx.yaml
$ watch oc get node,mcp
```

# 4. OCP 콘솔에서 카탈로그에 오퍼레이터 생겼는지 확인
