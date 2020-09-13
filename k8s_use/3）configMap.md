#### configMap

    许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。configMap API给我们提供了向容器中注入配置信息的机制。
    configMap可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制大对象
##### configMap的创建方式
##### 1）使用目录创建
    [root@master01 ~]# ls docs/user-guide/configmap/kubectl/
    game.properties  ui.properties
    [root@master01 ~]# cat docs/user-guide/configmap/kubectl/game.properties 
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.codo.allowed=true
    secret.code.lives=30
    [root@master01 ~]# cat docs/user-guide/configmap/kubectl/ui.properties 
    color.good=purple
    color.bad=yellow
    color.textmode=true
    how.nice.to.look=fairlyNice
    
    [root@master01 ~]# kubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl
    configmap/game-config created
    root@master01 ~]# kubectl get cm
    NAME          DATA   AGE
    game-config   2      25s
    [root@master01 ~]# 
    [root@master01 ~]# kubectl get cm game-config
    NAME          DATA   AGE
    game-config   2      47s
    
    查看配置信息
    [root@master01 ~]# kubectl get cm game-config -o yaml
    apiVersion: v1
    data:
      game.properties: |
        enemies=aliens
        lives=3
        enemies.cheat=true
        enemies.cheat.level=noGoodRotten
        secret.code.passphrase=UUDDLRLRBABAS
        secret.codo.allowed=true
        secret.code.lives=30
      ui.properties: |
        color.good=purple
        color.bad=yellow
        color.textmode=true
        how.nice.to.look=fairlyNice
    kind: ConfigMap
    metadata:
      creationTimestamp: "2020-08-30T16:41:05Z"
      name: game-config
      namespace: default
      resourceVersion: "295318"
      selfLink: /api/v1/namespaces/default/configmaps/game-config
      uid: 0e18d2e1-2361-4bb2-862f-8b36c743f41d
    
    [root@master01 ~]# kubectl describe cm game-config
    Name:         game-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    
    Data
    ====
    game.properties:
    ----
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.codo.allowed=true
    secret.code.lives=30
    
    ui.properties:
    ----
    color.good=purple
    color.bad=yellow
    color.textmode=true
    how.nice.to.look=fairlyNice

##### 2）使用文件创建
    
    备注：和目录没啥区别
    [root@master01 ~]# kubectl create configmap game-config-2 --from-file=docs/user-guide/configmap/kubectl/game.properties 
    configmap/game-config-2 created
    [root@master01 ~]# kubectl get cm
    NAME            DATA   AGE
    game-config     2      6m38s
    game-config-2   1      5s
    
##### 3）使用字面值创建

    使用文字值创建，利用 --from-literal参数传递配置信息，
    [root@master01 ~]# kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
    configmap/special-config created
    [root@master01 ~]# kubectl describe configmap special-config
    Name:         special-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    
    Data
    ====
    special.how:
    ----
    very
    special.type:
    ----
    charm
    Events:  <none>



#### pod中使用ConfigMap来替代环境变量

    [root@master01 env]# cat env.yaml 
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: env-config
      namespace: default
    data:
      log_level: INFO
    
    [root@master01 env]# kubectl apply -f env.yaml 
    configmap/env-config created
    [root@master01 env]# kubectl get cm
    NAME             DATA   AGE
    env-config       1      5s
    game-config      2      23h
    game-config-2    1      23h
    special-config   2      23h
    
    [root@master01 single_user]# cat pod_env.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: dapi-test-pod
    spec:
      containers:
      - name: test-container
        image: nginxlinux/myapp:v1
        command: ["/bin/sh","-c","env"]
        env:
          - name: SPECIAL_LEVEL_KEY
            valueFrom:
              configMapKeyRef:
                name: special-config
                key: special.how
          - name: SPECIAL_TYPE_KEY
            valueFrom: 
              configMapKeyRef:
                name: special-config
                key: special.type
        envFrom:
          - configMapRef:
              name: env-config
      restartPolicy: Never
    
    [root@master01 single_user]# kubectl logs pod/dapi-test-pod
    MYAPP_SVC_PORT_80_TCP_ADDR=10.98.57.156
    KUBERNETES_PORT=tcp://10.96.0.1:443
    KUBERNETES_SERVICE_PORT=443
    MYAPP_SVC_PORT_80_TCP_PORT=80
    HOSTNAME=dapi-test-pod
    SHLVL=1
    MYAPP_SVC_PORT_80_TCP_PROTO=tcp
    HOME=/root
    SPECIAL_TYPE_KEY=charm
    MYAPP_SVC_PORT_80_TCP=tcp://10.98.57.156:80
    NGINX_VERSION=1.12.2
    KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    KUBERNETES_PORT_443_TCP_PORT=443
    KUBERNETES_PORT_443_TCP_PROTO=tcp
    MYAPP_SVC_SERVICE_HOST=10.98.57.156
    SPECIAL_LEVEL_KEY=very
    log_level=INFO
    KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
    KUBERNETES_SERVICE_PORT_HTTPS=443
    PWD=/
    KUBERNETES_SERVICE_HOST=10.96.0.1
    MYAPP_SVC_SERVICE_PORT=80
    MYAPP_SVC_PORT=tcp://10.98.57.156:80

