# Server
## GrafanaLoki
![architecture](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/12998465/picture_pc_e512d6bbc17a1038e4ff0f264b9704bc.png)

* Grafana Loki
  * Logを収集する
* Promtail
  * Agent
  * Lokiに対してログを転送
* Grafana
  * データを可視化

### Install (Loki + Promtail)
* Grafana Loki
```
curl -fSL -o "/usr/local/bin/loki.gz" "https://github.com/grafana/loki/releases/download/v0.4.0/loki-linux-amd64.gz"
gunzip "/usr/local/bin/loki.gz"
chmod a+x "/usr/local/bin/loki"
```

* `/etc/loki/loki-config.yaml`
```
auth_enabled: false
server:
http_listen_port: 3100
ingester:
lifecycler:
    address: 127.0.0.1
    ring:
    kvstore:
        store: inmemory
    replication_factor: 1
    final_sleep: 0s
chunk_idle_period: 5m
chunk_retain_period: 30s
schema_config:
configs:
- from: 2019-10-30
    store: boltdb
    object_store: filesystem
    schema: v9
    index:
    prefix: index_
    period: 168h
storage_config:
boltdb:
    directory: /tmp/loki/index
filesystem:
    directory: /tmp/loki/chunks
limits_config:
enforce_metric_name: false
reject_old_samples: true
reject_old_samples_max_age: 168h
chunk_store_config:
max_look_back_period: 0
table_manager:
chunk_tables_provisioning:
    inactive_read_throughput: 0
    inactive_write_throughput: 0
    provisioned_read_throughput: 0
    provisioned_write_throughput: 0
index_tables_provisioning:
    inactive_read_throughput: 0
    inactive_write_throughput: 0
    provisioned_read_throughput: 0
    provisioned_write_throughput: 0
retention_deletes_enabled: false
retention_period: 0
```

* protail(ansibleでインストール)
    * `roles/promtail/tasks/main.yml`
```
- name: Download promtail binary tlocal folder
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.gz"
    dest: "/usr/local/bin/promtail.gz"
  retries: 5
  delay: 2
  when: ansible_architecture ="x86_64"

- name: Download node_exportebinary to local folder for armv7
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-arm64.gz"
    dest: "/usr/local/bin/promtail.gz"
  retries: 5
  delay: 2
  when: ansible_architecture ="armv7l"

- name: check promtail
  stat:
    path: /usr/local/bin/promtail
  register: promtail_file

- name: Unpack promtail binary
  shell: gunzip /usr/local/bipromtail.gz
  when:
    - not promtail_file.stat.exists

- name: change promtail mode
  file:
    path: /usr/local/bin/promtail
    owner: root
    group: root
    mode: 0755
  when:
    - not promtail_file.stat.exists

- name: Copy the promtail systemservice file
  copy:
    src: promtail.service
    dest: /etc/systemd/system/promtail.service
    owner: root
    group: root
    mode: 0644

- name: Create /etc/promtail
  file:
    path: /etc/promtail
    state: directory
    mode: 0755

- name: Copy the promtail confifile
  copy:
    src: promtail-config.yaml
    dest: /etc/promtail/promtail-config.yaml
    owner: root
    group: root
    mode: 0644

- name: replace    ipaddresnode_exporter.service
  replace:
    path: /etc/promtail/promtail-config.yaml
    regexp: 'hostname: hostname'
    replace: 'hostname: {{ansible_hostname}}'

- name: Enable my service
  systemd:
    name: promtail.service
    enabled: yes
    state: restarted
    daemon_reload: yes
```
    
* `promtail-config.yaml`
    * hostnameを追加することで、hostごとのlogをフィルターできる
```
server:
http_listen_port: 9080
grpc_listen_port: 0

positions:
filename: /tmp/positions.yaml

clients:
- url: http:monitor.node.kanai.lab:3100/lokapi/v1/push

scrape_configs:
- job_name: system
static_configs:
- targets:
    - localhost
    labels:
    job: varlogs
    __path__: /var/log/*log
    hostname: hostname
```

* `promtail.service`
```
[Unit]
Description=promtail

[Service]
Type=simple
ExecStart=/usr/local/bin/promtail \
    -config.file /etc/promtail/promtail-config.yaml

[Install]
WantedBy=multi-user.target
```

