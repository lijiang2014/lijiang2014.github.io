---
layout: post
title:  "Kubernets ingress 安装及配置"
date:   2018-08-24 12:08:45 +0800
categories: share
---

 Ingress 是 kubernetes 除了 nodeport, loadBalancer 以及 portProxy 之外的另一种暴露服务的方式 ， 相关的技术特点见 [kubernetes Docsconcepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
 
 ingress 并非默认组件，需要额外安装 Ingress controller , 有多种选择， 这里选择的是 ingress nginx controller.
 
 [ingress nginx controller. ](https://www.nginx.com/products/nginx/kubernetes-ingress-controller/) 的安装需要下载链接中的 [github repo](https://github.com/nginxinc/kubernetes-ingress), 默认的安装过程参考 其中的 [install/README.md](https://github.com/nginxinc/kubernetes-ingress/blob/master/install/README.md) 即可
 
 在安装完后，有些配置需要做响应的调整 ：
 
 1. 更新证书
 
 这里用的是 lets encrypt 的证书

```
cat /etc/letsencrypt/live/starlights.nscc-gz.cn/cert.pem | base64 | awk '{printf("%s",$0)}' | xargs echo " tls.cert :" > cert.part

cat /etc/letsencrypt/live/starlights.nscc-gz.cn/privkey.pem | base64 | awk '{printf("%s",$0)}' | xargs echo " tls.key :" > key.part

cat common/default-server-secret.yaml.H cert.part key.part > common/default-server-secret.yaml

rm *.part

kubectl describe secret default-server-secret --namespace=nginx-ingress

kubectl get DaemonSet --namespace=nginx-ingress
```
>  common/default-server-secret.yaml.H 是 由 原来的 common/default-server-secret.yaml 删除了 最后两行得到的文件
>  k8s secret 需要base64 编码且只能有一行

2. 暴露自己需要的服务 

