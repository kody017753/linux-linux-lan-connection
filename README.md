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
