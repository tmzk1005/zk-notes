---
title: "Ubuntu配置阿里的kubernats apt源"
date: 2021-10-26T10:29:24+08:00
draft: true
---


见如下脚本

```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF  
apt-get update

# 安装
# apt-get install -y kubelet kubeadm kubectl
```