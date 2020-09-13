####emptyDir的用法有
    1）暂存空间，例如用于基于磁盘的合并排序
    2）用作长时间计算奔溃恢复时的检查点
    3）web服务器容器提供数据时，保存内容管理器容器提取的文件
    
    [root@master01 k8s_use]# cat pod_volume.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pd
    spec:
      containers:
      - image: nginxlinux/myapp:v1
        name: test-container
        volumeMounts:
        - mountPath: /cache
          name: cache-volume
      - name: liveness-exec-container
        image: busybox:v1
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh","-c","sleep 6000s"]
        volumeMounts:
        - mountPath: /test
          name: cache-volume
      volumes:
      - name: cache-volume
        emptyDir: {}

    [root@master01 k8s_use]# kubectl apply -f pod_volume.yaml 
    pod/test-pd created
    [root@master01 k8s_use]# kubectl get pods
    NAME      READY   STATUS    RESTARTS   AGE
    test-pd   2/2     Running   0          6s
    [root@master01 k8s_use]# kubectl exec test-pd -c test-container -it /bin/sh
    / # cd /cache/
    /cache # 
    /cache # date > index.html
    /cache # cat index.html 
    Sat Sep  5 19:02:01 UTC 2020
    /cache # exit
    [root@master01 k8s_use]# 
    [root@master01 k8s_use]# kubectl exec test-pd -c liveness-exec-container -it /bin/sh
    / # cd test/
    /test # ls
    index.html
    /test # cat index.html 
    Sat Sep  5 19:02:01 UTC 2020
    /test # exit

#### hostPath，了解

    hostPath：将主机节点的文件系统中的文件或目录挂载到集群中
    运行需要docker内容的容器，使用 /var/lib/docker 的 hostPath
    在容器中运行 cAdvisor;使用 /dev/cgroups的hostPath
    
    [root@master01 k8s_use]# cat pod_volume.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pd
    spec:
      containers:
      - image: nginxlinux/myapp:v1
        name: test-container
        volumeMounts:
        - mountPath: /test-pd
          name: test-volume
      volumes:
      - name: test-volume
        hostPath:
          # directory location on host
          path: /data
          # this field is optional
          type: Directory
    
    [root@master01 k8s_use]# kubectl apply -f  pod_volume.yaml 
    pod/test-pd created
    [root@master01 k8s_use]# 
    [root@master01 k8s_use]# kubectl exec -it test-pd -- /bin/sh
    / # cd /test-pd
    /test-pd # date > index.html
    /test-pd # cat index.html 
    Sat Sep  5 19:28:44 UTC 2020
    Sun Sep  6 03:29:06 CST 2020
    备注：该存储，仅仅在对应的node节点上，才进行了相应的存储
    
    
    
#### PV-PVC的概念
    PersistentVolumeClaim(PVC)
    是用户存储的请求，它与Pod相似。Pod消耗节点资源,PVC消耗PV资源，Pod可以请求特定级别的资源(cpu和内存)。
    声明可以请求特定的大小和访问模式（例如，可以以读/写一次或只读多次模式挂载）
    
    静态PV
    集群管理员创建一些PV。它们带有可提供群集用户使用的实际存储的细节。
    他们存在于kubernetes API中，可用于消费

    PV访问模式
    ReadWriteOnce  该卷可以被单个节点以读/写模式挂载
    ReadOnlyMany   该卷可以被多个节点以只读模式挂载
    ReadWriteMany  该卷可以被多个节点以读/写模式挂载
    在命令行中。访问模式缩写为：
    RWO - ReadWriteOnce
    ROX - ReadOnlyMany
    RWX - ReadWriteMany
    
    回收策略
    Retain(保留) -- 手动回收
    Delete(删除) -- 关联的存储资产将被删除
    
    状态
    Available（可用） -- 一块空闲资源还没有被任何声明绑定
    Bound（已绑定） -- 卷已经被声明绑定
    Released（已释放） -- 声明被删除，但是资源还未被集群重新声明
    Failed（失效）-- 该卷的自动回收失效


####持久化演示 NFS

