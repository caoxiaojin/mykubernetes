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

=========================================================================================================================
[root@master01 dashboard]# kubectl -n kube-system get secret dashboard-admin-token-f4r59 -o yaml
获取到token并写入到文件
[root@master01 dashboard]# cat token.txt 
ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklsWkxZbDh0VGtKaFR6VkxiMkl4U0RBNFEyeGFPRXBCVEdGcFkyWndOR3g1VDNoNVNIQlpPR0pFVVdzaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbExYTjVjM1JsYlNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKa1lYTm9ZbTloY21RdFlXUnRhVzR0ZEc5clpXNHRaalJ5TlRraUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNXVZVzFsSWpvaVpHRnphR0p2WVhKa0xXRmtiV2x1SWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXpaWEoyYVdObExXRmpZMjkxYm5RdWRXbGtJam9pT0dNeE56VXdZV1F0TlRsaVpDMDBPREkzTFdFek1qTXRPVEk1WkRnNE1UazFaamd4SWl3aWMzVmlJam9pYzNsemRHVnRPbk5sY25acFkyVmhZMk52ZFc1ME9tdDFZbVV0YzNsemRHVnRPbVJoYzJoaWIyRnlaQzFoWkcxcGJpSjkub2duZjVXX2lwRV9FZWFTYkdKYmJNa25rWXRDbjFLZXdHOWRrWDR5UF9wLW9Fdi1yS21TYW1HR1pZcENlcmI5UjhzWi1tYlNlV3FwSXlab195TWtvNDlQMWpqdmNOU3Fzc19INGZzUW5mSmR1RFIzRE9QakM3dWxsTDRoNlBfa2FGZzVVeEtHZWtrVF9KS3ZSVzFVbFZhOWZzT0tuTC1majVSeGJRMWxBLXJKOGxpQ1RuRWVqNnRUMXNRZllQZTFZSlRfUG84d0VVWnczRTFCb3FmWG5SVWFHWjN1RkpaZ1VIZVFsRFNUQUJpSlJ0TUFjTVpKMDRvdWg2MFQwQmxNY3NiaThrQTJfcUY0UElRbW5hRXNCaU5oQW42VGZBb3FYUlhhQzFTTGR3MWxvNWluRE5aUmF4WXl1cmtrcXR2cnIxc0RiWkt0RzFoMjRkMk1qdnBoVU13

[root@master01 dashboard]# base64 -d token.txt 
eyJhbGciOiJSUzI1NiIsImtpZCI6IlZLYl8tTkJhTzVLb2IxSDA4Q2xaOEpBTGFpY2ZwNGx5T3h5SHBZOGJEUWsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tZjRyNTkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOGMxNzUwYWQtNTliZC00ODI3LWEzMjMtOTI5ZDg4MTk1ZjgxIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.ognf5W_ipE_EeaSbGJbbMknkYtCn1KewG9dkX4yP_p-oEv-rKmSamGGZYpCerb9R8sZ-mbSeWqpIyZo_yMko49P1jjvcNSqss_H4fsQnfJduDR3DOPjC7ullL4h6P_kaFg5UxKGekkT_JKvRW1UlVa9fsOKnL-fj5RxbQ1lA-rJ8liCTnEej6tT1sQfYPe1YJT_Po8wEUZw3E1BoqfXnRUaGZ3uFJZgUHeQlDSTABiJRtMAcMZJ04ouh60T0BlMcsbi8kA2_qF4PIQmnaEsBiNhAn6TfAoqXRXaC1SLdw1lo5inDNZRaxYyurkkqtvrr1sDbZKtG1h24d2MjvphUMw[root@master01 dashboard]#
或者一条命令搞定
[root@master01 dashboard]# kubectl -n kube-system get secret dashboard-admin-token-f4r59 -o jsonpath={.data.token} | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IlZLYl8tTkJhTzVLb2IxSDA4Q2xaOEpBTGFpY2ZwNGx5T3h5SHBZOGJEUWsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tZjRyNTkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOGMxNzUwYWQtNTliZC00ODI3LWEzMjMtOTI5ZDg4MTk1ZjgxIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.ognf5W_ipE_EeaSbGJbbMknkYtCn1KewG9dkX4yP_p-oEv-rKmSamGGZYpCerb9R8sZ-mbSeWqpIyZo_yMko49P1jjvcNSqss_H4fsQnfJduDR3DOPjC7ullL4h6P_kaFg5UxKGekkT_JKvRW1UlVa9fsOKnL-fj5RxbQ1lA-rJ8liCTnEej6tT1sQfYPe1YJT_Po8wEUZw3E1BoqfXnRUaGZ3uFJZgUHeQlDSTABiJRtMAcMZJ04ouh60T0BlMcsbi8kA2_qF4PIQmnaEsBiNhAn6TfAoqXRXaC1SLdw1lo5inDNZRaxYyurkkqtvrr1sDbZKtG1h24d2MjvphUMw[root@master01 dashboard]#
