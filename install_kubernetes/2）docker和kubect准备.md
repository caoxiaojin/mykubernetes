####安装docker软件

    yum install -y yum-utils device-mapper-persistent-data lvm2
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    yum-config-manager --enable docker-ce-edge
    yum update -y && yum install -y docker-ce
    mkdir /etc/docker
    cat > /etc/docker/daemon.json <<EOF
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size":"100m"
        }
    }

    #启动和开机自启
    mkdir -p /etc/systemd/system/docker.service.d
    systemctl daemon-reload && systemctl start docker.service && systemctl enable docker.service
####调整内核参数，对应k8s

    cat > /etc/sysctl.d/kubernetes.conf <<EOF
    net.bridge.bridge-nf-call-iptables=1
    net.bridge.bridge-nf-call-ip6tables=1
    net.ipv4.ip_forward=1
    net.ipv4.tcp_tw_recycle=0
    vm.swappiness=0    #禁止使用swap空间，只有当系统 OOM时才允许使用它
    vm.overcommit_memory=1 # 不检查物理内存首付够用
    vm.panic_on_oom=0 # 开启OOM
    fs.inotify.max_user_instances=8192
    fs.inotify.max_user_watches=1048576
    fs.file-max=52706963
    fs.nr_open=52706963
    net.ipv6.conf.all.disable_ipv6=1
    #net.netfiler.nf_conntrack_max=2310720  将内核升级为4.4，就可以了
    EOF



