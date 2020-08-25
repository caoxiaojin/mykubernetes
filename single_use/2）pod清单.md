####创建简单的pod资源
    
    [root@master02 ~]# kubectl explain pod.spec  查看字段
    [root@master01 single_user]# cat pod.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: app
        image: nginxlinux/myapp:v1

    [root@master01 single_user]# kubectl apply -f pod.yaml
    
####查看pod的描述和日志，便于拍错

    [root@master01 ~]# kubectl logs myapp-pod -c app
    [root@master01 ~]# kubectl describe pod myapp
