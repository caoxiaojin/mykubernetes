####Rrometheus的安装使用

    Rrometheus github地址：https://github.com/coreos/kube-prometheus
    
    组件说明
    1）MetricsServer: 是kubernetes集群资源使用情况的聚合器，收集数据给kubernetes集群内使用，如kubectl,hpa.scheduler等
    2）PrometheusOperator：是一个系统监测和警报工具箱，用来存储监控数据
    3）NodeExporter：用于各node的关键度量指标状态数据
    4）KubeStateMetrics：收集kubernetes集群内资源对象数据，制定告警规则
    5）Prometheus：采用pull方式收集apiserver，scheduler,controller-manager,kubelet组件数据，数据http协议传输
    6）Grafana：是可视化数据统计和监控平台
    
    构建记录
    git clone https://github.com/prometheus-operator/kube-prometheus.git
    cd kube-prometheus/manifests/
    
    修改文件 grafana-service.yaml，prometheus-service.yaml，alertmanager-service.yaml，为了测试和验证访问
    修改前
    [root@master01 manifests]# cat grafana-service.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: grafana
      name: grafana
      namespace: monitoring
    spec:
      ports:
      - name: http
        port: 3000
        targetPort: http
      selector:
        app: grafana
    修改后
    [root@master01 manifests]# cat grafana-service.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: grafana
      name: grafana
      namespace: monitoring
    spec:
      type: NodePort
      ports:
      - name: http
        port: 3000
        targetPort: http
        nodePort: 30100
      selector:
        app: grafana
    
    [root@master01 manifests]# cat prometheus-service.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        prometheus: k8s
      name: prometheus-k8s
      namespace: monitoring
    spec:
      type: NodePort
      ports:
      - name: web
        port: 9090
        targetPort: web
        nodePort: 30020
      selector:
        app: prometheus
        prometheus: k8s
      sessionAffinity: ClientIP
    
    [root@master01 manifests]# cat alertmanager-service.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        alertmanager: main
      name: alertmanager-main
      namespace: monitoring
    spec:
      type: NodePort
      ports:
      - name: web
        port: 9093
        targetPort: web
        nodePort: 30300
      selector:
        alertmanager: main
        app: alertmanager
      sessionAffinity: ClientIP
        
    需提前创建对应的名称空间
    [root@master01 ~]# cat monitoring_namespaces.yaml 
    apiVersion: v1
    kind: Namespace
    metadata:
      name: monitoring
      labels:
        name: monitoring
    [root@master01 manifests]# kubectl apply -f setup/    
    [root@master01 manifests]# kubectl apply -f ../manifests/
    
    查看所以的pod是否运行成功  
    [root@master01 manifests]# kubectl get pods -n monitoring -o wide
    NAME                                   READY   STATUS    RESTARTS   AGE     IP              NODE       NOMINATED NODE   READINESS GATES
    alertmanager-main-0                    2/2     Running   0          8m43s   10.244.3.142    node01     <none>           <none>
    alertmanager-main-1                    2/2     Running   0          8m17s   10.244.3.144    node01     <none>           <none>
    alertmanager-main-2                    2/2     Running   0          13m     10.244.3.138    node01     <none>           <none>
    grafana-85c89999cb-nbc5s               1/1     Running   1          50m     10.244.3.135    node01     <none>           <none>
    kube-state-metrics-6b7567c4c7-cc42z    3/3     Running   3          40m     10.244.3.133    node01     <none>           <none>
    node-exporter-dn9hf                    2/2     Running   0          66m     192.168.37.21   node01     <none>           <none>
    node-exporter-sms6g                    2/2     Running   0          27m     192.168.37.12   master02   <none>           <none>
    node-exporter-swdlv                    2/2     Running   0          30m     192.168.37.11   master01   <none>           <none>
    node-exporter-vmn52                    2/2     Running   0          66m     192.168.37.13   master03   <none>           <none>
    prometheus-adapter-b8d458474-6djfr     1/1     Running   1          48m     10.244.3.134    node01     <none>           <none>
    prometheus-k8s-0                       3/3     Running   0          13m     10.244.3.140    node01     <none>           <none>
    prometheus-k8s-1                       3/3     Running   1          29s     10.244.3.145    node01     <none>           <none>
    prometheus-operator-55cb794976-mdwtz   2/2     Running   0          15m     10.244.3.136    node01     <none>           <none>
    
    查看top命令是否可使用
    [root@master01 manifests]# kubectl top nodes
    NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
    master01   297m         14%    1325Mi          36%       
    master02   84m          4%     810Mi           22%       
    master03   299m         14%    1410Mi          38%       
    node01     155m         7%     1843Mi          50% 
        
        
    [root@master01 ~]# kubectl get svc -n monitoring
    NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    alertmanager-main       NodePort    10.106.153.28   <none>        9093:30300/TCP               74m
    alertmanager-operated   ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   21m
    grafana                 NodePort    10.99.188.194   <none>        3000:30100/TCP               74m
    kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP            74m
    node-exporter           ClusterIP   None            <none>        9100/TCP                     74m
    prometheus-adapter      ClusterIP   10.100.185.29   <none>        443/TCP                      74m
    prometheus-k8s          NodePort    10.105.161.79   <none>        9090:30020/TCP               74m
    prometheus-operated     ClusterIP   None            <none>        9090/TCP                     21m
    prometheus-operator     ClusterIP   None            <none>        8443/TCP                     23m
        
    prometheus页面访问 http://masterIP:30020
    Grafana页面访问 http://192.168.37.11:30100/
    默认密码：admin/admin
    添加默认的监控的数据源

