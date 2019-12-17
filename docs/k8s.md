# k8s

## v1.17を使用

## install k8s dependencies
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

## warning: cgroupsfs
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

## k8s setup
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