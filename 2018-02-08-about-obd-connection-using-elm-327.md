# Linux における ELM327 を使った OBD 接続について

## はじめに

Debian 9 を使用して Bluetooth 経由で　ELM327 に接続し、実車に接続する方法などについて以下に纏めます。ELM327 との接続については Python 及び python-OBD を使用するものとします。また、不具合が出た事情より、当初はコンソールモードで Linux を使用する形をとりますが、別途動作の確認が取れれば X-Window を使う形での手順についても纏める方向です。

検証に使用した車両は H26 年式の TOYOTA SPADE (NSP140) となります。

以下に設定手順を列挙します。

- 環境設定
- Bluetooth 接続
- シリアルデバイスとの接続
- python-OBD での接続
- 出現した不具合について
- X-Window を使った接続の方法について

## 環境設定

Debian9 導入後、以下のパッケージを導入しています。

- rxvt-unicode-ml
- emacs
- git 
- tmux

また、Python については　Anaconda を使っています。以下より download を行います。

- [https://www.continuum.io/downloads](https://www.continuum.io/downloads)

導入後、`.bashrc` を読み込んだ後に以下のコマンドを実行しておきます。

```
$ pip install pyserial pint
```

また、一般ユーザとしてシリアルデバイスに接続する権限に関する設定が必要です。一点目はログインユーザ (ここでは rms とします) を　`dialout` グループに所属させます。

```
$ adduser rms dialout
```

その後、`/etc/udev/rules.d/50-udev-default.rules` を新規作成し、以下を入力しておきます。

```
KERNEL=="tty[A-Z]*[0-9]|pppox[0-9]*|ircomm[0-9]*|noz[0-9]*|rfcomm[0-9]*", GROUP="dialout", MODE="0666"
```

その後、udev サービスの再起動が必要です。

```
$ sudo service udev restart
```

また、ELM327 が通常 Modem などのデバイスに割り当てられる `/dev/rfcomm`　に割り当てられるため、色々な不具合が生じます。`modemmanager` というパッケージは削除しておきます。

```
$ sudo apt-get remove --purge modemmanager
```

当初はこの ModemManager が引き起こす問題により X-Window が使えないということとなり、コンソールモードで検証を行っています。Debian9 においてコンソールモードでログインする方法についても以下に記載しておきます。

以下のドキュメントによれば `default-display-manager` を編集する、という手法が推奨されていました。

- [Debian 8 JessieをCUIで起動する方法](http://note.kurodigi.com/debian8-cuilaunch/)

このため、`/etc/X11/default-display-manager` を以下のように編集しています。

```
#/usr/sbin/lightdm
```

上記ですが、導入時に display manager を `lightdb` として選択していることから上記の記述となっています。基本的にはコメントアウトすることでコンソールモードでログインすることが可能となります。

また、当初の検証においてはコンソールモードでの操作となっておりますので、各種接続などについてもコマンドを使っており、以下にその方法などんついて記載していくこととします。

### 正常に情報を取得した例

以下に Android のコンテンツアプリ (OBD Car Doctor free) にて取得した情報を列挙します。この情報が取得できれば　Linux における接続は成功ということになります。

- protocol : 6 - ISO15764-4(CAN11/500)
- adaptor : ELM32V1.5
- スタンダード OBD-II : JOBD(Japan)
- サポートされているコマンド
 - `0100 10111110001111111010100000010011`
 - `0120 10010000000101011011000000010101`
 - `0140 01111110110111000000000000000000`
 - `0900 00010100010000000000000000000000`

## Bluetooth 接続

GUI 操作では `blueman-manager` というツールを使うことで接続までの一切が可能ですが、コンソールモードでは `bluetoothctl` というコマンドを使ってペアリングまでの操作を行います。手順としては

- `bluetoothctl` コマンドにより bluetooth の　REPL　(コマンド入力、評価機能）　を起動します
- `scan on` コマンドを発行して周辺デバイスの探索を行います
- obd2 デバイスが見つかったら `scan off` しておきます
- ELM327 については PIN の入力が必要ですのでその用意をします
 - `agent off` コマンド発行
 - `agent KeyboardOnly` コマンドの発行
- `pair` コマンドにデバイス ID を渡す形でコマンドを発行すると以下のような形で　PIN の入力を要求されます

```
[bluetooth]# agent off         # unregister agent
Agent unregistered
[bluetooth]# agent KeyboardOnly   # register proper agent
Agent registered
[bluetooth]# pair XX:XX:XX:04:F5:7C 
Attempting to pair with XX:XX:XX:04:F5:7C 
[CHG] Device XX:XX:XX:04:F5:7C Connected: yes
Request passkey
[agent] Enter passkey (number in 0-999999): 1234
[MoarBacon]# pair XX:XX:XX:04:F5:7C 
Attempting to pair with XX:XX:XX:04:F5:7C 
[CHG] Device XX:XX:XX:04:F5:7C Paired: yes
Pairing successful
```

上記のように `Pairing successful` と表示されればペアリングに成功しています。

## シリアルデバイスとの接続

この状態ではまだペアリングした bluetooth デバイスをシリアルデバイスとして使うことができる状態にはなっていません。以下のコマンドを発行する必要があります。

```
$ sudo rfcomm bind 0 XX:XX:XX:04:F5:7C
```

接続遮断のコマンドは以下です。

```
$ sudo rfcomm unbind 0 XX:XX:XX:04:F5:7C
```

上記接続コマンドの発行により、`/dev/rfcomm0` というデバイスファイルが作成され、これを使って ELM327 との通信を行う形となります。

## python-OBD での接続

python-OBD については `pip` による導入ではなく、ソースコードを入手してこれを使う形をとります。ソースコードは以下のコマンドで入手します。

```
$ git clone https://github.com/brendan-w/python-OBD
```

カレントディレクトリは上記コマンドを実行した後に `python-OBD` ディレクトリにしておく必要があります。

```
$ cd python-OBD
```

ここまで正常に設定をすすめているのであれば以下のように接続を行うことができるはずです。

```
$ python 
Python 3.6.3 |Anaconda, Inc. | (default, Oct 13 2017, 12:02:49)
[GCC 7.2.0] on Linux
Type "help", "copyright", "credits", or "license" for more information.
>> import obd.obd
>> con = obd.OBD()
[obd.obd] ================ python-OBD (v0.6.1) ================
[obd.obd] Using scan_serial to select port
[obd.obd] Available ports: ['/dev/rfcomm0']
[obd.obd] Attempting to use port: /dev/rfcomm0
[obd.elm327] Initializing ELM327: PORT=/dev/rfcomm0 BAUD=auto PROTOCOL=auto
[obd.elm327] Response from baud 38400: b'\x7f\x7f\r?\r\r>''
[obd.elm327] Choosing baud 38400
[obd.elm327] write: b'ATZ\r\n'
[obd.elm327] wait: 1 seconds
[obd.elm327] read: b'ATZ\r\r\rELM327 v1.5\r\r>'
[obd.elm327] write: b'ATE0\r\n'
[obd.elm327] read: b'ATE0\rOK\r\r>'
[obd.elm327] write b'ATE0\r\n'
[obd.elm327] read: b'OK\r\r>'
[obd.elm327] write: b'ATH1\r\n'
[obd.elm327] read: b'OK\r\r>'
[obd.elm327] write: b'ATL0\r\n'
[obd.elm327] read: b'OK\r\r>''
[obd.elm327] write: b'ATST62\r\n'
[obd.elm327] read: b'OK\r\r>'
[obd.elm327] write: b'ATSP0\r\n'
[obd.elm327] read: b'OK\r\r'
[obd.elm327] write: b'0100\r\n'
[obd.elm327] read: b'SEARCHING...\r7E8 06 41 00 BE 3F A8 13 \r\r>'
[obd.elm327] write: b'ATDPN\r\n'
[obd.elm327] read: b'A6\r\r'
[obd.protocols.protocol] map ECU 0 -> ENGINE
[obd.elm327] connected Successfully: PORT=/dev/rfcomm0 BAUD=38400 PROTOCOL=4
[obd.obd] querying for supported commands
[obd.obd] Sending command: b'0100' : Supported PIDs [01-20]
[obd.elm327] write: b'0100\r\n'
[obd.elm327] read: b'7E8 06 41 00 BE 3F A8 13 \r\r>'
[obd.obd] Sending command: b'0120' : Supported PIDs [21-40]
[obd.elm327] write: b'0120\r\n'
[obd.elm327] read: b'7E8 06 41 20 90 15 B0 15 \r\r>'
[obd.obd] Sending command: b'0140' : Supported PIDs [41-60]
[obd.elm327] write b'0140\r\n'
[obd.elm327] read: b'7E8 06 41 40 7E DC 00 00 \r\r>'
[obd.obd] Sending command: b'0600' : Supported MIDs [01-20]
[obd.elm327] write b'0600\r\n'
[obd.elm327] read: b'7E8 06 46 00 C0 00 00 01 \r\r>'
[obd.obd] Sending command: b'0620' : Supported MIDs [21-40]
[obd.elm327] write b'0620\r\n'
[obd.elm327] read: b'7E8 06 46 20 80 00 80 01 \r\r>'
[obd.obd] Sending command: b'0640' : Supported MIDs [41-60]
[obd.elm327] write b'0640\r\n'
[obd.elm327] read: b'7E8 06 46 40 00 00 00 01 \r\r>'
[obd.obd] Sending command: b'0660' : Supported MIDs [61-80]
[obd.elm327] write b'0660\r\n'
[obd.elm327] read: b'7E8 06 46 60 00 00 00 01 \r\r>'
[obd.obd] Sending command: b'0680' : Supported MIDs [81-A0]
[obd.elm327] write b'0680\r\n'
[obd.elm327] read: b'7E8 06 46 80 00 00 00 01 \r\r>'
[obd.obd] Sending command: b'06A0' : Supported MIDs [A1-C0]
[obd.elm327] write b'06A0\r\n'
[obd.elm327] read: b'7E8 06 46 A0 F8 00 00 00 \r\r>'
[obd.obd] finished querying with 101 commands supported
[obd.obd] =====================================================

```

上記ですが、ログレベルを DEBUG にしているために出力されているものです。ログレベルの設定は `__init__.py` の以下の部分です。

```
logger.setLevel(logging.WARNING)
```

上記の通り、登録されているソースコードにおいてはログレベルは　`WARNING` 	となっています。

## 出現した不具合について

確認された不具合は以下となります。個別に詳細情報を記載します。

- X-Window が使用不可能となる
- ペアリングできない (bluetoothctl)
- 接続できない
- 初期化はできるがコマンドの発行ができない不具合

### X-Window が使用不可能となる

原因は未だに不明なのですが、これまでの試行錯誤を鑑みるに X-Window-System と ModemManager が何らかの形で競合し、gnome-shell が sigfault で異常終了しているようです。

これまで、二度ほど ELM327 との接続完了後にストールしてしまっていることを確認しています。現象としては以下です。

- X-Window においては入力ができない状態となる
- コンソールモードであれば入力は可能

この現象を回避するため、コンソールモードにて検証を行うこととしました。

### ペアリングできない (bluetoothctl)

PIN の入力方法が不明でしたが、agent コマンドの確認により、上記の通りペアリングを行う方法を確認し、ペアリングに成功しています。

### 接続できない

こちらも rfcomm コマンドの確認により、デバイスへの接続については動作の確認ができております。

### 初期化はできるがコマンドの発行ができない不具合

これについても modemmanager パッケージの何かとの競合が発生していたようです。パッケージを除去することでコマンドの発行ができました。

```
$ sudo apt-get remove --purge modemmanager
```

## X-Window を使った接続の方法について

- この節については別途動作確認でき次第情報投入の方向です。

