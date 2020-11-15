1）创建存储池并启用RBD功能
2）创建ceph用户，提供给K8S使用
3）在k8s节点上安装ceph-common rpm包   yum install ceph-common -y
4）复制ceph.client.admin.keyring，ceph.conf，已经admin用户的keyring文件到k8s节点（master and node）
拷贝文件到/etc/ceph/ 目录下面
5）创建Secret资源，以keying的key值为data
6）静态pv使用
   创建pv
   创建pvc
   在ceph存储池里面创建对应的RBDImage
   创建pod
7）动态pv使用
   创建StorageClass
   创建pvc
   创建pv


[ceph@master01 ~]$ ceph version
ceph version 14.2.13 (1778d63e55dbff6cedb071ab7d367f8f52a8699f) nautilus (stable)
[ceph@master01 ~]$ ceph osd tree
ID CLASS WEIGHT  TYPE NAME         STATUS REWEIGHT PRI-AFF 
-1       0.02939 root default                              
-3       0.00980     host master01                         
 0   hdd 0.00980         osd.0         up  1.00000 1.00000 
-5       0.00980     host node01                           
 1   hdd 0.00980         osd.1         up  1.00000 1.00000 
-7       0.00980     host node02                           
 2   hdd 0.00980         osd.2         up  1.00000 1.00000 
[ceph@master01 ~]$ ceph osd pool ls
cephfs_data
cephfs_metadata
.rgw.root
default.rgw.control
default.rgw.meta
default.rgw.log
rbd
[ceph@master01 ~]$ ceph osd pool create mypool 10    这里创建的有问题，可用ceph -s查看
pool 'mypool' created
[ceph@master01 ~]$ ceph osd pool ls
cephfs_data
cephfs_metadata
.rgw.root
default.rgw.control
default.rgw.meta
default.rgw.log
rbd
mypool
[ceph@master01 ~]$ ceph auth get-or-create client.kube mon 'allow r' osd 'allow class-read object_prefix rbd_children,allow rwx pool=mypool'
[client.kube]
	key = AQDqGrFfuDmCHhAAtD+v5LPExAiz0YHUEyrdaA==
[root@master01 manifests]# ceph auth get-key client.admin
AQABCqhfE7AYFxAAjqMKpCWGc/wN41w/GkVHfw==[root@master01 manifests]# 
[root@master01 manifests]# ceph auth get-key client.admin| base64
QVFBQkNxaGZFN0FZRnhBQWpxTUtwQ1dHYy93TjQxdy9Ha1ZIZnc9PQ==
[root@master01 manifests]# cat ceph-admin-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: default
data:
  key: QVFBQkNxaGZFN0FZRnhBQWpxTUtwQ1dHYy93TjQxdy9Ha1ZIZnc9PQ==
type:
  kubernetes.io/rbd
[root@master01 manifests]# kubectl apply -f ceph-admin-secret.yaml 
secret/ceph-admin-secret created
[root@master01 manifests]# kubectl get secret|grep ceph
ceph-admin-secret                        kubernetes.io/rbd                     1      54s
[root@master01 manifests]# ceph auth get-key client.kube|base64
QVFEcUdyRmZ1RG1DSGhBQXREK3Y1TFBFeEFpejBZSFVFeXJkYUE9PQ==
[root@master01 manifests]# cat ceph-kube-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-kube-secret
  namespace: default
data:
  key: QVFEcUdyRmZ1RG1DSGhBQXREK3Y1TFBFeEFpejBZSFVFeXJkYUE9PQ==
type:
  kubernetes.io/rbd

[root@master01 manifests]# kubectl apply -f ceph-kube-secret.yaml 
secret/ceph-kube-secret created
[root@master01 manifests]# 
[root@master01 manifests]# kubectl get secret|grep ceph
ceph-admin-secret                        kubernetes.io/rbd                     1      4m44s
ceph-kube-secret                         kubernetes.io/rbd                     1      17s


备注：回收策略
persistentVolumeReclaimPolicy: Recycle   删除pv。存储也删除。不安全
persistentVolumeReclaimPolicy: Retain    删除pv，存储仍在

