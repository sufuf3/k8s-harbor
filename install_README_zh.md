# Harbor 安裝筆記

## Table of Contents
- [What is Harbor](#what-is-harbor)
- [Install Harbor with Kubernetes locally](#install-harbor-with-kubernetes-locally)

## What is Harbor
[Harbor](https://github.com/vmware/harbor) 是企業級的 container registry，基於 Docker 來部署。
簡單來說就是企業內部自己私有的 Docker Hub 。
如果想要參考其他的 container registry，可以參考這邊 [container-registry-list](https://gist.github.com/sufuf3/19018ed9429bb0247821883a876b10d9)。

## Install Harbor with Kubernetes locally
### 環境說明：
使用 Kubeadm 安裝在一台 Node 上，該台 Node 身兼 Master 與 Worker 。
使用 Kubernetes 安裝 Harbor 在該台 node 上。
該台 Node 的 IP address 是 10.0.2.15。
本次安裝皆是參考[kubernetes_deployment](https://github.com/vmware/harbor/blob/master/docs/kubernetes_deployment.md) 來進行的。
在這次的實作當中，並不採用 443 port 進行，且並沒有使用到 Ingress 。(使用的方式為 NodePort)

### 安裝紀錄
#### 1. 下載 Harbor v1.4.0

```sh
$ wget https://github.com/vmware/harbor/archive/v1.4.0.tar.gz
$ tar xvf v1.4.0.tar.gz
$ cd harbor-1.4.0
```

#### 2. 設定 `/etc/hosts`

```sh
10.0.2.15  reg.mydomain.com
```

#### 3. 建立 `/data/storage`

因為 `*pv.yaml` 會用到本機的 Volume ，所以先幫忙建立資料夾
```sh
$ sudo mkdir -p /data/storage
```

#### 4. 產生 `ConfigMap files`

```sh
$ python make/kubernetes/k8s-prepare
```
產生以下的檔案
```sh
make/kubernetes/adminserver/adminserver.cm.yaml
make/kubernetes/ingress.yaml
make/kubernetes/jobservice/jobservice.cm.yaml
make/kubernetes/mysql/mysql.cm.yaml
make/kubernetes/registry/registry.cm.yaml
make/kubernetes/ui/ui.cm.yaml
```

#### 5. 編輯 `make/kubernetes/**/*.svc.yaml` 檔案
將每個 `*.svc.yaml` 檔案加入 NodePort 的 type。因為部署後我們要用 NodePort 來 access service 。
- `make/kubernetes/ui/ui.svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30020
  selector:
    name: ui-apps
```
- `make/kubernetes/registry/registry.svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry
spec:
  type: NodePort
  ports:
    - name: repo
      port: 5000
      nodePort: 30010
    - name: debug
      port: 5001
  selector:
    name: registry-apps
```
- `make/kubernetes/jobservice/jobservice.svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jobservice
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30012
  selector:
    name: jobservice-apps
```
- `make/kubernetes/adminserver/adminserver.svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: adminserver
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30014
  selector:
    name: adminserver-apps
```
#### 6. 部署
- create pv & pvc
```sh
kubectl apply -f make/kubernetes/pv/log.pv.yaml
kubectl apply -f make/kubernetes/pv/registry.pv.yaml
kubectl apply -f make/kubernetes/pv/storage.pv.yaml
kubectl apply -f make/kubernetes/pv/log.pvc.yaml
kubectl apply -f make/kubernetes/pv/registry.pvc.yaml
kubectl apply -f make/kubernetes/pv/storage.pvc.yaml
```
- create config map
```sh
kubectl apply -f make/kubernetes/jobservice/jobservice.cm.yaml
kubectl apply -f make/kubernetes/mysql/mysql.cm.yaml
kubectl apply -f make/kubernetes/registry/registry.cm.yaml
kubectl apply -f make/kubernetes/ui/ui.cm.yaml
kubectl apply -f make/kubernetes/adminserver/adminserver.cm.yaml
```
- create service
```sh
kubectl apply -f make/kubernetes/jobservice/jobservice.svc.yaml
kubectl apply -f make/kubernetes/mysql/mysql.svc.yaml
kubectl apply -f make/kubernetes/registry/registry.svc.yaml
kubectl apply -f make/kubernetes/ui/ui.svc.yaml
kubectl apply -f make/kubernetes/adminserver/adminserver.svc.yaml
```
- create k8s deployment
```sh
kubectl apply -f make/kubernetes/registry/registry.deploy.yaml
kubectl apply -f make/kubernetes/mysql/mysql.deploy.yaml
kubectl apply -f make/kubernetes/jobservice/jobservice.deploy.yaml
kubectl apply -f make/kubernetes/ui/ui.deploy.yaml
kubectl apply -f make/kubernetes/adminserver/adminserver.deploy.yaml
```
- create k8s ingress
```sh
kubectl apply -f make/kubernetes/ingress.yaml
```

#### 7. 確認所有資訊
```
$ kubectl get pv,pvc,cm,svc,deploy,pod,ing -o wide
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
pv/log-pv        5Gi        RWO            Retain           Bound     default/log-pvc                                 41m
pv/registry-pv   100Gi      RWO            Retain           Bound     default/registry-pvc                            41m
pv/storage-pv    5Gi        RWO            Retain           Bound     default/storage-pvc                             41m

NAME               STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc/log-pvc        Bound     log-pv        5Gi        RWO                           41m
pvc/registry-pvc   Bound     registry-pv   100Gi      RWO                           41m
pvc/storage-pvc    Bound     storage-pv    5Gi        RWO                           41m

NAME                           DATA      AGE
cm/harbor-adminserver-config   42        41m
cm/harbor-jobservice-config    8         41m
cm/harbor-mysql-config         1         41m
cm/harbor-registry-config      2         41m
cm/harbor-ui-config            8         41m

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE       SELECTOR
svc/adminserver   NodePort    10.111.208.66    <none>        80:30014/TCP                    41m       name=adminserver-apps
svc/jobservice    NodePort    10.98.117.6      <none>        80:30012/TCP                    41m       name=jobservice-apps
svc/kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP                         3d        <none>
svc/mysql         ClusterIP   10.102.187.111   <none>        3306/TCP                        41m       name=mysql-apps
svc/registry      NodePort    10.110.64.93     <none>        5000:30010/TCP,5001:31336/TCP   41m       name=registry-apps
svc/ui            NodePort    10.102.161.35    <none>        80:30020/TCP                    41m       name=ui-apps

NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS        IMAGES                             SELECTOR
deploy/adminserver   1         1         1            1           40m       adminserver-app   vmware/harbor-adminserver:v1.2.0   name=adminserver-apps
deploy/jobservice    1         1         1            1           40m       jobservice-app    vmware/harbor-jobservice:v1.2.0    name=jobservice-apps
deploy/mysql         1         1         1            1           40m       mysql-app         vmware/harbor-db:v1.2.0            name=mysql-apps
deploy/registry      1         1         1            1           40m       registry-app      vmware/registry:2.6.2-photon       name=registry-apps
deploy/ui            1         1         1            1           40m       ui-app            vmware/harbor-ui:v1.2.0            name=ui-apps

NAME                              READY     STATUS    RESTARTS   AGE       IP            NODE
po/adminserver-6b4874b64d-xl2qw   1/1       Running   0          40m       10.244.0.42   sufuf3149spc
po/jobservice-69659fb46c-lxw5z    1/1       Running   1          40m       10.244.0.41   sufuf3149spc
po/mysql-69f5b99856-wqbd4         1/1       Running   0          40m       10.244.0.40   sufuf3149spc
po/registry-cd965df89-9dbsd       1/1       Running   0          40m       10.244.0.39   sufuf3149spc
po/ui-7dcfc78df7-9chtg            1/1       Running   1          40m       10.244.0.43   sufuf3149spc

NAME         HOSTS              ADDRESS   PORTS     AGE
ing/harbor   reg.mydomain.com             80        40m
```

#### 8. 將 ui 的 service cluster IP 加到 `/etc/hosts`
我們獲得 `svc/ui` 的資訊如下:
```
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE       SELECTOR
svc/ui            NodePort    10.102.161.35    <none>        80:30020/TCP                    41m       name=ui-apps
```
cluster IP 為 `10.110.64.93`，再次加入下列的資訊到 `/etc/hosts`
```sh
10.102.161.35   reg.mydomain.com
```
為何要將 svc/ui 的 cluster IP 加到 `/etc/hosts` 中呢？
因為在嘗試 `docker login reg.mydomain.com:30010` 時，會轉跳到 `http://reg.mydomain.com/service...`，這邊它要 access 的是 ui 的 URL，但是 `reg.mydomain.com` 會解析到 Node 的 IP，所以只好在幫 service ui 的 cluster IP 在 `/etc/hosts` 再加一條對應，這樣解析到的資訊就會是對的 IP。而這條規則我是放在 `10.0.2.15  reg.mydomain.com` 的上方。兩條在 `/etc/hosts` 中的規則設定如下：
```sh
10.102.161.35   reg.mydomain.com
10.0.2.15   reg.mydomain.com
```

#### 9. 設定 Docker 的 daemon.json
編輯 `/etc/docker/daemon.json` (如果沒有該檔案，就直接新增這個檔案吧！)
```json
{
    "registry-mirrors": ["reg.mydomain.com:30010"],
    "insecure-registries": ["reg.mydomain.com:30010"]
}
```
然後最重要的！！！
```sh
$ sudo service docker restart
```

Memo: 至於為何要編輯 daemon.json 檔案呢？
因為如果要使用 docker login 到我們的 registry ，現在都是要使用 TLS certificate ，然而我們在本次的設定中都沒有加任何的 TLS certificate ，所以要來設定 docker login 可信任的 registry 位址，讓我們的 registry 可以透過 insecure 的方式來得到驗證。
詳細可參考：https://docs.docker.com/registry/insecure/

#### 10. 來 access 網頁吧！
您可以在網址列使用 `http://reg.mydomain.com:30010` 或是 ui 的 service cluster IP `10.102.161.35` 來 access，帳號：`admin`；密碼：`Harbor12345` (這邊您可以在 `make/harbor.cfg` 中來設定，在使用 python 來重新產生 yaml 檔案，這組是官方預設的。)

#### 11. 來上傳一個 image 到我們的 registry 吧！
- 登入我們的 registry
```sh
$ sudo docker login reg.mydomain.com:30010 -u admin -p Harbor12345
Login Succeeded
```
- 看一下我們有哪些 docker image
```sh
$ sudo docker images
REPOSITORY                                                       TAG                 IMAGE ID            CREATED             SIZE
quay.io/kubernetes-ingress-controller/nginx-ingress-controller   0.11.0              7f6aa8354f0c        2 weeks ago         211 MB
nginx                                                            latest              e548f1a579cf        2 weeks ago         109 MB
gcr.io/google_containers/kube-apiserver-amd64                    v1.9.3              360d55f91cbf        4 weeks ago         210 MB
gcr.io/google_containers/kube-scheduler-amd64                    v1.9.3              d3534b539b76        4 weeks ago         62.7 MB
gcr.io/google_containers/kube-proxy-amd64                        v1.9.3              35fdc6da5fd8        4 weeks ago         109 MB
gcr.io/google_containers/kube-controller-manager-amd64           v1.9.3              83dbda6ee810        4 weeks ago         138 MB
quay.io/coreos/flannel                                           v0.10.0-amd64       f0fad859c909        6 weeks ago         44.6 MB
gcr.io/google_containers/etcd-amd64                              3.1.11              59d36f27cceb        3 months ago        194 MB
vmware/registry                                                  2.6.2-photon        c38af846a0da        3 months ago        240 MB
gcr.io/google_containers/k8s-dns-sidecar-amd64                   1.14.7              db76ee297b85        4 months ago        42 MB
gcr.io/google_containers/k8s-dns-kube-dns-amd64                  1.14.7              5d049a8c4eec        4 months ago        50.3 MB
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64             1.14.7              5feec37454f4        4 months ago        41 MB
vmware/harbor-jobservice                                         v1.2.0              1fb18427db11        6 months ago        164 MB
vmware/harbor-ui                                                 v1.2.0              b7069ac3bd4b        6 months ago        178 MB
vmware/harbor-adminserver                                        v1.2.0              a18331f0c1ae        6 months ago        142 MB
vmware/harbor-db                                                 v1.2.0              deb8033b1c86        6 months ago        329 MB
gcr.io/google_containers/pause-amd64                             3.0                 99e59f495ffa        22 months ago       747 kB
```
我們有 nginx 的 image，就拿它來上傳吧！
- 先來 tag 我們的 nginx image
```
$ sudo docker tag nginx:latest reg.mydomain.com:30010/library/nginx:latest
```
- 來上傳 nginx image 拉！
```sh
$ sudo docker push reg.mydomain.com:30010/library/nginx:latest
The push refers to a repository [reg.mydomain.com:30010/library/nginx]
e89b70d28795: Pushed
832a3ae4ac84: Pushed
014cf8bfcb2d: Pushed
latest: digest: sha256:85659faba67a762e1669668a4ac30f4359e094a223a791354ffe51e23ad3a226 size: 948
```
- 到網頁看看我們的 image 的上傳結果
![](https://i.imgur.com/Dmwevwo.png)

That's all.

註：其實 docker 前面不用加 sudo，只不過這次的安裝比較趕，所以沒有重新在我的本機 logout 就直接使用，要不然是要在安裝完 docker 後，下 `sudo usermod -aG docker <username>` 的指令，然後 logout。再重新 login。就可以在你目前的 username 直接使用 docker 的指令。至於 `sudo usermod -aG docker <username>` 的意思是，修改 username 這個使用者，將 username 加到 docker 這個 group (-g) 中。

