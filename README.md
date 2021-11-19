# linux機間のLANケーブルを使用した接続と設定

## ドライバのダウンロードとビルド 
※NIC(有線LAN)が認識されていないと思い実施した手順のため、必要ないかも？

1.  ビルドの環境をインストール
```bash
sudo apt install make
sudo apt install gcc
```

2. NICの種類を確認(用途不明のため無視でも)
```bash
lspci | grep Ethernet
```

3. 以下のサイトからドライバをダウンロード
 https://downloadcenter.intel.com/download/15817/Intel-Network-Adapter-Driver-for-PCIe-Intel-Gigabit-Ethernet-Network-Connections-Under-Linux-

4. このままビルドしても、Checksum エラーで引っかかります。ソースコード ./e1000e-3.8.4/src/nvm.c を一部修正

before:
```
s32 e1000e_validate_nvm_checksum_generic(struct e1000_hw *hw)
{
        s32 ret_val;
        u16 checksum = 0;
        u16 i, nvm_data;

        for (i = 0; i < (NVM_CHECKSUM_REG + 1); i++) {
                ret_val = e1000_read_nvm(hw, i, 1, &nvm_data);
                if (ret_val) {
                        e_dbg("NVM Read Error\n"); 
                        return ret_val;
                }       
                checksum += nvm_data;
        }       

        if (checksum != (u16)NVM_SUM) {
                e_dbg("NVM Checksum Invalid\n");
                return -E1000_ERR_NVM;
        }       

        return 0;
}
```

after:
```
s32 e1000e_validate_nvm_checksum_generic(struct e1000_hw *hw)
{
        return 0;
} 
```

5. ビルドする
```
sudo make install
```

6. カーネルモジュール追加
```
sudo modprobe -r e1000e
sudo modprobe e1000e
```

7. ログを確認
```
dmesg | grep -i e1000
```

8.
このままPCを再起動すると、古いドライバが読み込まれてしまうので、新しいドライバが読み込まれるように固定化
```
sudo update-initramfs -u
```
### 参考文献
https://qiita.com/matsuo7005/items/b0037b6450d752378316

## IPを設定する

1. LinuxのSettingからNetworkを選択
2. Ethernetから未使用の場所を選択
3. 歯車マーク→IPv4を選択
4. Manualにチェックをし「Address」に好きなIP（例）192.168.10.20　「Netmask」に255.255.255.0（固定？）、「Gateway」（例）192.168.10.1（Addressの末尾を1にするだけ？） を設定
5. Applyする


## IPを設定したPCとは別のPCにコマンドラインでUbuntuを固定IPにする

1. 設定ファイルを追加
/etc/netplan/01-network-manager-all.yaml というファイルの中に以下の内容を書き込む。

(.yamlファイルはOSによって？名前が変わるようだが.yamlファイルに書き込めればOK)

```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    <NETWORK_INTERFACE_NAME>:
      addresses:
        - <STATIC_IP_ADDR_WITH_NETMASK>
      gateway4: <DEFAULT_GATEWAY>
      nameservers:
          addresses: [<DNS, DNS, ...>]
```
<> で囲まれている部分はそれぞれ以下の環境に置き換えます。
|  変数  |  説明  |  例  |
| ---- | ---- | ---- |
|  <NETWORK_INTERFACE_NAME>  |  ネットワークインターフェース名  |  eth0  |
|  <STATIC_IP_ADDR_WITH_NETMASK>  |  固定 IP アドレスとネットマスク  |  192.168.3.2/24  |
|  <DEFAULT_GATEWAY>  |  	デフォルトゲートウェイ  |  192.168.3.1  |
|  <DNS, DNS, ...>  |  DNS サーバ (複数ある場合はカンマ区切り)  |  8.8.8.8, 8.8.4.4  |

### ネットワークインターフェース名
ネットワークインターフェース名は以下のコマンドで調べられます。
```bash
ip a | grep -E '192\.168\.[0-9]*\.[0-9]*|172\.(1[6-9]|2[0-9]|3[0-1])\.[0-9]*\.[0-9]*|10\.[0-9]*\.[0-9]*\.[0-9]*' -B 10
```

長い grep は IPv4 のプライベートアドレスとなりうる値を正規表現で全部指定しています。

すると、以下のように表示されます。

```bash
 ip a | grep -E '192\.168\.[0-9]*\.[0-9]*|172\.(1[6-9]|2[0-9]|3[0-1])\.[0-9]*\.[0-9]*|10\.[0-9]*\.[0-9]*\.[0-9]*' -B 10
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp7s0f1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether 80:fa:5b:77:95:3c brd ff:ff:ff:ff:ff:ff
    inet6 fe80::82fa:5bff:fe77:953c/64 scope link 
       valid_lft forever preferred_lft forever
3: wlp0s20f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ac:67:5d:5f:3c:eb brd ff:ff:ff:ff:ff:ff
    inet 192.168.88.50/24 brd 192.168.88.255 scope global dynamic noprefixroute wlp0s20f3
       valid_lft 257743sec preferred_lft 257743sec
    inet6 fe80::f5c1:f461:ac30:7e06/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:3d:05:ef:ae brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0

```

grep が引っかかった部分に該当するインターフェース名 (上記の例だと enp7s0f1 の部分) を <NETWORK_INTERFACE_NAME> に書きます。

### 固定 IP アドレスとネットマスク
IP設定で決めた「Address」の最後に「/24」を加えて入力

### デフォルトゲートウェイ
IP設定で決めた「Gateway」をそのまま入力

### DNS サーバ
DNS サーバの IP アドレスをカンマ区切りで複数指定できます。

よくわからない場合はとりあえず 8.8.8.8, 8.8.4.4 を指定しておけば ok

### 完成例
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp7s0f1:
      addresses:
        - 192.168.10.20/24
      gateway4: 192.168.10.1
      nameservers: 　#なくても可
          addresses: [8.8.8.8, 8.8.4.4]　　#なくても可
          
```

### 適用
上記のファイルを保存したら、以下のコマンドを実行
```bash
sudo netplan apply
```

これで固定 IP アドレスになったはずなので確認する。

```bash
ip a  # ifconfig でも可
※この際、機器間をLANケーブルで接続していないと表示されないため注意
```

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp7s0f1enp7s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 80:fa:5b:77:95:3c brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.20/24 brd 192.168.10.255 scope global enp7s0f1
       valid_lft forever preferred_lft forever
    inet6 fe80::82fa:5bff:fe77:953c/64 scope link 
       valid_lft forever preferred_lft forever
3: wlp0s20f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ac:67:5d:5f:3c:eb brd ff:ff:ff:ff:ff:ff
    inet 192.168.88.50/24 brd 192.168.88.255 scope global dynamic noprefixroute wlp0s20f3
       valid_lft 259150sec preferred_lft 259150sec
    inet6 fe80::f5c1:f461:ac30:7e06/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:9c:aa:ac:2a brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

```

自分が指定したインターフェース名とIPアドレスになっていればok

### 参考文献
https://qiita.com/noraworld/items/3e232fb7a25ed16c6a63