静态pv使用
[root@master01 manifests]# cat pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-test-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - master01:6789
      - node01:6789
      - node02:6789
    pool: mypool
    image: ceph-image
    user: admin
    secretRef:
      name: ceph-admin-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Retain
[root@master01 manifests]# kubectl apply -f pv.yaml 
persistentvolume/ceph-test-pv created
[root@master01 manifests]# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
ceph-test-pv   2Gi        RWO            Retain           Available                                   7s

[root@master01 manifests]# kubectl describe pv ceph-test-pv

[root@master01 manifests]# rbd create -p mypool -s 2G ceph-image   不清楚操作意义
[root@master01 manifests]# rbd ls -p mypool
ceph-image
[root@master01 manifests]# 
[root@master01 manifests]# 
[root@master01 manifests]# rbd info ceph-image -p mypool
rbd image 'ceph-image':
	size 2 GiB in 1280 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 1e58b16b89054
	block_name_prefix: rbd_data.1e58b16b89054
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Sun Nov 15 21:46:13 2020
	access_timestamp: Sun Nov 15 21:46:13 2020
	modify_timestamp: Sun Nov 15 21:46:13 2020
[root@master01 manifests]# 

[root@master01 manifests]# cat pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-test-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
[root@master01 manifests]# 
[root@master01 manifests]# 
[root@master01 manifests]# kubectl apply -f pvc.yaml 
persistentvolumeclaim/ceph-test-claim created
[root@master01 manifests]# 
[root@master01 manifests]# kubectl get pvc
NAME              STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-test-claim   Bound    ceph-test-pv   2Gi        RWO                           7s
[root@master01 manifests]# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
ceph-test-pv   2Gi        RWO            Retain           Bound    default/ceph-test-claim                           11m
[root@master01 manifests]# cat ceph-busybox-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod
spec: 
  containers:
  - name: ceph-busybox
    image: busybox:v1
    command: ["sleep","60000"]
    volumeMounts:
    - name: ceph-vol1
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-test-claim
[root@master01 manifests]# kubectl apply -f ceph-busybox-pod.yaml 
pod/ceph-pod created
[root@master01 manifests]# kubectl get pods   查看pod不能被创建
NAME                        READY   STATUS              RESTARTS   AGE
ceph-pod                    0/1     ContainerCreating   0          23s
[root@master01 manifests]# kubectl describe pod ceph-pod
......
bd: map failed exit status 6, rbd output: rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable mypool/ceph-image object-map fast-diff deep-flatten".

动态pv使用
1）删除之前的静态相关资源
[root@master01 ~]# kubectl delete pod ceph-pod
pod "ceph-pod" deleted
[root@master01 ~]# kubectl delete pvc ceph-test-claim
persistentvolumeclaim "ceph-test-claim" deleted
[root@master01 ~]# kubectl delete pv ceph-test-pv
persistentvolume "ceph-test-pv" deleted
[root@master01 manifests]# cat storageclass.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rbd
  annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/rbd
parameters:
  monitors: master01:6789,node01:6789,node02:6789
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: default
  pool: mypool
  userId: kube
  userSecretName: ceph-kube-secret
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
reclaimPolicy: "Retain"
[root@master01 manifests]# kubectl apply -f storageclass.yaml 
storageclass.storage.k8s.io/rbd created
[root@master01 manifests]# 
[root@master01 manifests]# kubectl get sc   # 备注该 storageclass 创建有异常
NAME            PROVISIONER         RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rbd (default)   kubernetes.io/rbd   Retain          Immediate           false                  24s
[root@master01 manifests]# kubectl get storageclass
NAME            PROVISIONER         RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rbd (default)   kubernetes.io/rbd   Retain          Immediate           false                  31s
[root@master01 manifests]# kubectl apply -f pvc.yaml  如果storageclass 没有问题，直接创建pvc，可直接绑定pv
persistentvolumeclaim/ceph-test-claim created
例如
[root@master01 manifests]# kubectl get pv
pvc-ce......   Bound
