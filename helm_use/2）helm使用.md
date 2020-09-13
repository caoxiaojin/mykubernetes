#### Helm 自定义模板
    创建文件夹
    mkdir test
    cd test
    
    创建描述文件 Chart.yaml,这个文件必须有name和version定义
    cat <<'EOF' > ./Chart.yaml
    name: hello-world
    version: 1.0.0
    EOF
    
    创建模板文件，用于生成 Kubernetes 资源清单（manifests）
    mkdir ./templates
    cat <<'EOF'> ./templates/deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-world
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hello-world
      template:
        metadata:
          labels:
            app: hello-world
        spec:
          containers:
          - name: hello-world
            image: nginxlinux/myapp:v1
            ports:
            - containerPort: 80
              protocol: TCP
    EOF
    
    cat <<'EOF'> ./templates/service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-world
    spec:
      type: NodePort
      selector:
        app: hello-world
      ports:
      - port: 80
        targetPort: 80
        protocol: TCP
    EOF
    
    部署应用
    [root@master01 test]# helm install .
    NAME:   elder-donkey
    LAST DEPLOYED: Sun Sep 13 22:05:08 2020
    NAMESPACE: default
    STATUS: DEPLOYED
    
    RESOURCES:
    ==> v1/Deployment
    NAME         READY  UP-TO-DATE  AVAILABLE  AGE
    hello-world  0/1    1           0          0s
    
    ==> v1/Pod(related)
    NAME                         READY  STATUS             RESTARTS  AGE
    hello-world-7d58b77cf-bhmgp  0/1    ContainerCreating  0         0s
    
    ==> v1/Service
    NAME         TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
    hello-world  NodePort  10.98.160.162  <none>       80:31518/TCP  0s
    
    [root@master01 test]# helm list
    NAME                	REVISION	UPDATED                 	STATUS  	CHART            	APP VERSION	NAMESPACE
    auxiliary-wildebeest	1       	Sun Sep 13 22:02:50 2020	FAILED  	hello-world-1.0.0	           	default  
    elder-donkey        	1       	Sun Sep 13 22:05:08 2020	DEPLOYED	hello-world-1.0.0	           	default  
    
    滚动更新
    [root@master01 test]# helm upgrade elder-donkey .
    Release "elder-donkey" has been upgraded.
    LAST DEPLOYED: Sun Sep 13 22:10:22 2020
    NAMESPACE: default
    STATUS: DEPLOYED
    
    RESOURCES:
    ==> v1/Deployment
    NAME         READY  UP-TO-DATE  AVAILABLE  AGE
    hello-world  1/1    1           1          5m14s
    
    ==> v1/Pod(related)
    NAME                         READY  STATUS   RESTARTS  AGE
    hello-world-7d58b77cf-bhmgp  1/1    Running  0         5m14s
    
    ==> v1/Service
    NAME         TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
    hello-world  NodePort  10.98.160.162  <none>       80:31518/TCP  5m14s
    
    查看历史版本
    [root@master01 test]# helm history elder-donkey
    REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
    1       	Sun Sep 13 22:05:08 2020	SUPERSEDED	hello-world-1.0.0	           	Install complete
    2       	Sun Sep 13 22:10:22 2020	SUPERSEDED	hello-world-1.0.0	           	Upgrade complete
    3       	Sun Sep 13 22:11:56 2020	DEPLOYED  	hello-world-1.0.0	           	Upgrade complete
    
    查看描述信息
    [root@master01 test]# helm status elder-donkey
    LAST DEPLOYED: Sun Sep 13 22:11:56 2020
    NAMESPACE: default
    STATUS: DEPLOYED
    
    RESOURCES:
    ==> v1/Deployment
    NAME         READY  UP-TO-DATE  AVAILABLE  AGE
    hello-world  1/1    1           1          13m
    
    ==> v1/Pod(related)
    NAME                         READY  STATUS   RESTARTS  AGE
    hello-world-7d58b77cf-bhmgp  1/1    Running  0         13m
    
    ==> v1/Service
    NAME         TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
    hello-world  NodePort  10.98.160.162  <none>       80:31518/TCP  13m
    
#### 将配置体现在配置文件中

    定义镜像的环境变量
    cat <<'EOF'> ./values.yaml
    image:
      repository: nginxlinux/myapp
      tag: '1.0'
    EOF
    
    deployment.yaml中
    image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
    
    在 values.yaml中的值可以被部署 release时用到的参数 --values YAML_FILE_PATH或 --set key1=value1,key2=value2 覆盖掉
    如： helm install --set image.tag='latest' .
    
    升级版本
    helm upgrade -f values.yaml test .