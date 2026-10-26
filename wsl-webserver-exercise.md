# Windows 11 + WSL で学ぶ Webサーバ構築演習

## この演習のゴール

- WSL上のUbuntuにWebサーバ（Apache）を構築し、Webページを公開する
- SSHで「パスワード認証 → 鍵認証 → パスワード認証禁止」の流れを体験する
- SFTPとFTPでファイルを転送し、両者の違いを理解する
- ufwでファイアウォールを設定し、不要なポートを閉じる
- MySQLにデータベースとテーブルを作成する
- Pythonのプログラムからデータを登録・表示・更新・削除（CRUD）する

## 前提条件

- Windows 11 に WSL2 がインストール済み
- Ubuntu 24.04 を想定（他のバージョンでは一部手順が異なる場合があります）
- 管理者権限で PowerShell を開けること

### 表記について

| 表記 | 意味 |
|---|---|
| **【Ubuntu】** | WSL上のUbuntuのターミナルで実行 |
| **【Windows】** | Windows側のPowerShellで実行 |
| **【MySQL（root）】** / **【MySQL（libuser）】** | `mysql>` プロンプトで実行（カッコ内はログインしているユーザー） |
| `ユーザー名` | 自分のUbuntuユーザー名に置き換える |

### 設定ファイルの編集方法（nano）

この演習では、設定ファイルを **エディタで開いて書き換え、保存する** 方法で編集します。サーバーの設定は「ファイルを開く → 書き換える → 保存する → サービスに反映させる」の繰り返しです。ファイルの中身を自分の目で見ながら作業することで、設定の構造に慣れていきましょう。

Ubuntuに最初から入っている `nano` を使います。`/etc` 以下のファイルは管理者しか書き込めないため、`sudo` を付けて開きます。

```bash
sudo nano ファイル名
```

| 操作 | キー |
|---|---|
| カーソル移動 | 矢印キー |
| ファイルの末尾へ移動 | `Alt` + `/` |
| 文字列を検索 | `Ctrl` + `W` |
| 保存 | `Ctrl` + `O` → `Enter` |
| 終了 | `Ctrl` + `X` |
| 保存せずに終了 | `Ctrl` + `X` → `N` |

> 💡 画面下部の `^O` や `^X` の `^` は `Ctrl` キーを表します。困ったら画面下の表示を確認しましょう。

> 💡 Windowsでコピーした文字は、ターミナル上で **右クリック** または `Ctrl` + `Shift` + `V` で貼り付けられます。

> ⚠️ **編集前にバックアップを取る習慣をつけましょう。**
>
> ```bash
> sudo cp 設定ファイル 設定ファイル.bak
> ```
>
> 書き換えに失敗しても、バックアップから元に戻せます。

---

## 1. Ubuntuの準備

**【Ubuntu】** パッケージを最新にします。

```bash
sudo apt update && sudo apt upgrade -y
```

systemdが有効か確認します。

```bash
cat /etc/wsl.conf
```

次の記述があればOKです。

```ini
[boot]
systemd=true
```

記述が無い場合は、ファイルを開いて追記します。

```bash
sudo nano /etc/wsl.conf
```

ファイルの末尾に次の2行を書き足し、`Ctrl` + `O` → `Enter` で保存、`Ctrl` + `X` で終了します。

```ini
[boot]
systemd=true
```

保存できたか確認します。

```bash
cat /etc/wsl.conf
```

**【Windows】** で WSL を再起動します。

```powershell
wsl --shutdown
```

その後、再度Ubuntuを起動します。

### ✅ 確認ポイント

- [ ] `systemctl` コマンドがエラーなく動く（例：`systemctl --version`）

---

## 2. Webサーバ（Apache）の構築

**【Ubuntu】**

```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
sudo systemctl status apache2 --no-pager
```

Windowsのブラウザで次のURLを開きます。

```
http://localhost
```

### ✅ 確認ポイント

- [ ] `status` で `active (running)` と表示される
- [ ] ブラウザに Apache のデフォルトページ（Apache2 Ubuntu Default Page）が表示される

> 💡 WSL2では、Windows側の `localhost` へのアクセスがUbuntu側へ自動で転送されます（localhost転送）。

### 2-1. IPアドレスでアクセスする

`localhost` は便利ですが、実際のサーバーには **IPアドレス** でアクセスするのが基本です。WSLのUbuntuもIPアドレスを持っているので、IPでのアクセスも体験しましょう。

#### WSLのネットワークのしくみ

WSL2のUbuntuは、Windowsの中に作られた **仮想ネットワーク** につながった、1台の別のコンピュータのように動いています。

```
┌──────────── Windows 11 ────────────┐
│                                     │
│   ブラウザ / PowerShell             │
│        │                            │
│        │ ① localhost（自動転送）    │
│        │ ② 172.25.x.x（直接アクセス）│
│        ▼                            │
│  ┌──── 仮想ネットワーク ────┐       │
│  │  Ubuntu（WSL）           │       │
│  │  IP: 172.25.x.x          │       │
│  │  Apache / SSH / FTP      │       │
│  └──────────────────────────┘       │
└─────────────────────────────────────┘
```

| アクセス先 | 意味 |
|---|---|
| `localhost`（= `127.0.0.1`） | 「自分自身」を表す特別な名前。WSL2ではUbuntuへ自動転送される |
| `172.25.x.x` など | 仮想ネットワーク上のUbuntuのIPアドレス |

#### UbuntuのIPアドレスを調べる

**【Ubuntu】**

```bash
hostname -I
```

より詳しく確認する場合は次のコマンドを使います。

```bash
ip -4 addr show eth0
```

`inet 172.25.x.x/20` のように表示された `/` より前の部分がIPアドレスです。

