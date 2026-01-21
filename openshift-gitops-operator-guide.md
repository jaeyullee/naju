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
> group 생성 <br/>
> user 생성 > password 설정 > group 할당 <br/>
> client 생성 > client authentication 활성화 > Direct access grants 활성화 > root url 등 설정 > client scope - group membership 추가

# 4. ArgoCD 인스턴스 설저 변경
> GitOps 관련 파드가 Infra 노드에 배포되도록 설정 및 클러스터 전체에 배포할수 있도록 권한 부여
```
$ oc create secret generic argocd-keycloak-secret \
  --from-literal=clientSecret=<client_secret> \
  -n openshift-gitops
$ oc create configmap ca-bundle --from-file=ca.crt=./ca.crt -n openshift-gitops
```
```
$ oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p '
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
```
$ oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p '
{
  "spec": {
    "server": {
      "env": [
        {
          "name": "KEYCLOAK_CLIENT_SECRET",
          "valueFrom": {
            "secretKeyRef": {
              "key": "clientSecret",
              "name": "argocd-keycloak-secret"
            }
          }
        },
        {
          "name": "ARGOCD_TLS_CA_PATH",
          "value": "/app/config/server/tls/ca.crt"
        }
      ],
      "volumes": [
        {
          "configMap": {
            "name": "ca-bundle"
          },
          "name": "custom-ca-bundle"
        }
      ],
      "volumeMounts": [
        {
          "mountPath": "/app/config/server/tls",
          "name": "custom-ca-bundle",
          "readOnly": true
        }
      ]
    }
  }
}'
```
```
$ oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p '
{
  "spec": {
    "oidcConfig": "name: Keycloak
 issuer: https://sso.kscada.<domain>/realms/ocp-kdneri
 clientID: gitops-kdneri-client
 clientSecret: $KEYCLOAK_CLIENT_SECRET
 requestedScopes: [\"openid\", \"profile\", \"email\", \"gitops-groups\"]
 rootCA: /app/config/server/tls/ca.crt
 logoutUrl: https://sso.kscada.kdneri.com/realms/ocp-kdneri/protocol/openid-connect/logout?id_token_hint={{token}}&post_logout_redirect_uri={{logoutRedirectURL}}",
    "rbac": {
      "policy": "g, gitops-admins, role:admin",
      "scopes": "[gitops-groups]"
    }
  }
}'
```
```
$ oc rollout status deployment/openshift-gitops-server -n openshift-gitops
$ oc get argocd openshift-gitops -n openshift-gitops -o yaml | grep -A 20 oidcConfig
```
> ❗참고 : 동일한 네임스페이스에 두 개의 Argo CD CR을 생성할 수 없습니다.


# 5. ArgoCD 인스턴스가 배포된 프로젝트 외 다른 프로젝트에 리소스 배포하기 위해 다른 프로젝트에 레이블 설정(필수)
```
$ oc label namespace <target_namespace> argocd.argoproj.io/managed-by=openshift-gitops
```

# 6. ArgoCD 알람
> Argo CD 알림을 사용하면 Argo CD 인스턴스에서 이벤트가 발생할 때 외부 서비스에 알림을 보낼 수 있습니다. 예를 들어 동기화 작업이 실패하면 Slack 또는 이메일에 알림을 보낼 수 있습니다. 기본적으로 알림은 Argo CD 인스턴스에서 비활성화되어 있습니다.  
> 참고 1 : https://docs.redhat.com/ko/documentation/red_hat_openshift_gitops/1.16/html-single/argo_cd_instance/index#gitops-argo-cd-notification_argo-cd-cr-component-properties  
> 참고 2 : https://docs.redhat.com/ko/documentation/red_hat_openshift_gitops/1.16/html-single/argo_cd_instance/index#notifications-configuration-custom-resource-properties_argo-cd-cr-component-properties
