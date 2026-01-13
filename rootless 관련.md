# 1. Root Group 권한 부여
> OpenShift는 보안을 위해 컨테이너를 실행할 때 **임의의 사용자 ID (Random UID)**를 할당합니다 (예: 1000580000). <br/>
> 하지만, Dockerfile에서 User ignite 등으로 소유권을 설정해 두면, 랜덤으로 생성된 UID는 해당 파일의 소유자가 아니므로 쓰기 권한이 없어 에러가 발생합니다. <br/>
> gid 0 권한을 부여할 경우, chgrp 0 에 따라 해당 경로는 gid 0 이어서 rwx 권한을 갖게되고, 기타 uid 0가 소유한 경로는 r-w 권한만을 갖게되어 쓰기가 불가능해집니다.
```
$ vi Dockerfile
# 공식 이미지를 베이스로 사용
FROM apacheignite/ignite:3.1.0

# 공식 이미지도 OCP에서 돌리려면 권한 조정이 필요할 수 있음
USER root
RUN chgrp -R 0 /opt/ignite/apache-ignite && \
    chmod -R g=u /opt/ignite/apache-ignite

# 다시 ignite 유저로 전환 (또는 OCP 랜덤 유저 사용을 위해 생략)
USER ignite
```
> Root Group 권한 부여 방식 대신 non-root-v2 SCC를 해당 프로젝트에 부여하는 방법도 있습니다.

|구분|GID 0 전략 (OpenShift 기본)|nonroot-v2 SCC 부여 (옵션)|
|UID 할당|랜덤 (예: 1000670000)|고정 (예: 1000)|
|보안 강도|🔒 최상 (Best)|🔓 보통 (OpenShift 기준 낮아짐)|
|관리자 개입,없음 (그냥 배포하면 됨)|필요 (ServiceAccount, RoleBinding 생성)|
|이미지 요건|까다로움 (chgrp 0, chmod g=u 필수)|관대함 (일반 Dockerfile도 잘 됨)|
|NFS 소유권|랜덤 숫자로 저장됨 (관리 불편)|1000으로 저장됨 (관리 편함)|

# 2. readOnlyRootFilesystem SecurityContext 설정
> 이 설정은 **"불변 인프라(Immutable Infrastructure)"**를 구현하는 핵심 기술입니다. <br/>
> 해킹 방지: 공격자가 웹쉘 등을 통해 침투하더라도, 악성 코드를 다운로드하거나 실행 파일을 심을 공간이 없습니다.(시스템 영역이 잠겨 있음) <br/>
> 변조 방지: 애플리케이션이나 운영자가 실수로 시스템 설정 파일을 지우거나 바꾸는 사고를 원천 차단합니다.
```
...
spec:
  template:
    spec:
      containers:
      - name: aaa
        securityContext:
          readOnlyRootFilesystem: true
```
> 주의사항 : 쓰기 권한이 필요한 모든 경로가 막히기 때문에 해당 Application이 꼭 써야하는 경로는 별도 볼륨 마운트 필요 <br/>
> ex) Ignite 같은 경우 /opt/ignite 와 /tmp 인데, /opt/ignite는 PV를 할당하므로 /tmp에 대한 emptyDir 할당 필요

# 3. YAML 예시
```
spec:
  containers:
    - name: ignite
      securityContext:
        readOnlyRootFilesystem: true  # [설정 적용]
      volumeMounts:
        - name: data
          mountPath: /opt/ignite/work # (기존) PVC - 쓰기 가능
        - name: ignite-config
          mountPath: /opt/ignite/etc/ignite-config.conf
          subPath: ignite-config.conf # (기존) ConfigMap - 읽기 전용이라 괜찮음
        
        # [필수 추가] 임시 파일용 공간 확보
        - name: tmp-volume
          mountPath: /tmp
      
  volumes:
    - name: data
      persistentVolumeClaim: ...
    - name: ignite-config
      configMap: ...
      
    # [필수 추가] 메모리 기반의 임시 볼륨 (껐다 켜면 사라짐)
    - name: tmp-volume
      emptyDir: {}
```