> ⚠️ **よくある間違い**
> Windowsの `ipconfig` に表示される「イーサネット アダプター vEthernet (WSL)」のアドレスは、**Windows側** の仮想アダプタのIPであり、UbuntuのIPではありません。UbuntuのIPは、必ずUbuntu側で調べましょう。

#### IPアドレスでWebページを開く

**【Windows】** 通信できるか確認します。

```powershell
ping 172.25.x.x
```

ブラウザで次のURLを開きます。

```
http://172.25.x.x
```

PowerShellから確認することもできます（`curl` ではなく `curl.exe` と入力します）。

```powershell
curl.exe -I http://172.25.x.x
```

`HTTP/1.1 200 OK` と表示されれば成功です。

> ⚠️ **IPアドレスは変わります**
> WSLのIPアドレスは、`wsl --shutdown` やPCの再起動のたびに変わることがあります。つながらなくなったら、まず `hostname -I` でIPを確認し直しましょう。

> 💡 この仮想ネットワークのIPは、Windows自身からしかアクセスできません。LAN内の別のPCからアクセスする方法は「付録B」を参照してください。

### ✅ 確認ポイント

- [ ] Ubuntu側でIPアドレスを調べられた
- [ ] `localhost` と IPアドレスの両方で同じページが表示された
- [ ] `ipconfig` の vEthernet (WSL) のアドレスがUbuntuのIPとは違うことを確認した

---

## 3. SSHサーバの構築と認証方式の変更

SSHは次の3段階で進めます。

1. パスワード認証でログインする
2. 鍵認証を設定する
3. パスワード認証を禁止する

### 3-0. Ubuntuユーザーのパスワード確認

パスワードは、SSHのパスワードログインと `sudo` の両方で使います。最初に確認しておきましょう。

**【Ubuntu】**

```bash
sudo -k && sudo true
```

パスワードを聞かれ、エラーなく終われば問題ありません。

#### パスワードを忘れた場合のリセット手順

パスワードを忘れると `sudo` も使えないため、Windows側から復旧します。

**【Windows】** ディストリビューション名を確認します。

```powershell
wsl -l -v
```

確認した名前を指定して、rootでUbuntuに入ります。

```powershell
wsl -d Ubuntu-24.04 -u root
```

ユーザー名が分からない場合は確認します。

```bash
getent passwd 1000 | cut -d: -f1
```

パスワードを再設定します。

```bash
passwd ユーザー名
exit
```

通常どおり `wsl` で起動し直し、`sudo -k && sudo true` で新しいパスワードが通ることを確認します。

> ⚠️ **考えてみよう**
> WSLでは、Windowsにログインできる人は誰でも `-u root` でroot権限を得られます。つまり「Windowsアカウントの保護 ＝ WSLの保護」です。
> 実機のサーバーではこのような復旧はできず、リカバリーモードやコンソールからの作業が必要になります。

### 3-1. SSHサーバの導入とパスワード認証でのログイン

