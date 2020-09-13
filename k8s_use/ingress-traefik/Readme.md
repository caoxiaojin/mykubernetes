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