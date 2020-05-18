# k8s

## k8s install
**v1.15を使用**

### install k8s dependencies (ansible)
```
- name: "install docker dependencies"
  apt:
    name: "{{ packages }}"
    update_cache: yes
  vars:
    packages:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - software-properties-common

- name: "Add an apt key for docker"
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: "add apt repo for docker"
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
    state: present

- name: "install docker"
  apt:
    name: "{{ packages }}"
    update_cache: yes
  vars:
    packages:
      - docker-ce
      - docker-ce-cli 
      - containerd.io

- name: add Kubernetes apt-key
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: add Kubernetes' APT repository
  apt_repository:
   repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
   state: present
   filename: 'kubernetes'

- name: "install kubeadm"
  apt:
    name: "{{ packages }}"
    state: present
    update_cache: yes
  vars:
    packages:
      - kubelet
      - kubeadm
      - kubectl

- name: Remove swapfile from /etc/fstab #swapを消しておかないと失敗する
  mount:
    name: swap
    fstype: swap
    state: absent

- name: Disable swap
  command: swapoff -a
  when: ansible_swaptotal_mb > 0

- name: Set docker service to start on boot. #再起動後もdockerを自動起動させる
  service: name=docker enabled=yes
- name: Set kubelet service to start on boot. #再起動後もk8sを自動起動させる
  service: name=kubelet enabled=yes
```

### warning: cgroupsfs
* /etc/docker/daemon.json
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

```
sudo systemctl daemon-reload
sudo systemctl restart docker
``` 

### k8s setup
```
# master
kubeadm init --apiserver-advertise-address 172.16.0.11 --pod-network-cidr 10.244.0.0/16
```

## metallb bgp
example:

* master: 10.0.0.11
* node01: 10.0.0.12
* node02: 10.0.0.13
  
### metallb 適用
`kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml`

### metallb config
* bgp
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - peer-address: 10.0.0.1
      peer-asn: 65000
      my-asn: 65001
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 192.168.10.0/24
```

`kubectl apply -f config.yaml`

* bgp routerの設定(FRRにて)
```
!
frr version 7.2
frr defaults traditional
hostname router01
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 10.0.0.11 remote-as 65001
 neighbor 10.0.0.12 remote-as 65001
 neighbor 10.0.0.13 remote-as 65001
 !
 address-family ipv4 unicast
  network 192.168.11.0/24
 exit-address-family
!
line vty
!
end
```

## gitlabとの連携
1. configure gitlab > kubernetesに移動し、`Add Kubernetes cluster`をクリック
2. 設定値を入力
* API URL
  `APIURL=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")`
* CA
```
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-xxxxx   kubernetes.io/service-account-token   3      7h18m

kubectl get secret default-token-xxxxx -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

* Token
  * `gitlab-admin.yaml`
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab-admin
    namespace: kube-system
```

```
kubectl apply -f gitlab-admin.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab-admin | awk '{print $1}')
```

3. Helm Tillerインストールをクリック
4. GitLab Runnerインストールをクリック


refs
[gitlab runner連携](https://qiita.com/khk-h/items/9e7a4a201654b588a493)

## ingress-nginx + metallb
### install 
1. ingress-nginx-controller インストール
`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml`
2. `curl -L -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml`
3. loadBalancerIP: <metallbで設定した範囲のアドレス>をspecの下に追記
4. `kubectl apply -f cloud-generic.yaml`

### ingress nginx test app
```
# cat <<EOF > hello-world.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: hello
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - image: gcr.io/hello-minikube-zero-install/hello-node
        name: hello-world
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: hello-world
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-router
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: hello-world
          servicePort: 8080
    host: hello-world.kubernetes.local
EOF
```

```
kubectl apply -f hello-world.yaml
curl http://hello-world.kubernetes.local
```

refs:
* https://blog.web-apps.tech/ingress-nginx-on-prem/
* https://qiita.com/megasys1968/items/aa94ddfb92f57e08c4dd

## prometheus (ingress nginx)
* refsを参考に構築したあとに以下をやることで、`http://prometheus.kubernetes.local`でPrometheusが見える

* prometheus-ingress.yaml
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-server
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: prometheus-service
          servicePort: 9090
    host: prometheus.kubernetes.local
```

refs:
* https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/
* https://linuxacademy.com/blog/kubernetes/running-prometheus-on-kubernetes/