#### 从远程仓库安装dashboard

更新仓库
helm repo update  

[root@master01 dashboard]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[root@master01 dashboard]# helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts  

下载官方dashboard仓库
[root@master01 dashboard]# helm fetch stable/kubernetes-dashboard
[root@master01 dashboard]# ls
kubernetes-dashboard-1.11.1.tgz


image:
  repository: k8s.gcr.io/kubernetes-dashboard-amd64
  tag:v1.10.1

docker pull k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1