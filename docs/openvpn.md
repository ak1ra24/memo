# OpenVPN
## Install
[install script](https://github.com/angristan/openvpn-install)ではこれをよく使ってる

## クライアント側のLANに接続できるようにしたい
1. `sudo ./openvpn-install.sh`を実行して、クライアントを追加
   * example) client file: client.ovpn
2. `sudo mkdir -p /etc/openvpn/ccd`
3. `/etc/openvpn/ccd/client`
   * ovpnのファイル名と名前を揃える
   * ファイル内に以下をクライアント側のLANを書く
  ```
  iroute 192.168.1.0 255.255.255.0
  ```
4. `/etc/openvpn/server.conf`
```
push "route 10.8.0.0 255.255.255.0"
push "route 192.168.1.0 255.255.255.0"
client-config-dir ccd
route 192.168.1.0 255.255.255.0
client-to-client
```

# refs
[archwiki openvpn](https://wiki.archlinux.jp/index.php/OpenVPN#.E3.82.AF.E3.83.A9.E3.82.A4.E3.82.A2.E3.83.B3.E3.83.88_LAN_.E3.82.92.E3.82.B5.E3.83.BC.E3.83.90.E3.83.BC.E3.81.AB.E6.8E.A5.E7.B6.9A)