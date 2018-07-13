# Harbor with Kubernetes Harbor

> 由 [kubernetes_deployment](https://github.com/vmware/harbor/blob/master/docs/kubernetes_deployment.md) 的說明，官方建議使用 Helm 來 deploy Harbor。  
> 本篇參考 https://github.com/vmware/harbor/tree/master/contrib/helm/harbor 做的筆記。  

## Requirements
- Kubernetes version: 1.10.2
- Helm CLI: 2.8.0+
- Kubernetes Ingress Controller is enabled
- PV provisioner support in the underlying infrastructure



## 安裝步驟
### 事前步驟

#### 1. 安裝 Helm
```sh
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
```
From: https://github.com/kubernetes/helm/blob/master/docs/install.md#from-script  

#### 2. Enable ingress-nginx
```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```
From: https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md  

#### 3. Install socat
Make sure there is socat package on every node, if you use CentOS.  
```sh
$ yum install -y socat
```

### 安裝
> https://github.com/vmware/harbor/tree/master/contrib/helm/harbor
1. initialize Helm
```sh
$ helm init
```

2. Add account for deploying tiller(have permission)
```sh
$ kubectl create serviceaccount --namespace kube-system tiller
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' 
$ helm list
$ helm repo update
```
> https://github.com/kubernetes/helm/issues/3130  

3. Download Harbor helm chart
```sh
$ git clone https://github.com/vmware/harbor
```

4. PV prepare
```
cd harbot/
kubectl apply -f make/kubernetes/pv/log.pv.yaml
kubectl apply -f make/kubernetes/pv/registry.pv.yaml
kubectl apply -f make/kubernetes/pv/storage.pv.yaml
```


5. Download external dependent charts required by Harbor chart.
```sh
$ cd harbor/contrib/helm/harbor
$ helm dependency update
```

6. Install the Harbor helm chart with a release name my-release
```sh
helm install . --debug --name my-release --set externalDomain=harbor.my.domain
```
```sh
NOTES:

Please wait for several minutes for Harbor deployment to complete.
Then follow the steps below to use Harbor.

1. Add the Harbor CA certificate to Docker by executing the following command:

  sudo mkdir -p /etc/docker/certs.d/harbor.my.domain
  kubectl get secret \
    --namespace default my-release-harbor-ingress \
    -o jsonpath="{.data.ca\.crt}" | base64 --decode | \
    sudo tee /etc/docker/certs.d/harbor.my.domain/ca.crt

2. Get Harbor admin password by executing the following command:

  kubectl get secret --namespace default my-release-harbor-adminserver -o jsonpath="{.data.HARBOR_ADMIN_PASSWORD}" | base64 --decode; echo

3. Add DNS resolution entry for Harbor FQDN harbor.my.domain to K8s Ingress Controller IP on DNS Server or in file /etc/hosts.
   Add DNS resolution entry for Notary FQDN notary-harbor.my.domain to K8s Ingress Controller IP on DNS Server or in file /etc/hosts.

4. Access Harbor UI via https://harbor.my.domain

5. Login Harbor with Docker CLI:

  docker login harbor.my.domain
```

## delete
```sh
helm del --purge my-release
```
