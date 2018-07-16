# 環境

パソコン

# 準備

## imgファイルのダウンロード
本家は遅いので、jaistのミラーから
http://ftp.jaist.ac.jp/pub/raspberrypi/raspbian_lite/images/

## local networkの状況確認
```shell-session
$ ifconfig  # Ubuntu 16.04
$ ip a  # Debian 9
```
## 設定ファイル(WiFi, source.list)と公開鍵の作成

- WiFi: wpa_supplicant_confを作成
- raspi-configを使っても良いはずだが、打ち間違え防止に...

```shell-session:wpa_supplicant_confの作成
$ echo "# /etc/wpa_supplicant/wpa_supplicant.conf" > wpa_supplicant_conf  # pathのメモ（なくても良い）
$ wpa_passphrase "SSID" "PASSPHRASE" >> wpa_supplicant_conf
```

- source.list: source_listを作成

```bash:sources_list
# /etc/apt/sources.list

deb http://ftp.jaist.ac.jp/raspbian stretch main contrib non-free
deb http://ftp.tsukuba.wide.ad.jp/Linux/raspbian/raspbian/ stretch main contrib non-free
deb http://ftp.yz.yamagata-u.ac.jp/pub/linux/raspbian/raspbian/ stretch main contrib non-free
```

- 公開鍵の作成

```shell-session
$ ssh-keygen
```
ssh-keygenコマンドの詳細は省略する

# インストール
## imgファイルのインストール
ddコマンドでインストールしてもよいが、~~コマンド打つのめんどい~~諸事情により、Disk Image Writerを使用する。
- imgファイルを右クリック
- Open With Disk Image Writerをクリック
- DestinationでSDカードを選択
- あとは、Startを押して、パスワードを入力して、待つだけ

## sshロックの解除
（最初からssh接続をする場合のみ）
SDカードの/bootにsshという空ファイルを作成する

```shell-session
$ touch ssh
```
右クリックでCreate Documentして、Renameして作成しても良い

## 設定ファイル(WiFi, source.list)と公開鍵のコピー
- 設定ファイル(wpa_supplicant_conf, source_list)と公開鍵を/home/pi/DIRECTORYのコピーする (DIRECTORYは任意)
- ID_RSA.pub (公開鍵) のみRasperry PIにコピーする (ID_RSAは任意)



# 初期設定
## 起動
- Ethernetケーブルとパソコンを接続する
- 電源を入れる
- 詳細は省略する

## Raspberry PIに割り振られたIPアドレスを調べる

```sudo arp-scan -I enp0s25 -l```

詳細は下のページから
[共有されたIPアドレスをしらべる（Linux）](https://qiita.com/sik103/items/5f7f6a84e5cb8fff8668)

## raspi-config
```shell-session:raspi-config
$ sudo raspi-config
```
- location, keyboard, language, wifi country 設定
- パスワード変更 などなど

## Wifi, source.listの設定
```shell-session
$ cd ~/DIRECTORY
$ sudo cp /etc/wpa_supplicant/wpa_supplicant.conf /etc/wpa_supplicant/wpa_supplicant.conf.org  # for Backup
$ cat wpa_supplicant_conf | sudo tee -a /etc/wpa_supplicant/wpa_supplicant.conf
$ sudo cp /etc/apt/sources.list /etc/apt/sources.list.org
$ cat sources_list | sudo tee -a /etc/apt/sources.list
```

# ファイアウォールのインストールなど
```shell-session
$ sudo apt install ufw
$ sudo ufw enable
$ sudo ufw default deny
```

sshで接続している場合は、sshが中断されないように注意！

# アップデート
```shell-session
$ sudo apt upgrade
$ sudo apt -y upgrade
$ sudo apt -y autoremove
$ sudo apt clean
$ sudo apt autoclean

$ sudo apt dist-upgrade  # apt upgrade やれば多分必要ない
```

# セキュリティ設定
## ユーザ名 (pi)の変更
### 仮のユーザを追加する
piユーザとしてログインしつつpiユーザのアカウント名を変更する事は不可能であるため、一時的に利用する仮のユーザを追加する。

```shell-session
$ sudo useradd -M tmp # 仮のユーザ(tmp)を作成する
$ sudo gpasswd -a tmp sudo  # tmpユーザをsudoグループに追加(そうしないとsudoが使えない)
$ sudo passwd tmp  # tmpユーザのパスワードを設定
$ exit # ログアウトする
```

### ユーザ名を変更する
先程作成した仮ユーザ(tmp)にログインする。
その後、piユーザのユーザ名、グループ名を変更する。

```shell-session
$ sudo usermod -l newpi pi  # ユーザ名をpiからnewpiに変更, newpiは任意
$ sudo usermod -d /home/newpi -m newpi  ホームディレクトリを/home/piから/home/newpiに変更
$ sudo groupmod -n newpi pi  # piグループをnewpiグループに変更
$ exit  # ログアウトする
```

### 仮ユーザの削除
newpiにログインする。

```shell-session
$ sudo userdel tmp　　# 仮ユーザを削除
```

## sudoコマンドの使用時にパスワードの入力を必須にする

設定ファイル```/etc/sudoers.d/010_pi-nopasswd```の```pi ALL=(ALL) NOPASSWD: ALL```をコメントアウトする

## root loginのロック
```shell-session
$ sudo passwd -l root
```

# ssh設定（内部ネットワークへの解放）
## static ip
- ルーターで適当に設定する
- Mac addressが必要な場合は、```ifconfig```を使用

## ssh
### ID_RSA.pubの移動

```shell-session
$ mkdir ~/.ssh
$ cp ~/DIRECTORY/ID_RSA.pub ~/.ssh/authorized_keys
```

### sshd_configの編集
```sudo nano /etc/ssh/sshd_config```を編集する
- Port 0000 (0000は任意)
- PermitRootLogin no
- PermitEmptyPasswords no
- PasswordAuthentication no

## ufwの変更
```shell-session
$ sudo ufw allow from 192.168.0.0/16 to any port 0000
```

# ssh設定2（外部ネットワークへの解放）
## グローバルip確認
右のサイトで確認できる：[アクセス情報【使用中のIPアドレス確認】](https://www.cman.jp/network/support/go_access.cgi)

## ufwの設定
```shell-session
$ sudo ufw status
$ sudo ufw delete 0 （0は任意）
$ sudo ufw allow to any port 0000
```
# dmzの設定
ルーターを適当にいじる

# 参考文献
[[Raspbian]ユーザ名変更の個人的に「正しい」と思うやり方](https://jyn.jp/raspberrypi-username-change/)
[root ユーザーのログインをロックする](http://takuya-1st.hatenablog.jp/entry/20120828/1346186699)
[無線LANのパスフレーズを暗号化するには？](http://www.atmarkit.co.jp/ait/articles/1601/23/news008.html)
