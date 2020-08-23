####准备高可用的节点镜像

    docker load -i haproxy.tar
    docker load -i keepalived.tar
    
    解压相关的配置文件。注意要解压在date目录
    tar xf start.keep.tar.gz

---
###先在master01上面安装
    高可用启动顺序：
    1）先在master01安装高可用的vip节点，防止调用到了未启动的节点上
    2）master01 启动 haproxy
    3）master01 启动 keepalived
    4）master01 初始话k8s集群
    5）master02和master03启动haproxy，keepalived，此时vip调度在mastero1上面
    6）master02和master03都加入集群后，修改调度在3台机器上
#### 修改配置start-haproxy.sh

    修改配置
    [root@master01 etc]# cat haproxy.cfg 
    global
    log 127.0.0.1 local0
    log 127.0.0.1 local1 notice
    maxconn 4096
    #chroot /usr/share/haproxy
    #user haproxy
    #group haproxy
    daemon
    
    defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        option redispatch
        timeout connect  5000
        timeout client  50000
        timeout server  50000
    
    frontend stats-front
      bind *:8081
      mode http
      default_backend stats-back
    
    frontend fe_k8s_6444
      bind *:6444
      mode tcp
      timeout client 1h
      log global
      option tcplog
      default_backend be_k8s_6443
      acl is_websocket hdr(Upgrade) -i WebSocket
      acl is_websocket hdr_beg(Host) -i ws
    
    backend stats-back
      mode http
      balance roundrobin
      stats uri /haproxy/stats
      stats auth pxcstats:secret
    
    backend be_k8s_6443
      mode tcp
      timeout queue 1h
      timeout server 1h
      timeout connect 1h
      log global
      balance roundrobin
      server rancher01 192.168.37.11:6443
      
      *备注：第一次启动时，先设置一个节点*
      
      查看启动脚本
     [root@master01 lb]# cat start-haproxy.sh 
     #!/bin/bash
     MasterIP1=192.168.37.11
     MasterIP2=192.168.37.12
     MasterIP3=192.168.37.13
     MasterPort=6443
      
    docker run -d --restart=always --name HAProxy-K8S -p 6444:6444 \
            -e MasterIP1=$MasterIP1 \
            -e MasterIP2=$MasterIP2 \
            -e MasterIP3=$MasterIP3 \
            -e MasterPort=$MasterPort \
            -v /data/lb/etc/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg \
            wise2c/haproxy-k8s
    
####启动服务

    [root@master01 lb]# ./start-haproxy.sh 
    54a23e4029b0497c0401ca1ee0eada30a4cea7665028ead38ed270327224f471
    [root@master01 lb]# docker ps
    CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS                    NAMES
    54a23e4029b0        wise2c/haproxy-k8s   "/docker-entrypoint.…"   18 minutes ago      Up 18 minutes       0.0.0.0:6444->6444/tcp   HAProxy-K8S
    
####查看keepalived的配置

    [root@master01 lb]# cat start-keepalived.sh 
    #!/bin/bash
    VIRTUAL_IP=192.168.37.100
    INTERFACE=ens33
    NETMASK_BIT=24
    CHECK_PORT=6444
    RID=10
    VRID=160
    MCAST_GROUP=224.0.0.18

    docker run -itd --restart=always --name=Keepalived-K8S \
            --net=host --cap-add=NET_ADMIN \
            -e VIRTUAL_IP=$VIRTUAL_IP \
            -e INTERFACE=$INTERFACE \
            -e CHECK_PORT=$CHECK_PORT \
            -e RID=$RID \
            -e VRID=$VRID \
            -e NETMASK_BIT=$NETMASK_BIT \
            -e MCAST_GROUP=$MCAST_GROUP \
            wise2c/keepalived-k8s    

    启动服务
    [root@master01 lb]# ./start-keepalived.sh 
    fd4cd77d08a10c01c615fca7d0f5b52c1768c4847a25777627f3f5850ffa6366
    