#####安装NFS服务器

    yum install -y nfs-common nfs-utils rpcbind
    mkdir /nfs
    chmod 777 /nfs/
    chown nfsnobody /nfs/
    cat  /etc/exports
      /nfs *(rw,no_root_squash,no_all_squash,sync)
    systemctl start rpcbind
    systemctl start nfs
    systemctl enable nfs
    systemctl enable rpcbind
    
    所有节点进行客户端
    yum install -y nfs-utils rpcbind
    
    测试
    [root@master01 ~]# mkdir /test
    [root@master01 ~]# showmount -e 192.168.37.12
    Export list for 192.168.37.12:
    /nfs *
    [root@master01 ~]# mount -t nfs 192.168.37.12:/nfs /test/
    [root@master01 test]# date > 1.html
    [root@master01 test]# cd
    [root@master01 ~]# umount /test/
    [root@master01 ~]# rm -rf /test/
    
#####演示测试    
    [root@master01 pv]# cat pv.yaml 
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfspv1
    spec:
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: nfs
      nfs:
        path: /nfs
        server: 192.168.37.12
    [root@master01 pv]# 
    [root@master01 pv]# kubectl create -f pv.yaml 
    persistentvolume/nfspv1 created
    [root@master01 pv]# kubectl get pv
    NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
    nfspv1   10Gi       RWO            Retain           Available           nfs                     14s
        
    [root@master02 ~]# cat  /etc/exports
    /nfs *(rw,no_root_squash,no_all_squash,sync)
    /nfs1 *(rw,no_root_squash,no_all_squash,sync)
    /nfs2 *(rw,no_root_squash,no_all_squash,sync)
    /nfs3 *(rw,no_root_squash,no_all_squash,sync)
    [root@master02 ~]# mkdir /nfs{1..3}
    [root@master02 /]# chmod 777 nfs1/ nfs2/ nfs3/    
    [root@master02 /]# chown nfsnobody nfs1/ nfs2/ nfs3/
    [root@master02 /]# systemctl restart rpcbind
    [root@master02 /]# systemctl restart nfs
    
    
    [root@master01 pv]# cat pod.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      ports:
      - port: 80
        name: web
      clusterIP: None
      selector:
        app: nginx
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: web
    spec:
      selector:
        matchLabels:
          app: nginx
      serviceName: "nginx"
      replicas: 1
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginxlinux/myapp:v1
            ports:
            - containerPort: 80
              name: web
            volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
      volumeClaimTemplates:
      - metadata:
          name: www
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: "nfs"
          resources:
            requests:
              storage: 1Gi
    [root@master01 pv]# kubectl apply  -f pod.yaml 
    service/nginx created
    statefulset.apps/web created
    
    内部快速解析
    [root@master01 pv]# kubectl get pods
    NAME      READY   STATUS    RESTARTS   AGE
    test-pd   1/1     Running   1          16h
    web-0     1/1     Running   0          82m
    [root@master01 pv]# kubectl exec test-pd -it -- /bin/sh
    / # ping web-0.nginx
    PING web-0.nginx (10.244.3.77): 56 data bytes
    64 bytes from 10.244.3.77: seq=0 ttl=64 time=0.129 ms

####回收资源

    [root@master01 pv]# kubectl delete -f pod.yaml 
    service "nginx" deleted
    statefulset.apps "web" deleted
    删除服务后，仍然是Bound
    [root@master01 pv]# kubectl get pv
    NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   REASON   AGE
    nfspv1   10Gi       RWO            Retain           Bound       default/www-web-0   nfs                     163m
    nfspv2   5Gi        RWX            Retain           Available                       nfs                     122m
    [root@master01 pv]# kubectl get pvc
    NAME        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    www-web-0   Bound    nfspv1   10Gi       RWO            nfs            100m
    [root@master01 pv]# kubectl delete pvc www-web-0
    persistentvolumeclaim "www-web-0" deleted
    [root@master01 pv]# kubectl get pvc
    No resources found in default namespace.
    删除pvc后，资源仍然是不可利用的状态
    [root@master01 pv]# kubectl get pv
    NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   REASON   AGE
    nfspv1   10Gi       RWO            Retain           Released    default/www-web-0   nfs                     164m
    nfspv2   5Gi        RWX            Retain           Available                       nfs                     124m
    [root@master01 pv]# kubectl edit pv nfspv1
    删除下面内容
      claimRef:
        apiVersion: v1
        kind: PersistentVolumeClaim
        name: www-web-0
        namespace: default
        resourceVersion: "403319"
        uid: 1f7bf299-978b-4f9c-841e-caff88feecb7
    资源已是可利用状态    
    [root@master01 pv]# kubectl get pv
    NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
    nfspv1   10Gi       RWO            Retain           Available           nfs                     168m
    nfspv2   5Gi        RWX            Retain           Available           nfs                     127m
