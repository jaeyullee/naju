## 1. 차트 및 종속성 다운로드
```
$ git clone https://github.com/SonarSource/helm-chart-sonarqube.git
$ cd helm-chart-sonarqube/charts/sonarqube
$ helm dependency update .
$ helm package sonarqube
```

## 2. 설정 변경
```
$ tar -zxvf sonarqube-2025.5.0.tgz
$ cp sonarqube/values.yaml my-sonarqube-values.yaml
```
```
$ vi my-sonarqube-values.yaml
image:
  repository: bastion.ocp419.test:5001/sonarqube/sonarqube ## 수정
  tag: "latest"  ## 수정

env:  ## 추가(원래는 #처리되어있음)
  # 1. 관리자 비밀번호 강제 초기화 비활성화 (admin/admin 유지 허용)
  - name: SONAR_FORCEADMINPASSWORDCHANGE
    value: "false"

  # 2. 비밀번호 최소 길이 완화 (예: 8자리)
  - name: SONAR_SECURITY_REALM_MINPASSWORDLENGTH
    value: "8"

  # 3. 대문자 필수 제한 해제
  - name: SONAR_SECURITY_REALM_PASSWORD_MUSTHAVEUPPERCASE
    value: "false"

  # 4. (선택 사항) 숫자 필수 제한 해제
  - name: SONAR_SECURITY_REALM_PASSWORD_MUSTHAVEDIGIT
    value: "false"

postgresql:
  enabled: true
  image:
    registry: "bastion.ocp419.test:5001"  ## 추가
    repository: "sonarqube/postgresql"    ## 수정
    tag: "14.5.0"    ## 수정
```

## 3. 이미지 미러링 및 배포 준비
```
$ oc new-project sonarqube

$ oc image mirror docker.io/library/sonarqube:latest bastion.ocp419.test:5001/sonarqube/sonarqube:latest
$ oc image mirror docker.io/bitnamilegacy/postgresql:14.5.0-debian-11-r35 bastion.ocp419.test:5001/sonarqube/postgresql:14.5.0

$ mkdir -p /nfs/sonarqube/{postgresql,sonarqube}
$ chmod 777 -R /nfs/sonarqube
$ vi /etc/exports
$ exportfs -r
$ showmount -e
```
## 4. scc 설정 (원랜 anyuid면 되는데 잘 안돼서 privileged로 함)
```
$ oc adm policy add-scc-to-user privileged -z default -n sonarqube
```

## 5. helm차트로 배포
```
$ export MONITORING_PASSCODE="redhat1!"
$ helm install sonarqube -f my-sonarqube-values.yaml sonarqube-2025.5.0.tgz -n sonarqube --set edition=developer,monitoringPasscode=$MONITORING_PASSCODE
$ oc expose svc/sonarqube-sonarqube
```
