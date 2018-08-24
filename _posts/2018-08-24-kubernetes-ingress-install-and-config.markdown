---
layout: post
title:  "Kubernets ingress 安装及配置"
date:   2018-08-24 12:08:45 +0800
categories: share
---

```
cat /etc/letsencrypt/live/starlights.nscc-gz.cn/cert.pem | base64 | awk '{printf("%s",$0)}' | xargs echo " tls.cert :" > cert.part

cat /etc/letsencrypt/live/starlights.nscc-gz.cn/privkey.pem | base64 | awk '{printf("%s",$0)}' | xargs echo " tls.key :" > key.part

cat common/default-server-secret.yaml.H cert.part key.part > common/default-server-secret.yaml

rm *.part

kubectl describe secret default-server-secret --namespace=nginx-ingress

kubectl get DaemonSet --namespace=nginx-ingress
```
