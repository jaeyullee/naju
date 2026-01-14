## 1. Gateway API 기능 활성화
> Gateway API 기능은 GA되긴했으나 기존 Ingress와의 충돌 방지 및 리소스 절약을 위해 수동으로 켜야(Opt-in) 합니다.
```
$ oc patch featuregate cluster --type='merge' -p '{"spec": {"featureSet": "TechPreviewNoUpgrade"}}'
```

## 2. Service Mesh 3 Operator 설치
> OpenShift 4.20의 "Native Gateway API" 기능은 내부적으로 "Red Hat OpenShift Service Mesh v3 (Sail Operator)"를 엔진으로 사용하기 때문에 설치가 필요합니다. <br/>

> - The OpenShift Container Platform Gateway API implementation relies on the Cluster Ingress Operator (CIO) to install and manage a specific version of OpenShift Service Mesh (OSSM v3.x) in the openshift-ingress namespace.
>  [[문서](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ingress_and_load_balancing/configuring-ingress-cluster-traffic#nw-ingress-gateway-api-enable_ingress-gateway-api)] <br/>
> - We’ll need both the OpenShift Service Mesh operator (for Gateway API support) and our Istio installation.
>  [[문서](https://developers.redhat.com/articles/2025/12/09/integrate-openshift-gateway-api-openshift-service-mesh#install_the_service_mesh_operators)[

## 3. Gateway API 배포
```
$ vi gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1

$ oc create -f gatewayclass.yaml
$ oc get gatewayclass -n openshift-default
$ oc get deployment -n openshift-ingress
```

## 4. Gateway 생성
> 공유 Gateway를 쓸 경우 포트 중복 문제가 발생할 수 있기 때문에 서비스 별 전용 Gateway를 사용하는 방식으로 진행합니다. <br/>
> 각 서비스 네임스페이스에 Gateway를 배포할 경우 관리가 힘들어지기 때문에 Gateway 전용 네임스페이스를 이용합니다.
```
$ oc new-project gateway-ns
$ oc annotate namespace gateway-ns openshift.io/node-selector="node-role.kubernetes.io/router=" --overwrite
$ oc get namespace gateway-ns -o yaml

$ vi gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-tcp-gateway
  namespace: gateway-ns
spec:
  gatewayClassName: openshift-default
  listeners:
  - name: tcp-9000
    protocol: TCP
    port: 9000
    allowedRoutes:
      namespaces:
        from: All  # 중요: 다른 네임스페이스(tcp-app-ns)의 연결 허용

$ oc create -f gateway.yaml
```

## 5. TCP 테스트 앱 생성
> 테스트를 위해 앱 배포합니다. <br/>
> alpine/socat 이미지 반입이 필요합니다.
```
$ vi tcp-test-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcp-echo-server
  namespace: tcp-app-ns
  labels:
    app: tcp-echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tcp-echo
  template:
    metadata:
      labels:
        app: tcp-echo
    spec:
      containers:
      - name: socat
        image: nexus.kscada.kdneri.com:5002/app/alpine/socat:latest
        args: ["-v", "TCP-LISTEN:9000,fork", "EXEC:/bin/cat"]
        ports:
        - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: tcp-echo-service
  namespace: tcp-app-ns
spec:
  selector:
    app: tcp-echo
  ports:
  - port: 9000
    targetPort: 9000
    protocol: TCP

$ oc new-project tcp-app-ns
$ oc create -f tcp-test-app.yaml
$ oc get po -w
```

## 5. TCPRoute 생성
> 아직 OCP에서는 해당 기능 지원 안하는것으로 확인되었습니다. <br/>
> Servicemesh3 stable-3.2 버전으로 설치하면 기능은 있으나 DP 지원단계입니다. <br/>

> - Gateway API Experimental Channel (TLSRoute, TCPRoute) : DP
>  [[문서](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.2/html-single/release_notes/index#istio-ambient-mode_ossm-release-notes-support-tables)]

```
$ vi tcproute.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: echo-route
  namespace: tcp-app-ns      # TCP 서비스 파드가 있는 네임스페이스 지정
spec:
  parentRefs:
  - name: my-tcp-gateway
    namespace: gateway-ns    # Gateway가 있는 네임스페이스 지정
    sectionName: tcp-9000 # Gateway의 리스너 이름 지정
  rules:
  - backendRefs:
    - name: tcp-echo-service
      port: 9000

$ oc create -f tcproute.yaml
```
