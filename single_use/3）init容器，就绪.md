####init容器特殊说明

    1）在pod启动过程中Init容器会按顺序在网络和数据卷初始化之后启动，每个容器必须在下一个容器启动之前成功退出
    2）如果由于运行时或失败退出，将导致容器启动失败，它会根据Pod的restartPolicy指定的策略进行重试。
    然而，如果Pod的restart设置为Always，Init容器失败时会使用RestartPolicy策略
    3）在所以的Init容器没有成功之前，Pod将不会变成Ready状态，Init容器的端口将不会在Service中聚集
     正在初始化中的Pod处于Pending状态，但应该会将Initializing状态设置为True
    4) 如果Pod重启，所以Init容器必须重新执行
    5）对Init容器spec的修改被限制在容器image字段，修改其他字段都不会生效。
     更改Init容器的image字段，等价于重启该Pod

#####init模板

    [root@master01 single_user]# cat init_busy.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: myapp-pod
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: busybox:v1
        command: ['sh', '-c','echo The app is running! && sleep 3600']
      initContainers:
      - name: init-myservice
        image: busybox:v1
        command: ['sh','-c','until nslookup myservice; do echo waiting for myservice;sleep 2;done;']
      - name: init-mydb
        image: busybox
        command: ['sh','-c','until nslookup mydb; do echo waiting for mydb; sleep 2;done;']
        
####查看pod信息。分析是否能启动

    [root@master01 ~]# kubectl get pods
    NAME        READY   STATUS     RESTARTS   AGE
    myapp-pod   0/1     Init:0/2   0          4m16s
    [root@master01 ~]# kubectl logs myapp-pod -c  init-myservice
    ......
    *** Can't find myservice.default.svc.cluster.local: No answer
    *** Can't find myservice.svc.cluster.local: No answer
    *** Can't find myservice.cluster.local: No answer
    
####创建pod启动的前置条件

    [root@master01 single_user]# cat mydb.yaml 
    kind: Service
    apiVersion: v1
    metadata:
      name: myservice
    spec:
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: mydb
    spec:
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9377

####此时查看pod

    [root@master01 ~]# kubectl get svc
    NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
    kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP           3d1h
    mydb               ClusterIP   10.107.199.13   <none>        80/TCP            7m9s
    myservice          ClusterIP   10.99.189.1     <none>        80/TCP            7m9s
    过1分钟查看
    [root@master01 ~]# kubectl get pods
    NAME        READY   STATUS    RESTARTS   AGE
    myapp-pod   1/1     Running   0          27m

    
    