#### 对资源进行自动伸缩扩容

    [root@master01 ~]# kubectl run php-apache --image=gcr.io/google_containers/hpa-example:v1 --requests=cpu=200m --expose --port=80
    kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
    service/php-apache created
    deployment.apps/php-apache created
    [root@master01 ~]# kubectl get deployment|grep php-apache
    php-apache                            1/1     1            1           51s
    
    创建HPA控制器
    [root@master01 ~]# kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
    horizontalpodautoscaler.autoscaling/php-apache autoscaled
    [root@master01 ~]# kubectl get hpa
    NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
    php-apache   Deployment/php-apache   <unknown>/50%   1         10        0          15s
    
    并观察pods
    [root@master01 ~]# kubectl top pods php-apache-5975898c4d-dgwng
    NAME                          CPU(cores)   MEMORY(bytes)   
    php-apache-5975898c4d-dgwng   0m           15Mi  
    
    无限循环访问增加负载
    kubectl run -i --tty load-generator --image=busybox:v1 /bin/sh
    while true ; do wget -q -O- http://php-apache.default.svc.cluster.local; done
    
    此时
    [root@master01 ~]# kubectl get deployments|grep php-apache
    php-apache                            3/10    10           3           19m
    [root@master01 ~]# 
    [root@master01 ~]# kubectl get hpa
    NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
    php-apache   Deployment/php-apache   159%/50%   1         10        10         12m

#### 对pod进行资源限制

    kubernetes对资源的限制，使用resources的requests和limits来实现
    比如
    
    spec:
        containers:
        - image: xxx
          imagePullPolicy: Always
          name: auth
          ports:
          - containerPort: 8080
            protocol: TCP
          resources:
            limits:
              cpu: "4"
              memory: 2Gi
            requests:
              cpu: 250m
              memory: 250Mi
              
#### 对名称空间的资源限制

    1） 计算资源配额
    
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-resources
      namespace: spark-cluster
    spec:
      hard:
        pods: "20"
        requests.cpu: "20"
        requests.memory: 100Gi
        limits.cpu: "40"
        limits.memory: 200Gi
        
    2）配置对象数量配置限制
    
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: object-counts
      namespace: spark-cluster
    spec:
      hard:
        configmaps: "10"
        persistentvolumeclaims: "4"
        replicationcontrollers: "20"
        secrets: "10"
        services: "10"
        services.loadbalancers: "2"
    
    3）如果pod没有使用限制，可在名称空间上面对pod进行限制
    
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: mem-limit-range
    spec:
      limits:
      - default:
          memory: 50Gi
          cpu: 5
        defaultRequest:
          memory: 1Gi
          cpu: 1
        type: Container
    
              