####通过数据卷插件使用ConfigMap

    [root@master01 env]# cat  special-config2.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: special-config2
      namespace: default
    data:
      special.how: very
      special.type: charm
      
    [root@master01 single_user]# cat pod_env2.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: dapi-test-pod2
    spec:
      containers:
        - name: test-container
          image: nginxlinux/myapp:v1
          command: ["/bin/sh","-c","sleep 600"]
          volumeMounts:
          - name: config-volume
            mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: special-config2
      restartPolicy: Never

    查看存储
    [root@master01 single_user]# kubectl get pod|grep dapi-test-pod2
    dapi-test-pod2   1/1     Running     0          17s
    [root@master01 single_user]# kubectl exec dapi-test-pod2 -it -- /bin/sh
    / # cd /etc/config/
    /etc/config # ls
    special.how   special.type
    /etc/config # cat special.how 
    very/etc/config # 
    /etc/config # cat special.type 
    charm/etc/config # 

#### ConfigMap的热更新

    [root@master01 k8s_use]# cat deploy_cofigmap.yaml 
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: log-config
    data:
      log_level: INFO
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: my-nginx
      template:
        metadata:
          labels:
            app: my-nginx
        spec:
          containers:
          - name: my-nginx
            image: nginxlinux/myapp:v1
            ports:
            - containerPort: 80
            volumeMounts:
            - name: config-volume
              mountPath: /etc/config
          volumes:
            - name: config-volume
              configMap:
                name: log-config
    [root@master01 k8s_use]# kubectl get pods
    NAME                        READY   STATUS    RESTARTS   AGE
    my-nginx-6ff7b86dcd-gnsg2   1/1     Running   0          2m57s
    [root@master01 k8s_use]# kubectl exec my-nginx-6ff7b86dcd-gnsg2 -it cat /etc/config/log_level
    INFO[root@master01 k8s_use]# 
    
    修改config内容，INFO改为DEBUG
    [root@master01 ~]# kubectl edit configmap log-config
    configmap/log-config edited
    等待大约10秒钟查看配置
    [root@master01 ~]# kubectl exec my-nginx-6ff7b86dcd-gnsg2 -it cat /etc/config/log_level
    DEBUG[root@master01 ~]# 
    备注： configMap如果以ENV的方式挂载至容器，修改configMap并不会实现热更新
    
#### ConfigMap更新后滚动更新pod

    更新ConfigMap目前并不会触发相关Pod的滚动更新，可以通过修改pod annotations的方式强制触发滚动更新
    # kubectl patch deployment my-nginx --patch '{"spec":{"template":{"metadata":{"annotations":{"version/config":"2020411"}}}}}'
    这个示例在 .spec.template.metadata.annotations中添加 version/config。每次通过修改version/config来触发滚动更新
    
    更新ConfigMap后：
    使用该ConfigMap挂载的Env不会同步更新
    使用该ConfigMap挂载的Volume中的数据需要一段时间（实测大概10秒）才能同步更新