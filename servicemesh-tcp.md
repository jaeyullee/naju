> TCPRoute는 최신버전인 ServiceMesh 3.2에서도 Ambient Mode에서만 D.P. 기능으로 제공하고 있음

# 1. servicemesh3 오퍼레이터의 최신버전 미러링 및 catalogsource,idms 등록

# 2. console에서 operator 설치

# 3. IstioCNI 및 Istio 설치
```
$ oc new-project istio-cni
$ oc new-project istio-system
```
> servicemesh3 오퍼레이터 콘솔에서 Istio CNI 및 Istio 기본값으로 생성 후 State: Healthy 확인

# 4. Gateway 배포 및 설정
```
$ oc new-project istio-ingress-gateway
```
```
$ vi gateway-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secret-reader
  namespace: istio-ingress-gateway
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: istio-ingress-gateway
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name:  secret-reader
  namespace: istio-ingress-gateway
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader
subjects:
  - kind: ServiceAccount
    name:  secret-reader
```
```
$ vi gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-ingress-gateway
  namespace: istio-ingress-gateway
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
      istio: ingress-gateway
  template:
    metadata:
      annotations:
        inject.istio.io/templates: gateway
      labels:
        app: istio-ingressgateway
        istio: ingress-gateway
        sidecar.istio.io/inject: "true"
    spec:
      containers:
        - name: istio-proxy
          image: auto
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 100m
              memory: 128Mi
          ports:
            - containerPort: 15021
              protocol: TCP
            - containerPort: 9000
              protocol: TCP
      serviceAccountName: secret-reader
```
```
$ vi gateway-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress-gateway
  labels:
    istio: ingress-gateway
spec:
  type: NodePort
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  ports:
    - name: status-port
      port: 15021
      targetPort: 15021
      protocol: TCP
    - name: tcp-9000
      port: 9000
      targetPort: 9000
      protocol: TCP
      nodePort: 31000
```
```
$ vi gateway-cr.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: tcp-ingress-gateway
  namespace: istio-ingress-gateway
spec:
  selector:
    istio: ingress-gateway
    app: istio-ingress-gateway
  servers:
  - port:
      number: 9000
      name: tcp
      protocol: TCP
    hosts:
    - "*"
```
```
$ oc apply -f gateway-sa.yaml
$ oc apply -f gateway-deployment.yaml
$ oc apply -f gateway-service.yaml
$ oc apply -f gateway-cr.yaml
```
```
$ oc get sa,role,rolebinding,po,svc,endpoint,gateway.networking.istio.io -n istio-ingress-gateway
$ oc get pods -n istio-ingress-gateway -l app=istio-ingressgateway --show-labels
```

# 5. TCP 앱 배포
```
$ oc new-project tcp-app-ns
$ oc label namespace tcp-app-ns istio-injection=enabled
```
```
$ vi tcp-app.yaml
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
```
```
$ tcp-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: tcp-echo-vs
  namespace: tcp-app-ns
spec:
  hosts:
  - "*"
  gateways:
  - istio-ingress-gateway/tcp-ingress-gateway
  tcp:
  - match:
    - port: 9000
    route:
    - destination:
        host: tcp-echo-service.tcp-app-ns.svc.cluster.local
        port:
          number: 9000
```
```
$ oc apply -f tcp-app.yaml
$ oc apply -f tcp-vs.yaml
```
```
$ oc get po,svc,virtualservice -n tcp-app-ns
$ oc get pods -n tcp-app-ns --show-labels
```
