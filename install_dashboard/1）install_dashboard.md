####dashboard安装

wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
charge image push and service type

          image: kubernetesui/dashboard:v2.0.3
          imagePullPolicy: Always
          
          修改为：
          imagePullPolicy: IfNotPresent
          
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
修改为
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30000
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard              
    
image: kubernetesui/metrics-scraper:v1.0.4
修改为
image: kubernetesui/metrics-scraper:v1.0.4
imagePullPolicy: IfNotPresent
    
[root@master01 k8s]# kubectl apply -f  recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

[root@master01 k8s]# kubectl get pods -n kubernetes-dashboard
NAME                                        READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-c79c65bb7-wdtt2   1/1     Running   0          29m
kubernetes-dashboard-b7cbc659d-f64sm        1/1     Running   0          29m
[root@master01 k8s]# kubectl get svc -n kubernetes-dashboard 
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.110.147.200   <none>        8000/TCP        30m
kubernetes-dashboard        NodePort    10.96.209.238    <none>        443:30000/TCP   30m
访问：https://192.168.37.11:30000/  此时没有token
    
生成最高权限的token    
[root@master01 k8s]# cat dashboard_token.yaml 
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile    

获取token    
[root@master01 k8s]# kubectl get secret --all-namespaces|grep dashboard-admin
kube-system            dashboard-admin-token-xqlq8                      kubernetes.io/service-account-token   3      18m
[root@master01 k8s]# kubectl describe secret -n kube-system dashboard-admin-token-xqlq8   