# Windows 11 + WSL で学ぶ Webサーバ構築演習

## この演習のゴール

- WSL上のUbuntuにWebサーバ（Apache）を構築し、Webページを公開する
- SSHで「パスワード認証 → 鍵認証 → パスワード認証禁止」の流れを体験する
- SFTPとFTPでファイルを転送し、両者の違いを理解する
- ufwでファイアウォールを設定し、不要なポートを閉じる

## 前提条件

- Windows 11 に WSL2 がインストール済み
- Ubuntu 24.04 を想定（他のバージョンでは一部手順が異なる場合があります）
- 管理者権限で PowerShell を開けること

### 表記について

| 表記 | 意味 |
|---|---|
| **【Ubuntu】** | WSL上のUbuntuのターミナルで実行 |
| **【Windows】** | Windows側のPowerShellで実行 |
| `ユーザー名` | 自分のUbuntuユーザー名に置き換える |

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

記述が無い場合は追記し、**【Windows】** で WSL を再起動します。

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

**【Ubuntu】** 別のウィンドウで設定ファイルを作成します。

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin no
EOF
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

設定を追記します。

```bash
sudo tee -a /etc/vsftpd.conf <<'EOF'
write_enable=YES
local_umask=022
chroot_local_user=YES
allow_writeable_chroot=YES
local_root=/var/www/html
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
EOF
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

## 付録A. トラブルシューティング

| 症状 | 確認すること |
|---|---|
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

## 付録B. 発展課題

- **LAN内の別PCから公開する**：`%USERPROFILE%\.wslconfig` に次を記述して `wsl --shutdown` し、Windowsファイアウォールで該当ポートを許可する

  ```ini
  [wsl2]
  networkingMode=mirrored
  ```

- **SFTP専用ユーザーを作る**：`sshd_config.d` に `Match User` と `ForceCommand internal-sftp`、`ChrootDirectory` を設定し、シェルログインできずファイル転送だけができるユーザーを作成する
- **FTPS（暗号化FTP）を試す**：vsftpdで `ssl_enable=YES` を設定し、FTPとの違いを `tcpdump` で比較する

## 付録C. 振り返りの問い

1. `localhost` とWSLのIPアドレスは、それぞれどのような仕組みでUbuntuに届いているか
2. パスワード認証と鍵認証では、それぞれ何を使って本人確認をしているか
3. 3-3で、現在のSSHセッションを閉じずに作業したのはなぜか
4. SFTPとFTPの違いを、セキュリティの観点から説明せよ
5. WSLと実機のサーバーで、パスワードを忘れたときの復旧方法はどう違うか
6. ufwで「SSHを許可してから有効化する」順番が重要なのはなぜか
