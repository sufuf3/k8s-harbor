# 快速安裝
- Download this repo
```sh
$ git clone https://github.com/sufuf3/k8s-harbor.git
$ cd k8s-harbor
```
- Deploy
```
kubectl apply -f make/kubernetes/pv/log.pv.yaml
kubectl apply -f make/kubernetes/pv/registry.pv.yaml
kubectl apply -f make/kubernetes/pv/storage.pv.yaml
kubectl apply -f make/kubernetes/pv/log.pvc.yaml
kubectl apply -f make/kubernetes/pv/registry.pvc.yaml
kubectl apply -f make/kubernetes/pv/storage.pvc.yaml
kubectl apply -f make/kubernetes/jobservice/jobservice.cm.yaml
kubectl apply -f make/kubernetes/mysql/mysql.cm.yaml
kubectl apply -f make/kubernetes/registry/registry.cm.yaml
kubectl apply -f make/kubernetes/ui/ui.cm.yaml
kubectl apply -f make/kubernetes/adminserver/adminserver.cm.yaml
kubectl apply -f make/kubernetes/jobservice/jobservice.svc.yaml
kubectl apply -f make/kubernetes/mysql/mysql.svc.yaml
kubectl apply -f make/kubernetes/registry/registry.svc.yaml
kubectl apply -f make/kubernetes/ui/ui.svc.yaml
kubectl apply -f make/kubernetes/adminserver/adminserver.svc.yaml
kubectl apply -f make/kubernetes/registry/registry.deploy.yaml
kubectl apply -f make/kubernetes/mysql/mysql.deploy.yaml
kubectl apply -f make/kubernetes/jobservice/jobservice.deploy.yaml
kubectl apply -f make/kubernetes/ui/ui.deploy.yaml
kubectl apply -f make/kubernetes/adminserver/adminserver.deploy.yaml
kubectl apply -f make/kubernetes/ingress.yaml
```
- Let's move [here](install_README_zh.md#7-%E7%A2%BA%E8%AA%8D%E6%89%80%E6%9C%89%E8%B3%87%E8%A8%8A)
