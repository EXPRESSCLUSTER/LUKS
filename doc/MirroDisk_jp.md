# LUKS で暗号化されたミラーディスククラスタの構築

## 評価環境
- MIRACLE LINUX 8.4 (4.18.0-305.25.1.el8_4.x86_64)
- CLUSTERPRO X 4.3 (4.3.2-1)
- 暗号化したデバイスを、ミラーディスクリソースのクラスタパーティションおよびデータパーティションとして使用します。
  ```
  +--------------------------------------------------+
  | server1, 2                                       |
  |   sdb                                            |
  |    |                                             |
  |    +-- sdb1                                      |
  |         |                                        |
  |         +-- encrypted_cp1: For Cluster Partition |
  |                                                  |
  |   sdc                                            |
  |    |                                             |
  |    +-- sdc1                                      |
  |         |                                        |
  |         +-- encrypted_dp1: For Data Partition    |
  +--------------------------------------------------+
  ```

## LUKS でディスクを暗号化する
- クラスタを構成する全てのサーバで以降のコマンドを実行してください。
1. 以下のコマンドを実行し、クラスタパーティションおよびデータパーティションのデバイスを LUKS 用に初期化します。
   ```sh
   cryptsetup --cipher aes-xts-plain --key-size 256 luksFormat /dev/sdb1
   cryptsetup --cipher aes-xts-plain --key-size 256 luksFormat /dev/sdc1
   ```
1. 以下のコマンドを実行し、キーファイルを追加してください。
   ```sh
   mkdir /etc/luks
   dd if=/dev/urandom of=/etc/luks/encrypted_sdb1.key bs=1 count=32
   dd if=/dev/urandom of=/etc/luks/encrypted_sdc1.key bs=1 count=32
   cryptsetup luksAddKey /dev/sdb1 /etc/luks/encrypted_sdb1.key
   cryptsetup luksAddKey /dev/sdc1 /etc/luks/encrypted_sdc1.key
   ```
1. 以下のコマンドを実行してください。
   ```sh
   cryptsetup luksOpen /dev/sdb1 encrypted_cp1 --key-file /etc/luks/encrypted_sdb1.key
   cryptsetup luksOpen /dev/sdc1 encrypted_dp1 --key-file /etc/luks/encrypted_sdc1.key
   ```   
1. 上記コマンド実行後、lsblk を実行し、encrypted_cp1、encrypted_dp1 が認識されていることを確認してください。
   ```sh
   sdb                 8:48   0    2G  0 disk
     sdb1              8:49   0    2G  0 part
       encrypted_cp1 253:8    0    2G  0 crypt
   sdc                 8:64   0    4G  0 disk
     sdc1              8:65   0    4G  0 part
       encrypted_dp1 253:7    0    4G  0 crypt   
   ```
1. /etc/crypttab に以下を追記してください。追記することで、OS 起動時に対象のデバイスが復号化されます。
   ```
   encrypted_cp1 /dev/sdb1 /etc/luks/encrypted_sdb1.key luks
   encrypted_dp1 /dev/sdc1 /etc/luks/encrypted_sdc1.key luks
   ```
   - /dev/sdb1, /dev/sdc1 については、by-id (/dev/disk/by-id/wwn-0x********************************-part1) を指定することも可能です。
1. OS を再起動し、再起動後、encrypted_cp1、encrypted_dp1 が認識されていることを確認してください。

## CLUSTERPRO のインストール
1. [インストール & 設定ガイド](https://docs.nec.co.jp/sites/default/files/minisite/static/7046aab7-c76f-436d-b513-53b9a20df485/clp_x43_linux/L43_IG_JP/index.html) に従い、CLUSTERPRO をインストールしてください。

## クラスタの構築
1. ミラーディスクリソース設定時に、データパーティションデバイス名、クラスタパーティションデバイス名に以下を設定してください。
   - データパーティションデバイス名: /dev/mapper/encrypted_dp1
   - クラスタパーティションデバイス名: /dev/mapper/encrypted_cp1

   

