> 전제조건 1) 넥서스 5000, 5001 포트 각각 사용하는 레지스트리 구성 <br/>
> 전제조건 2) 5000 포트 사용하는 레지스트리 : 기 설치된 오퍼레이터 및 OCP 미러 레지스트리 역할 <br/>
> 전제조건 3) 5001 포트 사용하는 레지스트리 : 업그레이드를 위한 미러 레지스트리 역할 <br/>
> 해당 문서에서는 web-terminal 오퍼레이터만 업그레이드 하는 케이스를 정리합니다.

### ** 기존 미러 레지스트리 상태
```
# curl -u 'admin:redhat1!' -X GET https://ocp-registry.kscada.kdneri.com:5000/v2/_catalog | jq .
{
  "repositories": [
    "ocp4/openshift/release",
    "ocp4/openshift/release-images",
    "olm-certified/elastic/eck",
    "olm-certified/elastic/eck-operator",
    "olm-certified/redhat/certified-operator-index",
    "olm-redhat/devworkspace/devworkspace-operator-bundle",
    "olm-redhat/devworkspace/devworkspace-project-clone-rhel9",
...
    "olm-redhat/workload-availability/self-node-remediation-rhel9-operator",
    "openshift-logging/eventrouter-rhel9",
    "openshift/graph-data",
    "oss/elasticsearch/elasticsearch",
    "rhel9/support-tools"
  ]
}
```
> ocp4 : OCP 설치용 미러 이미지 저장 경로 <br/>
> olm-certified : Certified Operator 미러 이미지 저장 경로 <br/>
> olm-redhat : Redhat Operator 미러 이미지 저장 경로 <br/>
> openshift-logging, rhel9 : OCP 관리 툴 미러 이미지 저장 경로 <br/>
> openshift : OSUS를 위한 미러 이미지 저장 경로 <br/>
> oss : oss 용 컨테이너 이미지 저장 경로


# 1. 오퍼레이터 업그레이드 패스 확인
> opm 명령어를 통해 확인한 각 버전 별 Replaces, Skips, SkipRange 정보를 확인하여 업그레이드를 위해 필요한 버전들을 확인합니다.
```
$ opm render registry.redhat.io/redhat/redhat-operator-index:v4.20 \
| jq 'select(.package == "web-terminal" and .schema == "olm.channel")
| { ChannelName: .name, UpgradeGraph: [.entries[] | { Bundle: .name, Replaces: .replaces, Skips: .skips, SkipRange: .skipRange }] }'
{
  "ChannelName": "fast",
  "UpgradeGraph": [
    {
      "Bundle": "web-terminal.v1.10.0",
      "Replaces": null,
      "Skips": null,
      "SkipRange": null
    },
...(중략)
    {
      "Bundle": "web-terminal.v1.14.0",
      "Replaces": "web-terminal.v1.13.0",
      "Skips": null,
      "SkipRange": null
    },
    {
      "Bundle": "web-terminal.v1.15.0",
      "Replaces": "web-terminal.v1.14.0",
      "Skips": null,
      "SkipRange": null
    }
  ]
}
```
> oc mirror 명령어로는 skip 관련 정보 확인이 불가능하여 opm 명령어로 확인할 것을 권장합니다.
```
$ oc mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.20 --package web-terminal
NAME          DISPLAY NAME  DEFAULT CHANNEL
web-terminal                fast

PACKAGE       CHANNEL  HEAD
web-terminal  fast     web-terminal.v1.15.0
```

# 2. 업그레이드를 위한 미러 이미지 준비
> 미러링 할 오퍼레이터 정보를 imageset-config.yaml에 작성합니다. <br/>
> 업그레이드를 위해서는 현재버전->업그레이드 할 버전 간 업그레이드 패스에 맞는 버전들 모두가 imageset-config.yaml에 포함되어야 합니다.
```
$ vi imageset-config-for-upgrade.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
      - name: web-terminal
        defaultChannel: fast
        channels:
        - name: fast
          minVersion: 1.8.0
          maxVersion: 1.8.0
        - name: fast
          minVersion: 1.9.0
          maxVersion: 1.9.0
vi imageset-config-for-new-version.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
      - name: web-terminal
        defaultChannel: fast
        channels:
        - name: fast
          minVersion: 1.9.0
          maxVersion: 1.9.0
```
```
$ mkdir -p mirror-data/{for-upgrade,for-new-version}
$ oc mirror -c imageset-config-for-upgrade.yaml \
  file://mirror-data/for-upgrade \
  --v2 \
  --authfile /root/pull-secret.json
$ oc mirror -c imageset-config-for-new-version.yaml \
  file://mirror-data/for-new-version \
  --v2 \
  --authfile /root/pull-secret.json
$ tar -czvf mirror-data.tar.gz mirror-data
```

# 3. 이미지 반입 후 업로드 & 클러스터 적용
```
$ tar -xzvf mirror-data.tar.gz
$ oc mirror \
  --from file://mirror-data/for-upgrade \
  docker://nexus.xxx.xxx.xxx:5001/olm-redhat \
  --v2 \
  --authfile /root/pull-secret.json
```
> mirror-data/for-upgrade/working-dir/cluster-resources/의 idms, catalogsource, clustercatalog 파일의 .metadata.name를 변경하여 기존 object를 덮어쓰지 않도록 합니다.
```
$ oc create -f mirror-data/for-upgrade/working-dir/cluster-resources/
```

# 4. 오퍼레이터 업그레이드 후 정상동작 확인
```
$ oc get subs -A
$ oc edit subscription <subs_name> -n <ns_name>
```
> subs의 .spec.source 값을 신규 반입한 catalogsource name으로 변경 <br/>
> OCP 웹콘솔에서 업그레이드 수행

# 5. 기존 버전 오퍼레이터 이미지 삭제
```

```

# 6. 신규 버전 이미지 업로드 & 클러스터 적용
```

```

# 7. 오퍼레이터 정상동작 확인 및 넥서스 레지스트리 blob 정리
```

```
