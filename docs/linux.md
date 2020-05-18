# Linux
##Network
### iptables
#### iptables永続化
`sudo apt install iptables-persistent`

* iptabls保存と再読み込み
```
sudo /etc/init.d/netfilter-persistent save
sudo /etc/init.d/netfilter-persistent reload
```

#### NAT
* NATテーブルのルール一覧を表示
`iptables -t nat -n -L`

* Dynamic NAT
	* -s local ip address
	* -o wan側interface
`iptables -t nat -A POSTROUTING -s 192.168.11.0/24 -o eth0 -j MASQUERADE`

* Static NAT
```
iptables -t nat -A POSTROUTING -o ens39 -j SNAT --to-source 2.2.2.2
iptables -t nat -A PREROUTING  -i ens38 -j DNAT --to-destination 1.1.1.1
```

### sysctl
* ip forwarding 有効化
`net.ipv4.ip_forward = 1`

### VLAN
`ip link add link eth0 name eth0.10 type vlan id 10`

### linux bridge
* Linux Bridge でタグ VLAN を扱うには vlan_filtering を有効にする必要
`ip link add dev br0 type bridge vlan_filtering 1`

### vrf
```
ip link add dev vrf-x type vrf table 10
ip link set dev vrf-x up
ip link set dev eth0 master vrf-x
ip route add 172.16.100.1/32 via 192.168.11.1 table 10 
ip route list vrf vrf-x
```


## Refs
### iptables
* https://qiita.com/yas-nyan/items/e5500cf67236d11cce72
* https://qiita.com/hana_shin/items/956dfaca4539ba257c16
* http://www.ksknet.net/cat13/iptables_1.html
* https://blog.nhiroki.net/2013/12/06/iptables-linux-router
* https://qiita.com/Shakapon/items/d29b0af036bf6796feb2
### VLAN
* https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/networking_guide/sec-configure_802_1q_vlan_tagging_using_the_command_line
### linux bridge
* https://blog.amedama.jp/entry/linux-bridge-8021q-vlan
### vrf
* http://yunazuno.hatenablog.com/entry/2017/06/17/225917
