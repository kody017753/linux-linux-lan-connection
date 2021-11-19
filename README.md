# linux機間のLANケーブルを使用した接続と設定

## ドライバのダウンロードとビルド
### ※NIC(有線LAN)が認識されていないと思い実施した手順のため、必要ないかも？

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

## IPを設定する

1. UbuntuのSettingからNetworkを選択
2. Ethernetから未使用の場所を選択
3. 歯車マーク→IPv4を選択
4. Manualにチェックをし「Address」に好きなIP（例）192.168.10.20　「Netmask」に255.255.255.0（固定？）、「Gateway」（例）192.168.10.1（Addressの末尾を1にするだけ？） を設定
5. Applyする




## IPを設定したPCとは別のPCにコマンドラインでUbuntuを固定IPにする

1. 設定ファイルを追加
/etc/netplan/99_config.yaml というファイルを新規作成し、その中に以下の内容を書き込む。

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
```
ip a | grep -E '192\.168\.[0-9]*\.[0-9]*|172\.(1[6-9]|2[0-9]|3[0-1])\.[0-9]*\.[0-9]*|10\.[0-9]*\.[0-9]*\.[0-9]*' -B 10
```

長い grep は IPv4 のプライベートアドレスとなりうる値を正規表現で全部指定しています。

すると、以下のように表示されます。

```
... (無視して OK)
...
...
...
...
...
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.14/24 brd 192.168.3.255 scope global dynamic eth0
```

grep が引っかかった部分 (上記の例だと 192.168.3.14 と 192.168.3.255) に該当するインターフェース名 (上記の例だと eth0 の部分) を <NETWORK_INTERFACE_NAME> に書きます。

### 固定 IP アドレスとネットマスク
IP設定で決めた「Address」の最後に「/24」を加えて入力

### デフォルトゲートウェイ
IP設定で決めた「Gateway」をそのまま入力

### DNS サーバ
DNS サーバの IP アドレスをカンマ区切りで複数指定できます。

よくわからない場合はとりあえず 8.8.8.8, 8.8.4.4 を指定しておけば Oｋ

### 完成例
```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.10.20/24
      gateway4: 192.168.10.1
      nameservers:
          addresses: [8.8.8.8, 8.8.4.4]
```
