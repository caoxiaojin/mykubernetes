#### traefik 使用
    官方网站：https://docs.traefik.io/user-guides/crd-acme/
    下面使用的配置来着官方网站
    
    [root@master01 traefik]# kubectl apply -f traefik-res.yaml 
    customresourcedefinition.apiextensions.k8s.io/ingressroutes.traefik.containo.us created
    customresourcedefinition.apiextensions.k8s.io/middlewares.traefik.containo.us created
    customresourcedefinition.apiextensions.k8s.io/ingressroutetcps.traefik.containo.us created
    customresourcedefinition.apiextensions.k8s.io/ingressrouteudps.traefik.containo.us created
    customresourcedefinition.apiextensions.k8s.io/tlsoptions.traefik.containo.us created
    customresourcedefinition.apiextensions.k8s.io/tlsstores.traefik.containo.us created
    customresourcedefinition.apiextensions.k8s.io/traefikservices.traefik.containo.us created
    clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
    clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
    [root@master01 traefik]# kubectl apply -f traefik-deploy.yaml 
    serviceaccount/traefik-ingress-controller created
    deployment.apps/traefik created
    [root@master01 traefik]# kubectl apply -f traefik-svc.yaml 
    service/traefik created
    [root@master01 traefik]# kubectl get svc|grep traefik
    traefik      ClusterIP   10.103.160.187   <none>        8000/TCP,8080/TCP,4443/TCP   32s
    [root@master01 traefik]# kubectl get pod|grep traefik
    traefik-97577cc87-s6sjs   1/1     Running   0          49s
    [root@master01 traefik]# kubectl get deployment
    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    traefik   1/1     1            1           2m4s

#### 将svc改为NodePort暴露端口验证

    apiVersion: v1
    kind: Service
    metadata:
      name: traefik
    
    spec:
      type: NodePort
      ports:
        - protocol: TCP
          name: web
          port: 8000
        - protocol: TCP
          name: admin
          port: 8080
        - protocol: TCP
          name: websecure
          port: 4443
      selector:
        app: traefik


    [root@master01 traefik]# kubectl apply -f traefik-svc_nodeport.yaml 
    service/traefik configured
    [root@master01 traefik]# kubectl get svc|grep traefik
    traefik      NodePort    10.103.160.187   <none>        8000:31666/TCP,8080:30104/TCP,4443:31117/TCP   4m38s

    访问 traefik 的管理界面
    http://192.168.37.12:30104
    
    访问demo服务
    curl test-demo.hungtcs.test:31666
    
#### 通过nginx配置域名访问
1）named配置域名
[root@harbor01 stream.d]# cat /var/named/eniot.io.zone
$ORIGIN eniot.io.
$TTL 600        ; 10 minutes;
@    IN SOA    dns.eniot.io.  dnsadmin.eniot.io. (
                    2020050401  ; serial
                    10800        ; refresh (3 hours)
                    900        ; retry  (15 minutes)
                    604800        ; expire (1 week)
                    86400       ; minimum (1 day)
                    )
            NS    dns.eniot.io.
$TTL 60 ; 1 minutes
dns                A   192.168.37.100
...........
*.k8s-test              A   192.168.37.100

2）修改traefix服务调为nodeport形式
apiVersion: v1
kind: Service
metadata:
  name: traefik

spec:
  type: NodePort
  ports:
    - protocol: TCP
      name: web
      port: 8000
      nodePort: 30800
    - protocol: TCP
      name: admin
      port: 8080
      nodePort: 30080
    - protocol: TCP
      name: websecure
      port: 4443
      nodePort: 30443
  selector:
    app: traefik
[root@master01 ~]# kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                        AGE
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP                                        14h
test-demo-service   ClusterIP   10.100.148.81   <none>        80/TCP                                         3h33m
traefik             NodePort    10.102.62.15    <none>        8000:30800/TCP,8080:30080/TCP,4443:30443/TCP   124m

3）测试服务
[root@master01 ~]# cat deployment_demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-demo
  labels:
    app: test-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-demo
  template:
    metadata:
      labels:
        app: test-demo
    spec:
      containers:
        - name: test-demo-app
          image: harbor-test.eniot.io/enos/myapp:v1
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "1000m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: test-demo-service
spec:
  selector:
    app: test-demo
  ports:
  - name: web
    port: 80
    protocol: TCP
[root@master01 ~]# cat deployment_demo_svc.yaml 
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: test-demo-service-ingress-route
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`myapp.k8s-test.eniot.io`)
    kind: Rule
    services:
    - name: test-demo-service
      port: 80
假设：myapp.k8s-test.eniot.io 域名绑定在了k8s机器上面，可直接 curl myapp.k8s-test.eniot.io:30800进行访问
   
4) 配置nginx  实现通配服访问
[root@harbor01 stream.d]# cat api.conf 
upstream apaas-api {
        server  master01.eniot.io:30800 max_fails=3 fail_timeout=60s weight=9;
        server  node01.eniot.io:30800 max_fails=3 fail_timeout=60s weight=9;
        server  node02.eniot.io:30800 max_fails=3 fail_timeout=60s weight=9;
}

server {
      listen       8000;
      proxy_connect_timeout 3s;
      proxy_timeout 30s;
      proxy_pass   apaas-api;
}

curl myapp.k8s-test.eniot.io:8000