####查看vip
    [root@master01 lb]# ip addr show|grep ens33
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        inet 192.168.37.11/24 brd 192.168.37.255 scope global noprefixroute ens33
        inet 192.168.37.100/24 scope global secondary ens33    
        
    #### master01机器准备安装
    
    mkdir -p /root/k8s/    相关的配置文件均放置里面
       
    [root@master01 k8s]# cat kubeadm-config.yaml
    apiVersion: kubeadm.k8s.io/v1beta2
    bootstrapTokens:
    - groups:
      - system:bootstrappers:kubeadm:default-node-token
      token: abcdef.0123456789abcdef
      ttl: 24h0m0s
      usages:
      - signing
      - authentication
    kind: InitConfiguration
    localAPIEndpoint:
      advertiseAddress: 192.168.37.11
      bindPort: 6443
    nodeRegistration:
      criSocket: /var/run/dockershim.sock
      name: master01
      taints:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
    ---
    apiServer:
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta2
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controlPlaneEndpoint: "192.168.37.100:6444"
    controllerManager: {}
    dns:
      type: CoreDNS
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: k8s.gcr.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.17.3
    networking:
      dnsDomain: cluster.local
      podSubnet: "10.244.0.0/16"
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
    ---
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    featureGates:
      SupportIPVSProxyMode: true
    mode: ipvs
    
    初始话服务
    kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.log
    
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config  
---    
####其他master节点启动高可用配置

    tar -czvf data.tar.gz data/
    scp data.tar.gz master02:/
    scp data.tar.gz master03:/
    
    [root@master02 lb]# ./start-haproxy.sh 
    7a00a51c9df8ebf08510ef3eda9397e17f1c876f51765daf58f59dd59ef24c22
    [root@master02 lb]# 
    [root@master02 lb]# ./start-keepalived.sh 
    dea68afa8735885cbccafbb0281f72492d49a0c1ee144285334134e8ebfadd16
    
    kubeadm join 192.168.37.100:6444 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:055142b121edd15278977e1bf92c7a051ae269c92f6966d534708561d6555a46 \
        --control-plane --certificate-key 508a8747fda3e7f20de63fb45284166e086946f6961b351c3a768f374b572232
    	
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
####检查是否有3台master
    [root@master03 ~]# kubectl get nodes
    NAME       STATUS     ROLES    AGE     VERSION
    master01   NotReady   master   32m     v1.17.3
    master02   NotReady   master   8m26s   v1.17.3
    master03   NotReady   master   88s     v1.17.3

