# k8s



## 启动minikube

```shell
minikube start --driver=docker --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --base-image="anjone/kicbase"
```



## 管理K8s核心资源的三种基本方法

- 陈述式管理
- 声明式管理
- GUI式管理

1. 陈述式管理

```shell
# namespace 
kubectl get ns
kubectl create ns app
kuebectl delete ns app

# 命名空间下的资源
kubectl get all [-n default] # 获得命名空间下的所有资源

#  deployment
kubectl create deployment nginx-dp --image=nginx -n default 
kubectl get deploy -n default [-o wide] 
kubectl delete deploy nginx-dp -n default 

kubectl	describe [deploy/svc] nginx-dp -n default # 详细描述

# pod
kubectl get pods -n default -o wide # 获取pod
kubectl delete pods  nginx-dp-6f9d9677fd-gf6jx -n  default # 删除pod(其实是重启一个,以保证预期目标pod数)
kubectl exec nginx-dp-6f9d9677fd-gf6jx -it bash -n default #进入pod

# service
kubectl expose deployment nginx-dp --port=8000 -n default
kubectl get svc -n default
kubectl delete svc nginx-dp -n default
kubectl scale deploy nginx-dp --replicas=2 -n default

```

2. 声明式管理

```shell
# 配置清单
kubectl get pods pod-name -o yaml -n default

#  醒配置
kubectl explain xx.xx.xx
#
kubectl [create|delete|apply] -f path/to/xxx.[yaml|json]
```



