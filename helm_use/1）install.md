#### Helm 部署

    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.15.1-linux-amd64.tar
    tar xf helm-v2.13.1-linux-amd64.tar.gz  ==> linux-amd64
    cp linux-amd64/helm /usr/bin/
    chmod a+x /usr/bin/helm
    
    oot@master01 helm]# cat tiller-rbac.yaml 
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: tiller
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: tiller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: tiller
        namespace: kube-system
    
    [root@master01 helm]# kubectl create -f tiller-rbac.yaml 
    serviceaccount/tiller created
    clusterrolebinding.rbac.authorization.k8s.io/tiller created
    
    [root@master01 ~]# kubectl get pods -n kube-system|grep tiller
    tiller-deploy-7b875fbf86-n99lx     1/1     Running            0          6h19m
    
    
    [root@master01 helm]# helm init --service-account tiller --skip-refresh
    $HELM_HOME has been configured at /root/.helm.
    Warning: Tiller is already installed in the cluster.
    (Use --client-only to suppress this message, or --upgrade to upgrade Tiller to the current version.)
    
    [root@master01 ~]# helm version
    Client: &version.Version{SemVer:"v2.15.1", GitCommit:"cf1de4f8ba70eded310918a8af3a96bfe8e7683b", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.15.1", GitCommit:"cf1de4f8ba70eded310918a8af3a96bfe8e7683b", GitTreeState:"clean"}