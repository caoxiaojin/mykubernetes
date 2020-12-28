#### 安装ingress-nginx

[root@master01 k8s]# kubectl apply -f ingress-controller.yaml 
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
daemonset.apps/nginx-ingress-controller created
service/ingress-nginx created

[root@master01 k8s]# kubectl get pods -n ingress-nginx
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-2wmqz   1/1     Running   0          34s
nginx-ingress-controller-5dbbc   1/1     Running   0          34s
[root@master01 k8s]# kubectl get svc -n ingress-nginx
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
ingress-nginx   ClusterIP   10.98.143.64   <none>        80/TCP,443/TCP   41s
node节点会有对应的node节点信息
[root@node01 ~]# netstat -lntup|grep "0 0.0.0.0:80"
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      121840/nginx: maste