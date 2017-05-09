# TeraTermから動的IPのEC2に簡単にSSHする方法

## 概要
WindowsからAWSのEC2インスタンスにSSH接続するには、TeraTermを使うことが多いと思われる。
接続するにはマクロが便利だが、マクロに書き込むIPアドレスが動的な場合には一工夫必要になる。
その一工夫を入れて、マクロ実行のみでSSH接続できるようにする。

## 前提条件
- 接続先EC2は動的なパブリックIPを持つ。
- EC2にはSSH認証でログインする。
- AWS CLIはインストール済みである。
- AWS CLIでEC2インスタンス情報を取得するためのプロファイルは設定済みである。
    - `aws configure --profile プロファイル名` で設定できる。
- EC2はNameで一意に特定できる。
    - ※別の方法でも一意に特定しても良い。

## 方法
### EC2のパブリックIPを取得するバッチ
プロファイル名とインスタンス名を渡し、該当するインスタンスのパブリックIPをip_work.txtというテキストファイルに書き込む。
ここで `--filters` の中身を調整すれば、Name以外でインスタンスを指定することもできる。

```bat:get_ec2_ip.bat
aws --profile %1 ec2 describe-instances --filters "Name=tag:Name,Values=%2" --query "Reservations[*].Instances[*].PublicIpAddress" --output text>ip_work.txt
```

### SSH接続するTeraTermマクロを作成

```:ssh_ec2.ttl
; 1. 動的IPアドレスを取得
; 1-1. バッチを実行し、テキストファイル(ip_work.txt)にIPを書き込む
exec 'cmd /c get_ec2_ip.bat プロファイル名 インスタンス名' 'hide' 1
; 1-2. ファイルからIPを取り出し変数にセット
;      (ファイルを仲介せず、コマンドの出力を直接受け取る方法を思い付かなかったので)
fileopen FH 'ip_work.txt' 0 0
filereadln FH HOST_ADDR
fileclose FH

; (2)ユーザ名／秘密鍵を設定
USER_NAME = 'ログインするユーザ名'
getenv 'HOME' KEY_FILE
strconcat KEY_FILE '\.ssh\秘密鍵ファイル(pem)'
 
; (3)コマンド組立て 
COMMAND = HOST_ADDR 
strconcat COMMAND ':22 /ssh /2 /auth=publickey /user=' 
strconcat COMMAND USER_NAME 
strconcat COMMAND ' /keyfile="' 
strconcat COMMAND KEY_FILE
; 設定ファイルを用意しているならここでセット
strconcat COMMAND '" /F=SETTING.INI' 
 
; (4)サーバへ接続 
connect COMMAND

; TeraTermのタイトルバーに任意の名前を記述しておくと分かりやすい
settitle 'タイトル名'
 
end
```

### TeraTermマクロを実行
作成したマクロをttpmacro.exe(TeraTerm本体のディレクトリにインストールされているはず)で実行すれば良い。
私は次のように運用している。

- 拡張子ttlのファイルをttpmacro.exeに関連付け
- 接続先ごとにこのマクロを用意
- 各マクロをコマンドランチャ(最近はAqulinaを使っている)に登録