### 参考
[note](https://note.mu/_k_e_k_e/n/n9bcfa4ef9278)

[promtail](https://i101330.hatenablog.com/entry/2019/08/18/142339)

[github loki](https://github.com/grafana/loki)


## sflow-rt + Vizceral
### sflow-rt
* ネットワークデバイス、ホスト、およびアプリケーションに埋め込まれたsFlowエージェントからsflowの情報を集計し、監視系ソフトウェアはsflow-rtのREST APIを叩くことでsflowの値を監視できる
  ![sflow-rt](https://sflow-rt.com/img/rt-ecosystem.png)

### hsflowd
* Host Sflow
  * sFlowプロトコルを使用して物理サーバーと仮想サーバーのパフォーマンスメトリックをエクスポート

* hsflowdをansibleで複数マシンをインストール (ubuntu 1804)
  *  `roles/sflow/tasks/main.yml`

```
- name: Download hsflowd deb package
  get_url:
    url: "https://github.com/sflow/host-sflow/releases/download/v{{ hsflowd_version }}/hsflowd-ubuntu18_{{ hsflowd_version }}_amd64.deb"
    dest: "/tmp/hsflowd.deb"
    retries: 5
    delay: 2

    - name: Install hsflowd deb
    apt:
        deb: /tmp/hsflowd.deb

    - name: Copy the hsflowd config file
    copy:
        src: hsflowd.conf
        dest: /etc/hsflowd.conf
        owner: root
        group: root
        mode: 0644

    - name: Enable hsflowd service
      systemd:
        name: hsflowd.service
        enabled: yes
        state: restarted
        daemon_reload: yes
```

* `/etc/hsflowd/hsflowd.conf`

```
# hsflowd configuration file
# http://sflow.nehost-sflow-linux-config.php

sflow {
# ======  Agent IP selection ======
# Selection is automatic, unless:
# (1) override with preferred CIDR:
#   agent.cidr = 192.168.0.0/16
# (2) Override with interface:
#   agent = eth0

# ====== Sampling/PollinCollectors ======
# EITHER: automatic (DNS SRV+TXfrom _sflow._udp):
#   DNS-SD { }
  DNSSD = off
# OR: manual:
#   Counter Polling:
  polling = 20
#   default sampling N:
  sampling = 400
#   sampling N on interfaces witifSpeed:
#     sampling.100M = 100
#     sampling.1G = 1000
#     sampling.10G = 10000
#     sampling.40G = 40000
#   sampling N for apache, nginx:
#     sampling.http = 50
#   sampling N for applicatio(requires json):
#     sampling.app.myapp = 100
#   collectors:
  collector { ip=<sflow-rt iaddress> udpport=6343 }
#   add additional collectors here

# ====== Local configuration ======
# listen for JSON-encoded input:
#   json { UDPport = 36343 }
# PCAP+BPF packet-sampling:
#   Bridge example:
#     pcap { dev = docker0 }
#   NIC example:
#     pcap { dev = eth0 }
#     pcap { dev = eth1 }
#   All NICs example:
#     pcap { speed=1G-1T }
  pcap { speed=1- }
# NFLOG packet-sampling:
#   nflog { group = 5  probabilit= 0.0025 }
# ULOG packet-sampling:
#   ulog { group = 1  probability 0.0025 }
# PSAMPLE packet-sampling:
#   psample { group = 1 }
# Nvidia NVML GPU monitoring:
#   nvml { }
# Xen hypervisor and VM monitoring:
#   xen { }
# Open vSwitch sFlow configuration:
#   ovs { }
# KVM (libvirt) hypervisor and Vmonitoring:
#   kvm { }
# Docker container monitoring:
#   docker { }
# TCP round-trip-time/loss/jitte(requires pcap/nflog/ulog)
  tcp { }
# monitoring of systemd cgroups
#   systemd { }
# DBUS agent
#   dbus { }
# Learn config from Arista EAPI
#   eapi { }
}
```

### Vizceral
* Vizceral
  * NetflixのOSS
  * WebGL
  * webglキャンバスに交通データを表示するためのコンポーネントです
    * トラフィック量に関するデータを含むノードとエッジのグラフが提供される場合、ノード間の接続量をアニメーション化するトラフィックグラフをレンダリングする
![vizceral](https://github.com/Netflix/vizceral/raw/master/vizceral-example.png) 
  
* sflow-rt + vizceral
  * Netflix Vizceralを使用してリアルタイムのメトリックを表示するsFlow-RTアプリケーション

    ![sflowrt-vizceral](https://sflow-rt.com/img/vizceral-top.png)

  * ドットのストリームは、インターネットとの間のパケットフローを表す
    * ドットの色はパケットタイプを表す
      * tcp/udp: ブルー
      * icmp: イエロー
      * その他: レッド

* Docker
  * `Makefile`
```
NAME=sflow-rt docker
VERSION=0.0.1

run:
    docker run --name sflow-rt --restart=always -d -v $(PWD)/groups.json:/sflow-rt/store/vizceral~traffic.js/groups -p 6343:6343/udp -p 8008:8008 sflow/vizceral -Dviz.maxVolume=100000

clean:
    docker rm -f sflow-rt
```

  * `groups.json`
    * ここにsflowのip addressの範囲を書いて監視できる
    * vizceralの円のところを管理できる
```
{
    "INTERNET": ["0.0.0.0/0"],
    "SiteA": ["10.0.0.0/16"]
}
```
  
## freeipa
