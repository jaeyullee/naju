# 1. gitops component 기본 리소스

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

# 2. OpenShift GitOps 오퍼레이터 설치
> 콘솔 GUI를 이용해서 오퍼레이터 설치  
```
$ oc get argocd -n openshift-gitops
NAME               AGE
openshift-gitops   15m
```

# 3. RHBK 설정
> groups 생성  
  - Name : gitops-admins  
> users 생성  
  - Username : gitopsadmin  
  - Email, First name, Last name 입력  
> user password 설정 (Credentials > Set password)  
  - Password 설정  
  - Temporary : Off  
> clients 생성  
  - Client ID : gitops-kdneri-client  
  - Client authentication : On  
  - Authentication flow : Standard flow, Direct access grants  
  - Root URL : https://ocp-gitops-cluster-server-openshift-gitops.apps.kscada.kdneri.com  
  - Valid redirect URIs : https://ocp-gitops-cluster-server-openshift-gitops.apps.kscada.kdneri.com/auth/callback  
  - Valid post logout redirect URIs : https://ocp-gitops-cluster-server-openshift-gitops.apps.kscada.kdneri.com  
> client scope 설정 (Client scopes > gitops-kdneri-client-dedicated > Add mapper)  
  - By configuration > Group Membership  
  - Name : gitops-admins  
  - Token Claim Name : groups  
  - Full group path : Off  
 
# 4. ArgoCD 인스턴스 설정 변경
> GitOps 관련 파드가 Infra 노드에 배포되도록 설정 및 클러스터 전체에 배포할수 있도록 권한 부여  
```
$ oc get argocd -n openshift-gitops
$ oc patch argocd ocp-gitops-cluster -n openshift-gitops --type=merge -p '
{
  "spec": {
    "nodePlacement": {
      "nodeSelector": {
        "node-role.kubernetes.io/infra": ""
      },
      "tolerations": [
        {
          "effect": "NoSchedule",
          "key": "role",
          "operator": "Equal",
          "value": "infra"
        }
      ]
    }
  }
}'
```
> ❗<client_secret> 와 \<domain\> 부분 수정하고 작업하기  
```
$ oc patch argocd ocp-gitops-cluster -n openshift-gitops --type=merge -p "
spec:
  oidcConfig: |
    name: Keycloak
    issuer: https://sso.kscada.kdneri.com/realms/ocp-kdneri
    clientID: gitops-kdneri-client
    clientSecret: <client_secret>
    requestedScopes: [\"openid\", \"profile\", \"email\", \"gitops-groups\"]
    logoutUrl: https://sso.kscada.<domain>/realms/ocp-kdneri/protocol/openid-connect/logout?id_token_hint={{token}}&post_logout_redirect_uri={{logoutRedirectURL}}
    rootCA: |
      -----BEGIN CERTIFICATE-----
      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
      3JWJ6BeHXoPnaj43E2i7jRoObWr0ORVnu4VPtpg=
      -----END CERTIFICATE-----
  rbac:
    policy: \"g, gitops-admins, role:admin\"
    scopes: \"[gitops-groups]\"
"
```
```
$ oc rollout status deployment/openshift-gitops-server -n openshift-gitops
$ oc get argocd openshift-gitops -n openshift-gitops -o yaml | grep -A 20 oidcConfig
```
> gitops 로그인 테스트해보기  
> ❗참고 : 동일한 네임스페이스에 두 개의 Argo CD CR을 생성할 수 없습니다.  


# 5. ArgoCD 인스턴스가 배포된 프로젝트 외 다른 프로젝트에 리소스 배포하기 위해 다른 프로젝트에 레이블 설정(필수)
```
$ oc label namespace <target_namespace> argocd.argoproj.io/managed-by=openshift-gitops
```

# 6. ArgoCD 알람
> Argo CD 알림을 사용하면 Argo CD 인스턴스에서 이벤트가 발생할 때 외부 서비스에 알림을 보낼 수 있습니다. 예를 들어 동기화 작업이 실패하면 Slack 또는 이메일에 알림을 보낼 수 있습니다. 기본적으로 알림은 Argo CD 인스턴스에서 비활성화되어 있습니다.  
> 참고 1 : https://docs.redhat.com/ko/documentation/red_hat_openshift_gitops/1.16/html-single/argo_cd_instance/index#gitops-argo-cd-notification_argo-cd-cr-component-properties  
> 참고 2 : https://docs.redhat.com/ko/documentation/red_hat_openshift_gitops/1.16/html-single/argo_cd_instance/index#notifications-configuration-custom-resource-properties_argo-cd-cr-component-properties
