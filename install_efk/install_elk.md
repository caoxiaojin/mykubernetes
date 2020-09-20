####helm部署elk

####1) 添加Google incubator仓库
    [root@master01 efk]# helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
    "incubator" has been added to your repositories

####2）部署Elasticsearch

    [root@master01 efk]# kubectl create namespace efk
    namespace/efk created
    
    [root@master01 efk]# helm fetch incubator/elasticsearch
    [root@master01 efk]# ls
    elasticsearch-1.10.2.tgz
    [root@master01 efk]# tar xf elasticsearch-1.10.2.tgz
    [root@master01 efk]# cd elasticsearch
    
    配置文件 values.yaml 
    MINIMUM_MASTER_NODES  最小节点数，测试可改为1
    
    客户端副本数量改为1
    client:
      name: client
      replicas: 1
    
    master副本数也改为1,由于没有存储卷，enabled: false
    master:
      name: master
      exposeHttp: false
      replicas: 1
      heapSize: "512m"
      persistence:
        enabled: false
    
    data节点也进行修改
    data:
      name: data
      exposeHttp: false
      replicas: 1
      heapSize: "1536m"
      persistence:
        enabled: false
    
    
    helm install --name els1 --namespace=efk -f values.yaml incubator/elasticsearch
    本地安装。备注由于kubernetes版本的问题，需要改相关的api接口，添加selector字段
    helm install --name els1 --namespace=efk -f values.yaml .  
    
    [root@master01 ~]# kubectl top nodes
    NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
    master01   236m         11%    1285Mi          35%       
    master02   149m         7%     844Mi           23%       
    master03   261m         13%    1297Mi          35%       
    node01     1346m        67%    3604Mi          98%       
    [root@master01 ~]# kubectl get pods -n efk
    NAME                                         READY   STATUS    RESTARTS   AGE
    els1-elasticsearch-client-6bf749ccfc-9s9df   1/1     Running   1          12m
    els1-elasticsearch-data-0                    0/1     Pending   0          3m6s
    els1-elasticsearch-master-0                  1/1     Running   1          12m

####3) 部署 Fluentd
    helm fetch stable/fluentd-elasticsearch
    [root@master01 efk]# ls fluentd-elasticsearch-2.0.7.tgz 
    fluentd-elasticsearch-2.0.7.tgz
    [root@master01 efk]# tar xf fluentd-elasticsearch-2.0.7.tgz
    vim values.yaml
    # 更改其中 elasticsearch 的访问地址
    helm install --name flu1 --namespace=efk -f value.yaml .

####4）部署kibana

备注 Elasticsearch 和 kibana 版本一定要一致
[root@master01 efk]# helm fetch stable/kibana --version 0.14.8
vim values.yaml
更改 elasticsearch.url 的地址