#### 部署 flannel 网络
    [root@master01 k8s]# cat kube-flannel.yml 
    ---
    apiVersion: policy/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: psp.flannel.unprivileged
      annotations:
        seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
        seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
        apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
        apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
    spec:
      privileged: false
      volumes:
        - configMap
        - secret
        - emptyDir
        - hostPath
      allowedHostPaths:
        - pathPrefix: "/etc/cni/net.d"
        - pathPrefix: "/etc/kube-flannel"
        - pathPrefix: "/run/flannel"
      readOnlyRootFilesystem: false
      # Users and groups
      runAsUser:
        rule: RunAsAny
      supplementalGroups:
        rule: RunAsAny
      fsGroup:
        rule: RunAsAny
      # Privilege Escalation
      allowPrivilegeEscalation: false
      defaultAllowPrivilegeEscalation: false
      # Capabilities
      allowedCapabilities: ['NET_ADMIN']
      defaultAddCapabilities: []
      requiredDropCapabilities: []
      # Host namespaces
      hostPID: false
      hostIPC: false
      hostNetwork: true
      hostPorts:
      - min: 0
        max: 65535
      # SELinux
      seLinux:
        # SELinux is unused in CaaSP
        rule: 'RunAsAny'
    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: flannel
    rules:
      - apiGroups: ['extensions']
        resources: ['podsecuritypolicies']
        verbs: ['use']
        resourceNames: ['psp.flannel.unprivileged']
      - apiGroups:
          - ""
        resources:
          - pods
        verbs:
          - get
      - apiGroups:
          - ""
        resources:
          - nodes
        verbs:
          - list
          - watch
      - apiGroups:
          - ""
        resources:
          - nodes/status
        verbs:
          - patch
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: flannel
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: flannel
    subjects:
    - kind: ServiceAccount
      name: flannel
      namespace: kube-system
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: flannel
      namespace: kube-system
    ---
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: kube-flannel-cfg
      namespace: kube-system
      labels:
        tier: node
        app: flannel
    data:
      cni-conf.json: |
        {
          "name": "cbr0",
          "cniVersion": "0.3.1",
          "plugins": [
            {
              "type": "flannel",
              "delegate": {
                "hairpinMode": true,
                "isDefaultGateway": true
              }
            },
            {
              "type": "portmap",
              "capabilities": {
                "portMappings": true
              }
            }
          ]
        }
      net-conf.json: |
        {
          "Network": "10.244.0.0/16",
          "Backend": {
            "Type": "vxlan"
          }
        }
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: kube-flannel-ds-amd64
      namespace: kube-system
      labels:
        tier: node
        app: flannel
    spec:
      selector:
        matchLabels:
          app: flannel
      template:
        metadata:
          labels:
            tier: node
            app: flannel
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: kubernetes.io/os
                        operator: In
                        values:
                          - linux
                      - key: kubernetes.io/arch
                        operator: In
                        values:
                          - amd64
          hostNetwork: true
          priorityClassName: system-node-critical
          tolerations:
          - operator: Exists
            effect: NoSchedule
          serviceAccountName: flannel
          initContainers:
          - name: install-cni
            image: quay.io/coreos/flannel:v0.12.0-amd64
            command:
            - cp
            args:
            - -f
            - /etc/kube-flannel/cni-conf.json
            - /etc/cni/net.d/10-flannel.conflist
            volumeMounts:
            - name: cni
              mountPath: /etc/cni/net.d
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          containers:
          - name: kube-flannel
            image: quay.io/coreos/flannel:v0.12.0-amd64
            command:
            - /opt/bin/flanneld
            args:
            - --ip-masq
            - --kube-subnet-mgr
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
              limits:
                cpu: "100m"
                memory: "50Mi"
            securityContext:
              privileged: false
              capabilities:
                add: ["NET_ADMIN"]
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            volumeMounts:
            - name: run
              mountPath: /run/flannel
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          volumes:
            - name: run
              hostPath:
                path: /run/flannel
            - name: cni
              hostPath:
                path: /etc/cni/net.d
            - name: flannel-cfg
              configMap:
                name: kube-flannel-cfg
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: kube-flannel-ds-arm64
      namespace: kube-system
      labels:
        tier: node
        app: flannel
    spec:
      selector:
        matchLabels:
          app: flannel
      template:
        metadata:
          labels:
            tier: node
            app: flannel
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: kubernetes.io/os
                        operator: In
                        values:
                          - linux
                      - key: kubernetes.io/arch
                        operator: In
                        values:
                          - arm64
          hostNetwork: true
          priorityClassName: system-node-critical
          tolerations:
          - operator: Exists
            effect: NoSchedule
          serviceAccountName: flannel
          initContainers:
          - name: install-cni
            image: quay.io/coreos/flannel:v0.12.0-arm64
            command:
            - cp
            args:
            - -f
            - /etc/kube-flannel/cni-conf.json
            - /etc/cni/net.d/10-flannel.conflist
            volumeMounts:
            - name: cni
              mountPath: /etc/cni/net.d
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          containers:
          - name: kube-flannel
            image: quay.io/coreos/flannel:v0.12.0-arm64
            command:
            - /opt/bin/flanneld
            args:
            - --ip-masq
            - --kube-subnet-mgr
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
              limits:
                cpu: "100m"
                memory: "50Mi"
            securityContext:
              privileged: false
              capabilities:
                 add: ["NET_ADMIN"]
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            volumeMounts:
            - name: run
              mountPath: /run/flannel
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          volumes:
            - name: run
              hostPath:
                path: /run/flannel
            - name: cni
              hostPath:
                path: /etc/cni/net.d
            - name: flannel-cfg
              configMap:
                name: kube-flannel-cfg
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: kube-flannel-ds-arm
      namespace: kube-system
      labels:
        tier: node
        app: flannel
    spec:
      selector:
        matchLabels:
          app: flannel
      template:
        metadata:
          labels:
            tier: node
            app: flannel
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: kubernetes.io/os
                        operator: In
                        values:
                          - linux
                      - key: kubernetes.io/arch
                        operator: In
                        values:
                          - arm
          hostNetwork: true
          priorityClassName: system-node-critical
          tolerations:
          - operator: Exists
            effect: NoSchedule
          serviceAccountName: flannel
          initContainers:
          - name: install-cni
            image: quay.io/coreos/flannel:v0.12.0-arm
            command:
            - cp
            args:
            - -f
            - /etc/kube-flannel/cni-conf.json
            - /etc/cni/net.d/10-flannel.conflist
            volumeMounts:
            - name: cni
              mountPath: /etc/cni/net.d
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          containers:
          - name: kube-flannel
            image: quay.io/coreos/flannel:v0.12.0-arm
            command:
            - /opt/bin/flanneld
            args:
            - --ip-masq
            - --kube-subnet-mgr
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
              limits:
                cpu: "100m"
                memory: "50Mi"
            securityContext:
              privileged: false
              capabilities:
                 add: ["NET_ADMIN"]
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            volumeMounts:
            - name: run
              mountPath: /run/flannel
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          volumes:
            - name: run
              hostPath:
                path: /run/flannel
            - name: cni
              hostPath:
                path: /etc/cni/net.d
            - name: flannel-cfg
              configMap:
                name: kube-flannel-cfg
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: kube-flannel-ds-ppc64le
      namespace: kube-system
      labels:
        tier: node
        app: flannel
    spec:
      selector:
        matchLabels:
          app: flannel
      template:
        metadata:
          labels:
            tier: node
            app: flannel
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: kubernetes.io/os
                        operator: In
                        values:
                          - linux
                      - key: kubernetes.io/arch
                        operator: In
                        values:
                          - ppc64le
          hostNetwork: true
          priorityClassName: system-node-critical
          tolerations:
          - operator: Exists
            effect: NoSchedule
          serviceAccountName: flannel
          initContainers:
          - name: install-cni
            image: quay.io/coreos/flannel:v0.12.0-ppc64le
            command:
            - cp
            args:
            - -f
            - /etc/kube-flannel/cni-conf.json
            - /etc/cni/net.d/10-flannel.conflist
            volumeMounts:
            - name: cni
              mountPath: /etc/cni/net.d
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          containers:
          - name: kube-flannel
            image: quay.io/coreos/flannel:v0.12.0-ppc64le
            command:
            - /opt/bin/flanneld
            args:
            - --ip-masq
            - --kube-subnet-mgr
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
              limits:
                cpu: "100m"
                memory: "50Mi"
            securityContext:
              privileged: false
              capabilities:
                 add: ["NET_ADMIN"]
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            volumeMounts:
            - name: run
              mountPath: /run/flannel
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          volumes:
            - name: run
              hostPath:
                path: /run/flannel
            - name: cni
              hostPath:
                path: /etc/cni/net.d
            - name: flannel-cfg
              configMap:
                name: kube-flannel-cfg
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: kube-flannel-ds-s390x
      namespace: kube-system
      labels:
        tier: node
        app: flannel
    spec:
      selector:
        matchLabels:
          app: flannel
      template:
        metadata:
          labels:
            tier: node
            app: flannel
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: kubernetes.io/os
                        operator: In
                        values:
                          - linux
                      - key: kubernetes.io/arch
                        operator: In
                        values:
                          - s390x
          hostNetwork: true
          priorityClassName: system-node-critical
          tolerations:
          - operator: Exists
            effect: NoSchedule
          serviceAccountName: flannel
          initContainers:
          - name: install-cni
            image: quay.io/coreos/flannel:v0.12.0-s390x
            command:
            - cp
            args:
            - -f
            - /etc/kube-flannel/cni-conf.json
            - /etc/cni/net.d/10-flannel.conflist
            volumeMounts:
            - name: cni
              mountPath: /etc/cni/net.d
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          containers:
          - name: kube-flannel
            image: quay.io/coreos/flannel:v0.12.0-s390x
            command:
            - /opt/bin/flanneld
            args:
            - --ip-masq
            - --kube-subnet-mgr
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
              limits:
                cpu: "100m"
                memory: "50Mi"
            securityContext:
              privileged: false
              capabilities:
                 add: ["NET_ADMIN"]
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            volumeMounts:
            - name: run
              mountPath: /run/flannel
            - name: flannel-cfg
              mountPath: /etc/kube-flannel/
          volumes:
            - name: run
              hostPath:
                path: /run/flannel
            - name: cni
              hostPath:
                path: /etc/cni/net.d
            - name: flannel-cfg
              configMap:
                name: kube-flannel-cfg
