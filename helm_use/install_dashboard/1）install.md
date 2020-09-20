#### 从远程仓库安装dashboard

    更新仓库
    helm repo update  
    
    [root@master01 dashboard]# helm repo update
    Hang tight while we grab the latest from your chart repositories...
    ...Skip local chart repository
    ...Successfully got an update from the "stable" chart repository
    Update Complete.
    [root@master01 dashboard]# helm repo list
    NAME  	URL                                             
    stable	https://kubernetes-charts.storage.googleapis.com
    local 	http://127.0.0.1:8879/charts  
    
    下载官方dashboard仓库
    [root@master01 dashboard]# helm fetch stable/kubernetes-dashboard
    [root@master01 dashboard]# ls
    kubernetes-dashboard-1.11.1.tgz

####编辑相关配置文件

    [root@master01 kubernetes-dashboard]# cat kubernetes-dashboard.yaml 
    image:
      repository: k8s.gcr.io/kubernetes-dashboard-amd64
      tag: v1.10.1
    ingress:
      enabled: true
      hosts:
        - k8s.frognew.com
      annotations:
        nginx.ingress.kubernetes.io/ssl-redirect: "true"
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
      tls:
        - secretName: frognew-com-tls-secret
          hosts:
          - k8s.frognew.com
    rbac:
      clusterAdminRole: true
      
    执行安装
    第一种从官方进行安装
    helm install stable/kubernetes-dashboard \
    -n kubernetes-dashboard \
    --namespace kube-system \
    -f kubernetes-dashboard.yaml
    
    第二种从本地进行安装
    helm install . \
    -n kubernetes-dashboard \
    --namespace kube-system \
    -f kubernetes-dashboard.yaml
    
    [root@master01 ~]# kubectl get pod -n kube-system|grep kubernetes
    kubernetes-dashboard-bfdf5fc85-vz4xz   1/1     Running            0          6m23s
    [root@master01 ~]# kubectl get svc -n kube-system|grep kubernetes
    kubernetes-dashboard      ClusterIP   10.105.76.45    <none>        443/TCP                  9m42s
    如果需要访问
    kubectl edit svc kubernetes-dashboard -n kube-system
    修改
    type: ClusterIP
    改后
    type: NodePort
    
    [root@master01 ~]# kubectl get svc -n kube-system|grep kubernetes
    kubernetes-dashboard      NodePort    10.105.76.45    <none>        443:31445/TCP            12m
    
    获取token
    [root@master01 ~]# kubectl get secret -n kube-system|grep kubernetes-dashboard-token
    kubernetes-dashboard-token-4f7fx                 kubernetes.io/service-account-token   3      25m
 
    kubectl describe secret -n kube-system kubernetes-dashboard-token-4f7fx  
    
    
    访问：https://192.168.37.11:31445/
    thisisunsafe
    
    