**【Ubuntu】**

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo sshd -T | grep passwordauth
```

`passwordauthentication yes` と表示されることを確認します。

**【Windows】** パスワードでログインします。

```powershell
ssh ユーザー名@localhost
```

初回は次のような確認が表示されるので、`yes` と入力します。

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

> 💡 これは接続先サーバーの「ホスト鍵」の確認です。一度承認すると `C:\Users\<Windowsユーザー>\.ssh\known_hosts` に記録され、次回からは聞かれません。接続先がすり替えられた場合は警告が出る仕組みです。

ログインできたら `exit` で抜けます。

次に、**IPアドレスでもログイン** してみましょう（IPは 2-1 で調べたもの）。

```powershell
ssh ユーザー名@172.25.x.x
```

同じサーバーなのに、もう一度フィンガープリントの確認が表示されます。

> ⚠️ **考えてみよう**
> 同じUbuntuに接続しているのに、なぜ再び確認されたのでしょうか？
> → `known_hosts` は **接続先の名前（`localhost` や IPアドレス）ごと** にホスト鍵を記録しているためです。中身を見ると、2つの行が登録されています。
>
> ```powershell
> type $env:USERPROFILE\.ssh\known_hosts
> ```
>
> WSLのIPが変わった後も、新しいIPに対して同じ確認が表示されます。

> 💡 以降の手順では `localhost` で説明しますが、IPアドレスに置き換えても同じように動作します。

> 💡 Ubuntu 24.04 では、SSHは `ssh.socket` によって起動されます。ポート番号を変更する場合は `sshd_config` ではなく、`sudo systemctl edit ssh.socket` で `ListenStream` を変更します。

#### ✅ 確認ポイント

- [ ] パスワードを入力してSSHログインできた
- [ ] `localhost` と IPアドレスの両方でログインできた
- [ ] `known_hosts` に2つの接続先が記録されている

### 3-2. 鍵認証の設定

**【Windows】** 鍵ペアを作成します。

```powershell
ssh-keygen -t ed25519
```

保存場所はそのまま Enter、パスフレーズは任意で設定します。次の2つのファイルが作成されます。

| ファイル | 役割 | 扱い |
|---|---|---|
| `id_ed25519` | 秘密鍵 | **絶対に他人に渡さない** |
| `id_ed25519.pub` | 公開鍵 | サーバーに登録する |

公開鍵をUbuntuに登録します。Windowsには `ssh-copy-id` が無いため、次のコマンドで転送します。

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh ユーザー名@localhost "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

> 💡 この時点ではまだ鍵が登録されていないので、転送はパスワード認証で行われます。

鍵認証だけでログインできるか確認します。

```powershell
ssh -o PreferredAuthentications=publickey ユーザー名@localhost
```

詳細ログで認証方式を確認することもできます。

```powershell
ssh -v ユーザー名@localhost
```

ログの中に `Authenticated ... using "publickey"` と表示されていれば、鍵認証でログインしています。

#### ✅ 確認ポイント

- [ ] パスワードを聞かれずに（パスフレーズを設定した場合はパスフレーズのみで）ログインできた
- [ ] `-v` のログで `publickey` による認証を確認できた

### 3-3. パスワード認証の禁止

> ⚠️ **重要：現在のSSHセッションは閉じずに残したまま作業してください。**
> 設定を誤って締め出されても、残したセッションから修正できます。本番サーバーでも必須の習慣です。

**【Ubuntu】** 別のウィンドウで作業します。まず、SSHの設定ファイルの構成を確認しましょう。

```bash
ls /etc/ssh/sshd_config.d/
grep -n Include /etc/ssh/sshd_config
```

メインの設定ファイル `/etc/ssh/sshd_config` の先頭付近に `Include /etc/ssh/sshd_config.d/*.conf` という行があり、このフォルダ内のファイルも設定として読み込まれることが分かります。

> 💡 `sshd_config` 本体を直接書き換えることもできますが、追加の設定を別ファイルに分けておくと、何を変更したかが一目で分かり、元に戻すのも簡単です。

新しい設定ファイルを作成して開きます（存在しないファイル名を指定すると、新規作成になります）。

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

次の2行を入力します。

```
PasswordAuthentication no
PermitRootLogin no
```

| 設定 | 意味 |
|---|---|
| `PasswordAuthentication no` | パスワードによるログインを禁止する |
| `PermitRootLogin no` | rootユーザーでの直接ログインを禁止する |

`Ctrl` + `O` → `Enter` で保存し、`Ctrl` + `X` で終了します。保存した内容を確認します。

```bash
cat /etc/ssh/sshd_config.d/99-hardening.conf
```

設定の文法チェックをしてから再起動します。

```bash
sudo sshd -t && sudo systemctl restart ssh
```

実際に有効になっている値を確認します。

```bash
sudo sshd -T | grep -E 'passwordauth|permitroot'
```

> ⚠️ `passwordauthentication no` になっていない場合、`50-cloud-init.conf` など別の設定ファイルが先に読み込まれて優先されています。`sshd_config.d` 内のファイルは名前の辞書順に読まれ、**最初に書かれた値が有効**になります。

**【Windows】** パスワード認証が拒否されることを確認します。

```powershell
ssh -o PubkeyAuthentication=no ユーザー名@localhost
```

次のように表示されれば成功です。

```
ユーザー名@localhost: Permission denied (publickey).
```

最後に、通常のコマンドで鍵ログインできることも確認します。

```powershell
ssh ユーザー名@localhost
```

#### ✅ 確認ポイント

- [ ] 3-1 と 3-3 の `sshd -T` の結果を比べ、`passwordauthentication` が `yes` → `no` に変わった
- [ ] パスワード認証が `Permission denied (publickey)` で拒否される
- [ ] 鍵認証では引き続きログインできる

> ⚠️ **よくある誤解**
> SSHのパスワード認証を禁止しても、`sudo` にはUbuntuユーザーのパスワードが引き続き必要です。「SSHでパスワードを使わない」ことと「パスワードが不要になる」ことは別物です。

> 💡 SSHの秘密鍵を失くした場合も、3-0 と同じく `wsl -d Ubuntu-24.04 -u root` で入り、`~/.ssh/authorized_keys` を修正すれば復旧できます。

---

## 4. SFTPでのファイル転送

SFTPはSSHの仕組みを使ったファイル転送です。追加のインストールは不要で、3で設定した鍵認証がそのまま使えます。

### 4-1. 公開ディレクトリの権限設定

**【Ubuntu】** 自分のユーザーが `/var/www/html` に書き込めるようにします。

```bash
sudo usermod -aG www-data $USER
sudo chown -R $USER:www-data /var/www/html
sudo chmod -R 2775 /var/www/html
```

> 💡 `usermod` によるグループ追加は、再ログイン後に反映されます。`2775` の先頭の `2`（SGID）は、ディレクトリ内に作ったファイルのグループを自動的に `www-data` にする設定です。

### 4-2. SFTPで接続する

**【Windows】**

```powershell
sftp ユーザー名@localhost
```

主なコマンドは次のとおりです。

| コマンド | 意味 |
|---|---|
| `pwd` / `lpwd` | サーバー側 / 手元側の現在地 |
| `ls` / `lls` | サーバー側 / 手元側の一覧 |
| `cd` / `lcd` | サーバー側 / 手元側の移動 |
| `put ファイル` | アップロード |
| `get ファイル` | ダウンロード |
| `exit` | 終了 |

> 💡 WinSCP や FileZilla などのGUIクライアントでも、プロトコル「SFTP」・ホスト「localhost」・ポート「22」・秘密鍵 `id_ed25519` を指定すれば接続できます（WinSCPは `.ppk` 形式への変換を案内してくれます）。

### ✅ 確認ポイント

- [ ] パスワードを使わずにSFTP接続できた
- [ ] `cd /var/www/html` で公開ディレクトリに移動できた

---

## 5. FTPサーバ（vsftpd）の構築

### 5-1. インストールと設定

**【Ubuntu】**

```bash
sudo apt install -y vsftpd
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
```

設定ファイルを開きます。

```bash
sudo nano /etc/vsftpd.conf
```

まずは中身を眺めてみましょう。`#` で始まる行は **コメント**（説明文や無効化された設定）で、vsftpdには読み込まれません。たとえば `Ctrl` + `W` で `write_enable` を検索すると、次のような行が見つかります。

```
#write_enable=YES
```

先頭に `#` が付いているため、この設定は現在無効になっています。

`Alt` + `/` でファイルの末尾に移動し、次の内容を書き足します。

```
# --- 演習用の追加設定 ---
write_enable=YES
local_umask=022
chroot_local_user=YES
allow_writeable_chroot=YES
local_root=/var/www/html
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
```

`Ctrl` + `O` → `Enter` で保存し、`Ctrl` + `X` で終了します。

> 💡 末尾に追記する代わりに、既存の `#write_enable=YES` などの行頭の `#` を消して有効化する方法もあります。どちらでも動作しますが、この演習では「自分が追加した設定」がまとまって分かるよう、末尾に追記しています。

コメントと空行を除いて、有効な設定だけを表示して確認します。

```bash
grep -v -e '^#' -e '^$' /etc/vsftpd.conf
```

追記した設定が表示されていればOKです。バックアップとの差分を見ることもできます。

```bash
diff /etc/vsftpd.conf.bak /etc/vsftpd.conf
```

| 設定 | 意味 |
|---|---|
| `write_enable=YES` | アップロード・削除などの書き込みを許可 |
| `local_umask=022` | 作成ファイルの権限を `644`（ディレクトリは `755`）にする |
| `chroot_local_user=YES` | ログインユーザーを指定ディレクトリより上に移動させない |
| `allow_writeable_chroot=YES` | 書き込み可能なディレクトリでのchrootを許可（演習用） |
| `local_root` | ログイン直後のディレクトリ |
| `pasv_*` | パッシブモードと、データ転送に使うポート範囲 |

再起動して状態を確認します。

```bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd --no-pager
```

> ⚠️ `allow_writeable_chroot=YES` は演習のための簡略化です。本番環境ではchroot先のディレクトリを書き込み不可にし、その配下に書き込み用ディレクトリを用意するのが定石です。

### 5-2. FTPで接続する

> ⚠️ Windows標準の `ftp` コマンドはパッシブモードに対応していないため、接続できないことがほとんどです。**FileZilla** または **WinSCP** を使用してください。

| 項目 | 値 |
|---|---|
| プロトコル | FTP |
| ホスト | localhost |
| ポート | 21 |
| ユーザー名 / パスワード | Ubuntuのユーザー名とパスワード |

Ubuntu内で確認する場合は `lftp` が便利です。

```bash
sudo apt install -y lftp
lftp ユーザー名@localhost
```

### ✅ 確認ポイント

- [ ] FTPクライアントで接続でき、最初から `/var/www/html` の中にいる
- [ ] `/var/www` など上の階層へは移動できない

> ⚠️ **考えてみよう**
> 3-3でSSHのパスワード認証を禁止したのに、FTPはパスワードでログインできました。なぜでしょうか？
> → 認証の設定はサービスごとに独立しているためです。SSHの設定はFTPには影響しません。

> ⚠️ **FTPの危険性**
> FTPはパスワードも転送内容も**暗号化されずに**送られます。次のコマンドを実行した状態でFTPログインすると、パスワードがそのまま見えます。
>
> ```bash
> sudo tcpdump -i lo -A port 21
> ```

---

## 6. Webページの公開

### 6-1. HTMLファイルを作成する

Windows側で、任意のフォルダに `index.html` を作成します（文字コードは UTF-8 で保存）。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>演習サイト</title>
</head>
<body>
  <h1>WSLのWebサーバで公開中</h1>
</body>
</html>
```

### 6-2. アップロードして表示する

1. **SFTP**（手順4）で `index.html` を `/var/www/html` にアップロードし、`http://localhost` を再読込する
2. 見出しの文言を少し書き換え、今度は **FTP**（手順5）でアップロードして再読込する

**【Ubuntu】** 権限を確認します。

```bash
ls -l /var/www/html
```

ファイルが `644`（`-rw-r--r--`）、ディレクトリが `755` 以上であれば、Apacheから読み込めます。

### 6-3. 発展：サブディレクトリのページ

`/var/www/html/about/index.html` を作成し、次のURLでアクセスしてみましょう。

```
http://localhost/about/
```

URLとサーバー上のディレクトリがどう対応しているかを確認します。

### ✅ 確認ポイント

- [ ] SFTPとFTPのどちらでアップロードしても、ブラウザに反映された
- [ ] サブディレクトリのページを表示できた

> 💡 **403 Forbidden** が出た場合は、まずファイルとディレクトリの権限を確認してください。

---

## 7. ファイアウォール（ufw）の設定

### 7-1. ルールを設定して有効化する

> ⚠️ **有効化する前に、必ずSSHを許可してください。**
> 順番を逆にすると、実機のサーバーでは自分自身がSSHで入れなくなります。

**【Ubuntu】**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 21/tcp
sudo ufw allow 40000:40100/tcp
sudo ufw enable
sudo ufw status numbered
```

| ポート | 用途 |
|---|---|
| 22 | SSH / SFTP |
| 80 | HTTP（Webページ） |
| 21 | FTP（制御用） |
| 40000〜40100 | FTP（パッシブモードのデータ転送用） |

> 💡 SSHのポートを変更している場合は、`22` をそのポート番号に読み替えてください。

### 7-2. 動作を確認する

> ⚠️ Windowsから `localhost` でアクセスすると、WSLの転送の仕組みによってufwを通らずに接続できてしまう場合があります。ufwの効果を確かめるときは、**WSLのIPアドレス**でアクセスしてください。

**【Ubuntu】** IPアドレスを確認します。

```bash
hostname -I
```

Windowsのブラウザで、表示されたIPアドレス（例：`http://172.25.x.x`）を開き、ページが表示されることを確認します。

次に、HTTPを一時的にブロックします。

```bash
sudo ufw delete allow 80/tcp
```

ブラウザを再読込すると、つながらなくなります。許可を戻します。

```bash
sudo ufw allow 80/tcp
```

再読込で再び表示されることを確認します。

### 7-3. 不要なサービスを閉じる

SFTPがあればファイル転送はできるため、平文のFTPは不要です。サービスとポートを閉じましょう。

```bash
sudo systemctl disable --now vsftpd
sudo ufw delete allow 21/tcp
sudo ufw delete allow 40000:40100/tcp
sudo ufw status numbered
```

### ✅ 確認ポイント

- [ ] `ufw status` に許可したポートが表示される
- [ ] ポート80を閉じるとWebページが表示されなくなり、開くと戻る
- [ ] 最終的に、22番と80番だけが許可されている

> 💡 **原則：使わないサービスは止め、使わないポートは閉じる。**
> 攻撃される入口を減らすことが、サーバーセキュリティの基本です。

---

## 8. データベース（MySQL）の構築

### 8-1. MySQLのインストール

**【Ubuntu】**

```bash
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo systemctl status mysql --no-pager
```

`active (running)` と表示されればOKです。バージョンも確認しておきます。

```bash
mysql --version
```

### 8-2. 初期セキュリティ設定

MySQLには、初期状態の不要な設定を片付けるためのコマンドが用意されています。

```bash
sudo mysql_secure_installation
```

質問には次のように答えます。

| 質問 | 回答 | 意味 |
|---|---|---|
| VALIDATE PASSWORD component | `n` | パスワード強度チェック（演習では使わない） |
| Remove anonymous users? | `y` | 匿名ユーザーを削除 |
| Disallow root login remotely? | `y` | rootの外部からのログインを禁止 |
| Remove test database? | `y` | テスト用データベースを削除 |
| Reload privilege tables now? | `y` | 設定をすぐ反映 |

### 8-3. MySQLにrootでログインする

```bash
sudo mysql
```

`mysql>` というプロンプトが表示されれば、MySQLの中に入っています。

> 💡 **パスワードを聞かれないのはなぜ？**
> UbuntuのMySQLでは、MySQLの `root` ユーザーが **auth_socket** という方式で認証されます。「Ubuntuの root（`sudo`）で実行している人なら、MySQLの root として認める」という仕組みです。次のSQLで確認できます。
>
> ```sql
> SELECT user, host, plugin FROM mysql.user;
> ```
>
> `root` の `plugin` が `auth_socket` になっています。

> 💡 MySQLの中で実行するSQL文は、最後に **`;`（セミコロン）** を付けます。付け忘れると `->` が表示されて入力待ちになるので、`;` を入力して Enter を押しましょう。

### 8-4. データベースと専用ユーザーの作成

アプリから root で接続するのは危険なので、`library` データベース専用のユーザーを作ります。

**【MySQL（root）】**

```sql
CREATE DATABASE library CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
CREATE USER 'libuser'@'localhost' IDENTIFIED BY 'LibPass#2026';
GRANT ALL PRIVILEGES ON library.* TO 'libuser'@'localhost';
SHOW DATABASES;
EXIT;
```

| SQL | 意味 |
|---|---|
| `CREATE DATABASE library ...` | `library` データベースを作成。`utf8mb4` は日本語や絵文字を扱える文字コード |
| `CREATE USER 'libuser'@'localhost' ...` | このサーバー内（localhost）からだけ接続できるユーザーを作成 |
| `GRANT ALL PRIVILEGES ON library.* ...` | `library` データベースに対してだけ、すべての権限を与える |

> ⚠️ `LibPass#2026` は演習用の例です。実際には自分で決めたパスワードにし、忘れないよう控えておきましょう。

> 💡 **権限は必要な範囲だけ**
> `libuser` は `library` 以外のデータベースを操作できません。万一パスワードが漏れても、被害を `library` だけに抑えられます。これを **最小権限の原則** といいます。

### 8-5. 文字コードの設定

`mysql` コマンドから日本語を正しく入力・表示できるよう、クライアントの文字コードを設定します。環境によってはこの設定をしないと、日本語が `å¾è¼©...` のように文字化けします。

**【Ubuntu】**

```bash
sudo cp /etc/mysql/conf.d/mysql.cnf /etc/mysql/conf.d/mysql.cnf.bak
sudo nano /etc/mysql/conf.d/mysql.cnf
```

`[mysql]` の行のすぐ下に、次の1行を書き足して保存します。

```ini
[mysql]
default-character-set=utf8mb4
```

> 💡 `[mysql]` のような `[ ]` で囲まれた行は **セクション** と呼ばれ、「ここから下は `mysql` コマンド用の設定」という意味です。

### 8-6. booksテーブルの作成

作成した `libuser` でログインします。

**【Ubuntu】**

```bash
mysql -u libuser -p library
```

パスワードを入力すると、`library` データベースを使う状態でログインします。

文字コードの設定が効いているか確認します。

**【MySQL（libuser）】**

```sql
SHOW VARIABLES LIKE 'character_set_client';
```

`utf8mb4` と表示されればOKです。

`books` テーブルを作成します。

```sql
CREATE TABLE books (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200) NOT NULL,
  author VARCHAR(100) NOT NULL,
  published_year INT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

| 列 | 型・制約 | 意味 |
|---|---|---|
| `id` | `INT AUTO_INCREMENT PRIMARY KEY` | 本の番号。自動で1, 2, 3…と振られ、重複しない |
| `title` | `VARCHAR(200) NOT NULL` | タイトル（最大200文字、必須） |
| `author` | `VARCHAR(100) NOT NULL` | 著者（最大100文字、必須） |
| `published_year` | `INT` | 出版年（空欄可） |
| `created_at` | `DATETIME DEFAULT CURRENT_TIMESTAMP` | 登録日時。省略すると現在日時が自動で入る |

テーブルを確認します。

```sql
SHOW TABLES;
DESCRIBE books;
```

動作確認として、SQLで2件登録して表示します。

```sql
INSERT INTO books (title, author, published_year)
VALUES ('吾輩は猫である', '夏目漱石', 1905),
       ('銀河鉄道の夜', '宮沢賢治', 1934);

SELECT * FROM books;
```

```
+----+-----------------------+--------------+----------------+---------------------+
| id | title                 | author       | published_year | created_at          |
+----+-----------------------+--------------+----------------+---------------------+
|  1 | 吾輩は猫である        | 夏目漱石     |           1905 | 2026-10-27 08:00:00 |
|  2 | 銀河鉄道の夜          | 宮沢賢治     |           1934 | 2026-10-27 08:00:00 |
+----+-----------------------+--------------+----------------+---------------------+
```

`id` と `created_at` は指定していませんが、自動で値が入っています。確認できたら `EXIT;` で抜けます。

### 8-7. MySQLのポートは開けなくてよい

**【Ubuntu】** MySQLがどのアドレスで待ち受けているか確認します。

```bash
sudo ss -tlnp | grep 3306
```

```
LISTEN 0 151 127.0.0.1:3306 0.0.0.0:* users:(("mysqld",...))
```

`127.0.0.1:3306` は「このサーバーの中からの接続だけを受け付ける」という意味です。PythonのプログラムもApacheも同じサーバー内で動くので、外部に3306番を公開する必要はありません。**ufwで3306番を許可しない** のが正解です。

> 💡 この設定は `/etc/mysql/mysql.conf.d/mysqld.cnf` の `bind-address = 127.0.0.1` で決まっています。`cat` で確認してみましょう。

### ✅ 確認ポイント

- [ ] `sudo mysql` でrootとしてログインできた
- [ ] `libuser` でログインでき、`library` データベースに `books` テーブルがある
- [ ] 日本語のデータが文字化けせずに表示された
- [ ] MySQLが `127.0.0.1` だけで待ち受けていることを確認した

---

## 9. PythonでデータベースをCLI操作する

Pythonのプログラムから `books` テーブルに対して、データの **登録（INSERT）・表示（SELECT）・更新（UPDATE）・削除（DELETE）** を行います。この4つの操作は、頭文字をとって **CRUD**（Create / Read / Update / Delete）と呼ばれ、ほとんどのWebアプリの基本になっています。

| 操作 | SQL | 作成するプログラム |
|---|---|---|
| 登録（Create） | `INSERT` | `insert_book.py` |
| 表示（Read） | `SELECT` | `select_books.py` |
| 更新（Update） | `UPDATE` | `update_book.py` |
| 削除（Delete） | `DELETE` | `delete_book.py` |

### 9-1. Python環境の準備

**【Ubuntu】** Ubuntu 24.04 にはPythonが最初から入っています。

```bash
python3 --version
```

作業用のフォルダを作り、**仮想環境（venv）** を用意します。

```bash
sudo apt install -y python3-venv
mkdir ~/library-app
cd ~/library-app
python3 -m venv .venv
source .venv/bin/activate
```

プロンプトの先頭に `(.venv)` と表示されれば、仮想環境に入っています。

> 💡 **仮想環境とは？**
> プロジェクトごとに専用のPython環境を作る仕組みです。ライブラリをプロジェクトの中だけにインストールするので、OSのPythonや他のプロジェクトに影響しません。
> Ubuntu 24.04 では、仮想環境を使わずに `pip install` すると `error: externally-managed-environment` というエラーで止められます。

MySQLに接続するためのライブラリ **PyMySQL** をインストールします。

```bash
pip install "PyMySQL[rsa]"
pip list
```

> 💡 `[rsa]` を付けると、MySQL 8.0 の標準の認証方式（caching_sha2_password）に必要な暗号化ライブラリも一緒にインストールされます。

> ⚠️ ターミナルを開き直したときは、毎回 `cd ~/library-app` と `source .venv/bin/activate` を実行してから作業します。仮想環境を抜けるときは `deactivate` です。

### 9-2. 接続設定ファイルの作成（db.py）

どのプログラムからも同じ方法でデータベースに接続できるよう、接続処理を1つのファイルにまとめます。

```bash
nano db.py
```

次の内容を入力して保存します。

```python
import pymysql

# データベースへの接続設定
DB_CONFIG = {
    "host": "localhost",
    "user": "libuser",
    "password": "LibPass#2026",
    "database": "library",
    "charset": "utf8mb4",
    "cursorclass": pymysql.cursors.DictCursor,
}


def get_connection():
    """MySQLに接続し、接続オブジェクトを返す"""
    return pymysql.connect(**DB_CONFIG)
```

| 設定 | 意味 |
|---|---|
| `host` / `user` / `password` / `database` | 8-4 で作成した接続先・ユーザー・データベース |
| `charset` | 日本語を正しく扱うための文字コード |
| `cursorclass` | 結果を `book['title']` のように列名で取り出せるようにする |

> ⚠️ `password` は 8-4 で設定したものに書き換えてください。
> このファイルにはパスワードが書かれているので、他の人に読まれないよう権限を絞っておきます。
>
> ```bash
> chmod 600 db.py
> ```
>
> 本番の開発では、パスワードをプログラムに直接書かず、環境変数や設定ファイルに分けるのが一般的です。

### 9-3. 登録（INSERT）

```bash
nano insert_book.py
```

```python
from db import get_connection

print("=== 本の登録 ===")
title = input("タイトル: ")
author = input("著者: ")
year = input("出版年（例: 1905）: ")

sql = "INSERT INTO books (title, author, published_year) VALUES (%s, %s, %s)"

with get_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute(sql, (title, author, int(year)))
        new_id = cursor.lastrowid
    conn.commit()

print(f"登録しました（ID: {new_id}）")
```

実行します。

```bash
python insert_book.py
```

```
=== 本の登録 ===
タイトル: 坊っちゃん
著者: 夏目漱石
出版年（例: 1905）: 1906
登録しました（ID: 3）
```

#### ポイント①：`%s`（プレースホルダ）を使う

SQLの中の値を入れる場所は `%s` にしておき、実際の値は `execute()` の2つ目の引数で渡します。

```python
# ✅ 正しい書き方
cursor.execute("... VALUES (%s, %s, %s)", (title, author, int(year)))

# ❌ やってはいけない書き方
cursor.execute(f"... VALUES ('{title}', '{author}', {year})")
```

入力された文字をそのままSQLに埋め込むと、悪意のある入力によってSQL文を書き換えられてしまいます。これを **SQLインジェクション** といい、Webアプリで最も有名な脆弱性の1つです。`%s` を使うと、PyMySQLが値を安全な形に変換してくれます。

#### ポイント②：`commit()` で確定する

`INSERT`・`UPDATE`・`DELETE` でデータを変更したら、最後に `conn.commit()` を実行して変更を **確定** します。`commit()` を忘れると、プログラムが終わったときに変更が取り消されます。

> 🧪 **実験してみよう**
> `conn.commit()` の行の先頭に `#` を付けてコメントにし、もう一度登録してみましょう。「登録しました」と表示されるのに、次の 9-4 で一覧を見ると増えていません。確認できたら `#` を外して元に戻します。
> なお、取り消された登録でもIDの番号は使われたことになるため、次に登録した本のIDは1つ飛んだ番号になります。

### 9-4. 表示（SELECT）

```bash
nano select_books.py
```

```python
from db import get_connection

print("=== 本の一覧 ===")
keyword = input("タイトルの検索語（空欄で全件表示）: ")

sql = "SELECT id, title, author, published_year FROM books WHERE title LIKE %s ORDER BY id"

with get_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute(sql, (f"%{keyword}%",))
        books = cursor.fetchall()

if not books:
    print("該当する本はありません")
for book in books:
    print(f"[{book['id']}] {book['title']} / {book['author']} ({book['published_year']}年)")

print(f"--- {len(books)} 件 ---")
```

実行します。まずは何も入力せずに Enter を押して、全件表示します。

```bash
python select_books.py
```

```
=== 本の一覧 ===
タイトルの検索語（空欄で全件表示）:
[1] 吾輩は猫である / 夏目漱石 (1905年)
[2] 銀河鉄道の夜 / 宮沢賢治 (1934年)
[3] 坊っちゃん / 夏目漱石 (1906年)
--- 3 件 ---
```

次に `猫` と入力して、検索できることを確認します。

| コード | 意味 |
|---|---|
| `LIKE %s` と `f"%{keyword}%"` | タイトルに検索語を **含む** 本を探す（SQLの `%` は「任意の文字列」） |
| `fetchall()` | 結果をすべて取り出す（1件だけなら `fetchone()`） |
| `ORDER BY id` | ID順に並べる |

> 💡 SELECT はデータを変更しないので、`commit()` は不要です。

### 9-5. 更新（UPDATE）

```bash
nano update_book.py
```

```python
from db import get_connection

print("=== 本の情報を更新 ===")
book_id = int(input("更新する本のID: "))

with get_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM books WHERE id = %s", (book_id,))
        book = cursor.fetchone()

        if book is None:
            print("そのIDの本は見つかりません")
        else:
            print(f"現在: {book['title']} / {book['author']} ({book['published_year']}年)")
            print("変更しない項目は空欄のまま Enter を押してください")
            title = input("新しいタイトル: ") or book["title"]
            author = input("新しい著者: ") or book["author"]
            year = input("新しい出版年: ") or book["published_year"]

            sql = "UPDATE books SET title = %s, author = %s, published_year = %s WHERE id = %s"
            cursor.execute(sql, (title, author, int(year), book_id))
            conn.commit()
            print(f"{cursor.rowcount} 件更新しました")
```

実行し、ID `3` の出版年だけを変更してみます。

```bash
python update_book.py
```

```
=== 本の情報を更新 ===
更新する本のID: 3
現在: 坊っちゃん / 夏目漱石 (1906年)
変更しない項目は空欄のまま Enter を押してください
新しいタイトル:
新しい著者:
新しい出版年: 1907
1 件更新しました
```

`python select_books.py` で変更を確認します。

> ⚠️ **`WHERE` を忘れると全件が書き換わります**
> `UPDATE books SET title = ...` だけで `WHERE id = ...` が無いと、**すべての本** が同じタイトルになってしまいます。UPDATE と DELETE では、必ず `WHERE` で対象を絞り込みましょう。
> このプログラムでは、更新の前に SELECT で対象を表示し、存在しないIDなら更新しないようにしています。

> 💡 `cursor.rowcount` には、SQLで影響を受けた行の数が入ります。

### 9-6. 削除（DELETE）

```bash
nano delete_book.py
```

```python
from db import get_connection

print("=== 本の削除 ===")
book_id = int(input("削除する本のID: "))

with get_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM books WHERE id = %s", (book_id,))
        book = cursor.fetchone()

        if book is None:
            print("そのIDの本は見つかりません")
        else:
            print(f"削除対象: {book['title']} / {book['author']}")
            answer = input("本当に削除しますか？ (y/n): ")
            if answer == "y":
                cursor.execute("DELETE FROM books WHERE id = %s", (book_id,))
                conn.commit()
                print(f"{cursor.rowcount} 件削除しました")
            else:
                print("削除を中止しました")
```

実行します。最初は `n` で中止し、次に `y` で削除してみましょう。

```bash
python delete_book.py
```

```
=== 本の削除 ===
削除する本のID: 2
削除対象: 銀河鉄道の夜 / 宮沢賢治
本当に削除しますか？ (y/n): y
1 件削除しました
```

> 💡 削除したデータは元に戻せません。そのため、削除の前に対象を表示して確認を求めるのが、実際のアプリでも一般的な作りです。

### 9-7. MySQL側から結果を確かめる

Pythonで行った操作が、本当にデータベースに反映されているかを確認します。

```bash
mysql -u libuser -p library
```

```sql
SELECT * FROM books;
EXIT;
```

Pythonのプログラムで登録・更新・削除した結果が、そのまま表示されます。プログラムもMySQLのコマンドも、**同じデータベースを操作している** ことが分かります。

### ✅ 確認ポイント

- [ ] 仮想環境に入り、PyMySQLをインストールできた
- [ ] `insert_book.py` で本を登録できた
- [ ] `select_books.py` で全件表示とタイトル検索ができた
- [ ] `update_book.py` で、指定した項目だけを変更できた
- [ ] `delete_book.py` で、確認のうえ本を削除できた
- [ ] `commit()` を外すと変更が保存されないことを確認した
- [ ] Pythonでの操作結果を、`mysql` コマンドでも確認できた

---

## 付録A. トラブルシューティング

| 症状 | 確認すること |
|---|---|
| 設定を変えたのに反映されない | 保存したか（`cat` で中身を確認）。保存後にサービスを再起動したか |
| 設定ファイルを壊してしまった | `.bak` から戻す：`sudo cp 設定ファイル.bak 設定ファイル` |
| nanoで保存できない | `sudo` を付けて開いたか |
| IPアドレスでつながらなくなった | `wsl --shutdown` や再起動でIPが変わっていないか。`hostname -I` で再確認 |
| `ipconfig` のIPでつながらない | vEthernet (WSL) はWindows側のIP。UbuntuのIPはUbuntu側で調べる |
| `systemctl` が動かない | `/etc/wsl.conf` に `systemd=true` があるか。追記後に `wsl --shutdown` したか |
| SSHで接続できない（22番） | Windows側で「OpenSSH Server」サービスが動いていてポートが競合していないか。競合時はWSL側のポートを2222などに変更 |
| 鍵認証にならない | `~/.ssh` が `700`、`authorized_keys` が `600` になっているか |
| パスワード禁止が効かない | `sudo sshd -T` で実際の値を確認。先に読まれる設定ファイルが無いか |
| `sudo` のパスワードが分からない | 3-0 のリセット手順を参照 |
| FTPで一覧が取得できない | パッシブモードのポート範囲（40000〜40100）がufwで許可されているか。クライアントがパッシブモードになっているか |
| ブラウザで403になる | ファイル `644`、ディレクトリ `755` 以上か |
| アップロードできない | `usermod` 後に再ログインしたか。`/var/www/html` の所有者・権限 |
| `mysql` で日本語が文字化けする | 8-5 の `default-character-set=utf8mb4` を設定したか。設定前に登録したデータは化けたままなので、削除して入れ直す |
| `Access denied for user 'libuser'` | パスワードの打ち間違い。`'libuser'@'localhost'` で作成したか |
| `mysql>` の後に `->` が出て進まない | SQLの最後の `;` を忘れている。`;` を入力して Enter |
| `error: externally-managed-environment` | 仮想環境に入っていない。`source .venv/bin/activate` |
| `ModuleNotFoundError: No module named 'pymysql'` | 仮想環境に入っていない、または PyMySQL を未インストール |
| `Can't connect to MySQL server` | MySQLが起動しているか（`sudo systemctl status mysql`） |
| 登録したのにデータが増えていない | `conn.commit()` を実行しているか |
| `ValueError: invalid literal for int()` | 出版年やIDに数字以外を入力した |

## 付録B. 発展課題

- **LAN内の別PCから公開する**：**【Windows】** `notepad $env:USERPROFILE\.wslconfig` でメモ帳を開き、次を記述・保存して `wsl --shutdown` し、Windowsファイアウォールで該当ポートを許可する

  ```ini
  [wsl2]
  networkingMode=mirrored
  ```

- **SFTP専用ユーザーを作る**：`sshd_config.d` に `Match User` と `ForceCommand internal-sftp`、`ChrootDirectory` を設定し、シェルログインできずファイル転送だけができるユーザーを作成する
- **FTPS（暗号化FTP）を試す**：vsftpdで `ssl_enable=YES` を設定し、FTPとの違いを `tcpdump` で比較する
- **メニュー形式のアプリにまとめる**：4つのプログラムを1つにまとめ、「1: 登録 / 2: 一覧 / 3: 更新 / 4: 削除 / 0: 終了」を選んで繰り返し操作できる `library_app.py` を作る
- **入力チェックを加える**：出版年に数字以外が入力されたら、エラーで止まらずに再入力を求めるようにする（`try` / `except ValueError`）
- **パスワードをプログラムから分離する**：`db.py` のパスワードを環境変数から読み込むように変更する（`os.environ`）
- **Webページに一覧を表示する**：Apacheと連携し、ブラウザで `books` の一覧を表示する

## 付録C. 振り返りの問い

1. `localhost` とWSLのIPアドレスは、それぞれどのような仕組みでUbuntuに届いているか
2. パスワード認証と鍵認証では、それぞれ何を使って本人確認をしているか
3. 3-3で、現在のSSHセッションを閉じずに作業したのはなぜか
4. SFTPとFTPの違いを、セキュリティの観点から説明せよ
5. WSLと実機のサーバーで、パスワードを忘れたときの復旧方法はどう違うか
6. ufwで「SSHを許可してから有効化する」順番が重要なのはなぜか
7. アプリ用に root ではなく `libuser` を作ったのはなぜか
8. MySQLの3306番ポートをufwで許可しなかったのはなぜか
9. SQLに値を入れるとき、文字列を直接埋め込まず `%s` を使うのはなぜか
10. `commit()` を実行しないと、何が起きるか
11. UPDATE や DELETE で `WHERE` を付け忘れると、何が起きるか
