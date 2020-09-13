####Secret有三种类型

    Service Account: 用来访问Kubernetes Api，由Kubernetes自动创建，
      并且会自动挂载到Pod的 /run/secrets/kubernetes.io/serviceaccount
    Opaque: base64编码格式的Secret，用来存储密码，秘钥等
    kubernetes.io/dockerconfigjson:用来存储私有docker registry的认证信息
    
#### Service Account
    service Account用来访问Kubernetes Api,由kubernetes自动创建，
     并且会挂载Pod的/run/secrets/kubernetes.io/serviceaccount 中
     
    [root@master01 ~]# kubectl get pods -n kube-system|grep proxy
    kube-proxy-c8xkm                   1/1     Running   8          13d
    kube-proxy-jrtpw                   1/1     Running   7          13d
    kube-proxy-n68dq                   1/1     Running   8          12d
    kube-proxy-snmpd                   1/1     Running   8          13d
    [root@master01 ~]# kubectl exec -it kube-proxy-c8xkm  -- /bin/sh
    Error from server (NotFound): pods "kube-proxy-c8xkm" not found
    [root@master01 ~]# kubectl exec -it -n kube-system kube-proxy-c8xkm  -- /bin/sh
    # cd /run/secrets/kubernetes.io/serviceaccount
    # ls
    ca.crt	namespace  token
    # cat token
    eyJhbGciOiJSUzI1NiIsImtpZCI6IktTUENOcDFBR2hialFvT3hQanljNkpST0RGejBRbWJnTk1HUERkZ0Fac1UifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlLXByb3h5LXRva2VuLW5tZnNzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6Imt1YmUtcHJveHkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YWI2ZTg4MS1kOGRkLTRmNGYtOWJmNy1mYjkwZDAzYmU2MGQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06a3ViZS1wcm94eSJ9.EJX95PWfRvVYgqPz_k4e9E1VRhbEb_Hf2vaNG6vHV9P1WFNV2ZJKCreetmZKsIkjsDeRvlrHQFlbCHWdUZC8IWtTf5J7ebPJgMfYqxP4CtB54w50W-Vl_7IbgCmvX8L0Bt7kbFuME-PnxML30SYGo0qYa6meVM0wNlk0bo56HascbOQJFBpc60fEbG7nz0R8a09Qth8Jusm7PDpIdzj3MOSxUaLv33vWRVUsTOxtJ1dYWDu6DNByx8FjYZ5g4YJCZc6TkTzGK1KqIHcKJa2_3sFIQoaQXIPuDQWnBQiNl1sBaO2RVKyVRaEgJC3zabTvyNvuiRrSZd6bQXsYOBtm9A#  
    
#### Opaque Secret
    1）创建说明Opaque类型的数据是一个map类型，要求value是base64编码格式
    加密
    [root@master01 ~]# echo -n "admin" | base64
    YWRtaW4=
    [root@master01 ~]# echo -n "1f2d1e2e67df" | base64
    MWYyZDFlMmU2N2Rm
    
    解密
    [root@master01 ~]# echo -n "MWYyZDFlMmU2N2Rm" | base64 -d
    1f2d1e2e67df[root@master01 ~]# 
    
    [root@master01 k8s_use]# cat sec.yaml 
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      password: MWYyZDFlMmU2N2Rm
      username: YWRtaW4=
      
    [root@master01 k8s_use]# kubectl apply -f sec.yaml 
    secret/mysecret created
    [root@master01 k8s_use]# kubectl get secret
    NAME                  TYPE                                  DATA   AGE
    default-token-lszjd   kubernetes.io/service-account-token   3      13d
    mysecret              Opaque   
    
    [root@master01 k8s_use]# cat pod_sec.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        name: seret-test
      name: seret-test
    spec:
      volumes:
      - name: secrets
        secret:
          secretName: mysecret
      containers:
      - image: nginxlinux/myapp:v1
        name: db
        volumeMounts:
        - name: secrets
          mountPath: "/etc/secrets"
          readOnly: true
    [root@master01 ~]# kubectl get pod|grep seret-test
    seret-test                  1/1     Running   0          60s
    [root@master01 ~]# kubectl exec -it seret-test -- /bin/sh
    / # cd /etc/secrets
    /etc/secrets # ls
    password  username
    /etc/secrets # cat username 
    admin/etc/secrets # 
    /etc/secrets # cat password 
    1f2d1e2e67df/etc/secrets # 
          
    和 deployment 结合使用     
    [root@master01 k8s_use]# cat deploy_sec.yaml 
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: pod-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: pod-deployment
      template:
        metadata:
          labels:
            app: pod-deployment
        spec:
          containers:
          - name: pod-1
            image: nginxlinux/myapp:v1
            imagePullPolicy: IfNotPresent
            ports:
            - containerPort: 80
            env: 
            - name: TEST_USER
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: username
            - name: TEST_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: password     
          
    [root@master01 k8s_use]# kubectl apply -f deploy_sec.yaml 
    deployment.apps/pod-deployment created
    [root@master01 k8s_use]# 
    [root@master01 k8s_use]# kubectl get pods
    NAME                              READY   STATUS    RESTARTS   AGE
    pod-deployment-86c776c58f-d8wss   1/1     Running   0          7s
    [root@master01 k8s_use]# kubectl exec -it pod-deployment-86c776c58f-d8wss -- /bin/sh
    / # echo $TEST_USER
    admin
    / # echo $TEST_PASSWORD
    1f2d1e2e67df
      
#### kubernetes.io/dockerconfigjson  私有仓库认证
    使用kubectl创建docker registry认证的secret      
    
    [root@master01 ~]# kubectl create secret docker-registry myregistrykey --docker-server=nginxlinux.com --docker-username=admin --docker-password=Harbor12345 --docker-email=mylinux@163.com
    secret/myregistrykey created
    
    [root@master01 reg]# cat myregistrykey.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: foo
    spec:
      containers:
        - name: foo
          image: nginxlinux.com/myapp:v1
        imagePullSecrets:
          - name: myregistrykey