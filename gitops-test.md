1. gitops component 기본 리소스

|워크로드|CPU 요청|CPU 제한|메모리 요청|메모리 제한|
|:---|:---|:---|:---|:---|
|argocd-application-controller|1|2|1024Mi|2048Mi|
|applicationset-controller|1|2|512Mi|1024Mi|
|argocd-server|0.125|0.5|128Mi|256Mi|
|argocd-repo-server|0.5|1|256Mi|1024Mi|
|argocd-redis|0.25|0.5|128Mi|256Mi|
|argocd-dex|0.25|0.5|128Mi|256Mi|
|HAProxy|0.25|0.5|128Mi|256Mi|

> ❗참고 : GitOps 버전 1.10 이상의 경우 기본 네임스페이스가 openshift-operators 에서 openshift-gitops Operator 로 변경되었습니다.

2. OpenShift GitOps 오퍼레이터 설치
> 콘솔 GUI를 이용해서 오퍼레이터 설치
```
$ oc get argocd -n openshift-gitops
NAME               AGE
openshift-gitops   15m
```

3. 기본 구성되는 ArgoCD 인스턴스 삭제
```
$ oc patch subscription openshift-gitops-operator -n openshift-gitops-operator --type=merge -p '{"spec":{"config":{"env":[{"name":"DISABLE_DEFAULT_ARGOCD_INSTANCE","value":"true"}]}}}'
$ oc get subs -n openshift-gitops-operator openshift-gitops-operator -oyaml
...
spec:
  channel: latest
  config:    ## 해당열부터 아래내용 추가되었는지 확인
    env:
    - name: DISABLE_DEFAULT_ARGOCD_INSTANCE
      value: "true"
...
```
```
$ oc delete argocd openshift-gitops -n openshift-gitops
$ oc delete appproject default -n openshift-gitops
```

4. ArgoCD 인스턴스 배포
> GitOps 관련 파드가 Infra 노드에 배포되도록 설정 및 클러스터 전체에 배포할수 있도록 권한 부여
```
$ oc new-project ocp-gitops
$ oc annotate namespace ocp-gitops openshift.io/node-selector='node-role.kubernetes.io/infra=' --overwrite
$ oc annotate namespace ocp-gitops scheduler.alpha.kubernetes.io/defaultTolerations='[{"key": "role", "value": "infra", "operator": "Equal", "effect": "NoSchedule"}]' --overwrite
$ oc label namespace ocp-gitops argocd.argoproj.io/managed-by-cluster-argocd=true --overwrite
```
```
$ vi argocd.yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: ocp-gitops-cluster
  namespace: ocp-gitops
spec:
  monitoring:
    enabled: true
  server:
    insecure: true
    route:
      enabled: true
      tls:
        termination: edge
  sso:
    provider: dex
    dex:
      config: |
       connectors:
          - type: oidc
            id: keycloak
            name: Keycloak
            config:
              issuer: https://<keycloak-url>/auth/realms/<realm>
              clientID: <client-id>
              clientSecret: $argocd-keycloak-secret:clientSecret
```
```
$ oc create -f argocd.yaml
```
> ❗참고 : 동일한 네임스페이스에 두 개의 Argo CD CR을 생성할 수 없습니다.

> 만약 route 주소를 임의로 변경하고 싶은 경우 (*.apps.xxx.xxx.xxx 인 경우만, 그 외의 경우는 아래 설정 이외에도 인증서 설정 등 추가 고려사항이 많음)
```
$ oc patch argocd ocp-gitops-cluster -n ocp-gitops --type=merge -p '{"spec":{"server":{"route":{"enabled":false}}}}'
$ oc create route edge ocp-gitops-custom-route \
  --service=ocp-gitops-cluster-server \
  --port=http \
  --hostname=<GitOps-Domain-URL> \
  --insecure-policy=Redirect \
  -n ocp-gitops
```

5. ArgoCD 인스턴스가 배포된 프로젝트 외 다른 프로젝트에 리소스 배포하기 위해 다른 프로젝트에 레이블 설정(필수)
```
$ oc label namespace <target_namespace> argocd.argoproj.io/managed-by=openshift-gitops
```

6. ArgoCD 알람
> Argo CD 알림을 사용하면 Argo CD 인스턴스에서 이벤트가 발생할 때 외부 서비스에 알림을 보낼 수 있습니다. 예를 들어 동기화 작업이 실패하면 Slack 또는 이메일에 알림을 보낼 수 있습니다. 기본적으로 알림은 Argo CD 인스턴스에서 비활성화되어 있습니다.
> 참고 1 : https://docs.redhat.com/ko/documentation/red_hat_openshift_gitops/1.16/html-single/argo_cd_instance/index#gitops-argo-cd-notification_argo-cd-cr-component-properties
> 참고 2 : https://docs.redhat.com/ko/documentation/red_hat_openshift_gitops/1.16/html-single/argo_cd_instance/index#notifications-configuration-custom-resource-properties_argo-cd-cr-component-properties