---
    
    kubectl apply -f  kube-flannel.yml
    [root@master02 ~]# kubectl get nodes
    NAME       STATUS   ROLES    AGE   VERSION
    master01   Ready    master   73m   v1.17.3
    master02   Ready    master   49m   v1.17.3
    master03   Ready    master   42m   v1.17.3
    
####修改haproxy的配置文件，让其可用调度到3台机器上
    [root@master01 /]# cat /data/lb/etc/haproxy.cfg
    .......	
    backend be_k8s_6443
      mode tcp
      timeout queue 1h
      timeout server 1h
      timeout connect 1h
      log global
      balance roundrobin
      server rancher01 192.168.37.11:6443
      server rancher02 192.168.37.12:6443
      server rancher03 192.168.37.13:6443  
    
    重启 haproxy 
    docker rm -f HAProxy-K8S && bash /data/lb/start-haproxy.sh     
    
#### 更改 master节点的config配置
    将原来的vip改为本机ip
    原先的
    server: https://192.168.37.100:6444
    
    改为：
    server: https://192.168.37.11:6443   
    意义：如果其他一个master节点down了，执行kubect命令会随机卡住。改为本机ip，则不会有这个情况
    
#### 其他节点加入集群
    kubeadm join 192.168.37.100:6444 --token abcdef.0123456789abcdef \
    >     --discovery-token-ca-cert-hash sha256:055142b121edd15278977e1bf92c7a051ae269c92f6966d534708561d6555a46
    [root@master02 .kube]# kubectl get nodes
    NAME       STATUS   ROLES    AGE    VERSION
    master01   Ready    master   148m   v1.17.3
    master02   Ready    master   124m   v1.17.3
    master03   Ready    master   117m   v1.17.3
    node01     Ready    <none>   18s    v1.17.3    