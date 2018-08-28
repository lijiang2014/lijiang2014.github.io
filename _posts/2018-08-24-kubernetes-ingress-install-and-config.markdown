---
layout: post
title:  "Kubernets ingress 安装及配置"
date:   2018-08-24 12:08:45 +0800
categories: share
---

 Ingress 是 kubernetes 除了 nodeport, loadBalancer 以及 portProxy 之外的另一种暴露服务的方式 ， 相关的技术特点见 [kubernetes Docsconcepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
 
 ingress 并非默认组件，需要额外安装 Ingress controller , 有多种选择， 这里选择的是 ingress nginx controller.
 
 ingress nginx controller 目前有两个主流的版本，相互之间有一定差异，需要注意选择：

* [nginxinc/kubernetes-ingress](https://github.com/nginxinc/kubernetes-ingress) ,nginx 提供的版本，基础版可以很简单的安装，但功能较弱，可以先拿来熟悉 ingress , [安装过程](https://github.com/nginxinc/kubernetes-ingress/blob/master/install/README.md)

* [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx) , kubernetes 提供的版本，需要根据自己的集群修改配置，[安装过程](https://kubernetes.github.io/ingress-nginx/deploy/)

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
POD_NAMESPACE=ingress-nginx
POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=ingress-nginx -o jsonpath={.items[0].metadata.name})
kubectl exec -it $POD_NAME -n $POD_NAMESPACE -- /nginx-ingress-controller --version
``` 

默认的 配置在自己的kubernetes 集群上使用可能会有问题，针对我的场景，由于不是AWS这种提供了LoadBalence 的平台，所以采用 DaemonSet 的方式进行部署, 对mandatory.yaml 进行了修改：

```
apiVersion: extensions/v1beta1
#kind: Deployment
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  #replicas: 1
  ...
            #- --publish-service=$(POD_NAMESPACE)/ingress-nginx
  ...
            ports:
          - name: http
            containerPort: 80
            hostPort: 80
          - name: https
            containerPort: 443
            hostPort: 443
```




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

我这里需要暴露的主要是网站服务，另外有一个非内部管理的服务也需要通过 此域名对外暴露

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sl-web-ui
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.org/websocket-services: "sl-web-ui"
    nginx.ingress.kubernetes.io/compute-full-forwarded-for: "true"
spec:
  tls:
  - hosts:
    - starlights.nscc-gz.cn
    secretName: starlights-secret
  rules:
  - host: starlights.nscc-gz.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: sl-web-ui
          servicePort: 80
      - path: /api/guacamole/
        backend:
          serviceName: guacamole-proxy
          servicePort: 80

---
---
kind: Service
apiVersion: v1
metadata:
  name: guacamole-proxy
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8088
  externalIPs:
  - 10.127.48.18
---
kind: Endpoints
apiVersion: v1
metadata:
  name: guacamole-proxy
subsets:
  - addresses:
      - ip: 10.127.48.18
    ports:
      - port: 8088
```

这里通过直接配置 endpoint 的方式来设置非通过 kubernetes 部署的服务.

但是如此设置后发现未能生效，后来进入 nginx-ingress 的 pod 中查看，可以发现 nginx 的配置文件并未按照预先设想的方式产生，有两处配置没有生效：

* `rewrite` 没有生效 
* `proxy_buffer ` 按照nginx ingress 的文档默认应该是 off , 但实际发现不知为何成了 on 

目前通过手动修复 nginx-ingress pods 中的 nginx conf 文件并reload 的方式使得服务成功部署了，但是如果更新 ingress 设置后 ， conf 会如何变化尚未知晓，还是得多检测 nginx conf 文件的变化 ， 说明文档未必完全可信




