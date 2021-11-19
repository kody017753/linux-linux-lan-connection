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
