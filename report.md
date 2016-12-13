# 第8回 (11/30)レポート(team-amn:東野研)
### メンバー
* 今井 友揮
* 成元 椋祐
* 西村 友佑
* 原 佑輔
* 三浦 太樹

## 1. スライスの分割・結合
### 1.1 コマンドの仕様
　スライスの分割ではサブコマンド`split`を定義した．
[講義資料](http://handai-trema.github.io/deck/week8/sliceable_switch.pdf)を参考に、スライスに属する各ホストの指定にはMACアドレスを用いるものとして、サブコマンドの仕様を定義した．
　host1(MACアドレス: 11:11:11:11:11:11)・host2(22:22:22:22:22)・host3(33:33:33:33:33:33)・host4(44:44:44:44:44:44)が属するslice_aを、host1・host2が属するslice_bと、host3・host4が属するslice_cに分割する際は以下のようにコマンドを実行する．
```
./bin/slice split slice_a --into slice_b:11:11:11:11:11:11, 22:22:22:22:22:22 slice_c:33:33:33:33:33:33, 44:44:44:44:44:44
```

　スライスを分割する際に、分割元のスライスに属する全てのホストを分割先のスライスに割り当てなかった場合、分割元のスライスは残す仕様とした．

　スライスの結合ではサブコマンド`join`を定義した．
コマンドの仕様は、引数として結合元のスライスをスペースで区切って指定し、結合後の新たなスライス名を`--into`オプションで指定するものとした．
２つのスライスslice_aとslice_bを結合し、新たにslice_cを作成する際のコマンド例は以下となる．

```
./bin/slice join slice_a slice_b --into slice_c
```

### 1.2 実装内容
 　スライスの分割・結合を実装するにあたって修正、作成した主なファイルについて説明する．
* /bin/slice
 * スライスの分割・結合を実行する際のコマンドを定義
* /lib/slice.rb
 * スライスの分割・結合を行うメソッドとしてsplitとjoinを追加

## 2. スライスの可視化
### 2.1 使用方法
　/output/index.htmlによって表示されるトポロジ図において、ホストの属するスライスの情報を重ねて表示する．index.htmlでは、トポロジ図におけるホストのラベルとして出力されるMACアドレスの文字の色をスライスごとに異なる色で表示される．  
　index.htmlは1秒間隔で更新されるが、表示間隔の更新にはタイムスタンプの取得が必要である．事前準備として/output/server.shを起動する必要がある．

### 2.2 ファイルの呼び出し関係図
　以下にファイルを介して行っているコントローラプロセス~ブラウザ間のトポロジ情報，スライス情報のやり取りの関係を図で示す．

![関係図](./fileflow.png)

### 2.3 実装内容
* /lib/slice.rb
 * スライスの出力ファイルslice.jsを出力するメソッドwrite_slice_infoを追加．このメソッドでは、Sliceクラスのクラス変数allを参照し、スライスごとに生成されるSliceクラスのインスタンスから、そのスライスに属するホストのmacアドレスを取得し、スライスの情報をノードのラベルの色情報として付加してslice.jsに出力している．


## 3. REST APIの追加
### 3.1 実装内容
スライスの分割・結合を行う機能を持つREST APIを追加するために，以下の通り ```lib/rest_api.rb```に，分割を行う```Split a slice.```，結合を行う```Join a slice.``` から始まるブロックをそれぞれ追加した．

```
  desc 'Split a slice.'
  params do
    requires :base_slice_id, type: String, desc: 'Base slice.'
    requires :into_slices_id, type: String, desc: 'Into slices(multiple).'
  end
  get 'base_slice_id/:base_slice_id/into_slices_id/:into_slices_id' do
    rest_api do
      arr = params[:into_slices_id].split(",")
      Slice.split(params[:base_slice_id], arr[0],arr[1])
    end
  end
```
```
  desc 'Join a slice.'
  params do
    requires :base_slices_id, type: String, desc: 'Base slices(multiple).'
    requires :into_slice_id, type: String, desc: 'Into slice.'
  end
  get 'base_slices_id/:base_slices_id/into_slice_id/:into_slice_id' do
    rest_api do
      Slice.join(params[:base_slices_id].split(","), params[:into_slice_id])
    end
  end
```

既に実装されていた他のブロックを参考にしてこれらを実装した．```Slice```クラスにおいて定義した，スライスの分割を行うための```split```メソッド，スライスの結合を行うための```join```メソッドをそれぞれのブロックが呼び出す処理を行う．  
以下にスライスの分割，結合それぞれのコマンドについての簡素な説明を示す．

### ①分割
第１入力引数に分割される元のスライスID```base_slice_id```，第２入力引数に分割後のそれぞれのスライスID(複数個)である```into_slices_id```を入力することで，REST APIによるスライスの分割機能を実現する．このとき，第２引数には```,```区切りで分割先の複数のスライスIDとスライスに加えるホストのMACアドレスの組を入力する必要がある．  
以下に分割機能のコマンドの使い方の例を示す．  
```
curl -sS -X GET 'http://localhost:9292/base_slice_id/slice_a/into_slices_id/slice_b:11:11:11:11:11:11,slice_c:22:22:22:22:22:22'
```

### ②結合
第１入力引数に結合したいスライス（複数）を```,```で区切ったID，第２入力引数に結合後に新たに生成されるスライスIDを入力することで，REST APIによるスライスの結合機能を実現する．  
以下に統合機能のコマンドの使い方の例を示す．  
```
curl -sS -X GET 'http://localhost:9292/base_slices_id/slice_a,slice_b/into_slice_id/slice_c'
```


## 4. 仮想ネットワークでの動作検証
### 4.1 スライスの分割・結合
　./trema.confでのネットワークトポロージを使用し、仮想ネットワークにおいて動作を検証した．コントローラを起動後、コマンドライン上でスライスを定義した上で、スライスの分割・結合を行い、ブラウザ上で表示されるトポロジ図を確認した．
#### STEP.1 スライスの作成
スライスの作成のために以下のコマンドを実行した．

```
$ ./bin/slice add slice_a
$ ./bin/slice add_host --mac 11:11:11:11:11:11 --port 0x1:1 --slice slice_a
$ ./bin/slice add_host --mac 22:22:22:22:22:22 --port 0x4:1 --slice slice_a
$ ./bin/slice add_host --mac 33:33:33:33:33:33 --port 0x5:1 --slice slice_a
$ ./bin/slice add_host --mac 44:44:44:44:44:44 --port 0x6:1 --slice slice_a
```
![](./step1.png)

しかし，このままではホストの位置をコントローラが把握することが出来ないために以下のようにパケットを送信することで，各ホストの位置を検出できる．

```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema send_packets --source host2 --dest host3
$ ./bin/trema send_packets --source host3 --dest host4
$ ./bin/trema send_packets --source host4 --dest host1
```
以下にホストの接続状況が把握できた時のトポロジ情報を示す．
![](./step2.png)
また，この時の各ホストのパケット送受信の状況を以下に示す．

```
$ ./bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
Packets received:
  192.168.0.4 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host2
Packets sent:
  192.168.0.2 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ ./bin/trema show_stats host3
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.3 = 1 packet
$ ./bin/trema show_stats host4
Packets sent:
  192.168.0.4 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.3 -> 192.168.0.4 = 1 packet
```

#### STEP.2 スライスの分割
スライスの分割を以下のコマンドで実行した．

```
$ ./bin/slice split slice_a --into slice_b:11:11:11:11:11:11,33:33:33:33:33:33 slice_c:22:22:22:22:22:22,44:44:44:44:44:44
$ ./bin/slice list
slice_b
  0x1:1
    11:11:11:11:11:11
  0x5:1
    33:33:33:33:33:33
slice_c
  0x4:1
    22:22:22:22:22:22
  0x6:1
    44:44:44:44:44:44
```

スライスのlistコマンドでスライスが分割出来ていることができる．
また，以下にブラウザ上でのスライスの状況を示す．スライスが分割でき，スライスごとに色分けされていることが分かる．

![](./step3.png)

#### STEP.3 パケットの送信
host1とhost3をslice\_bにhost2とhost4をslice\_cに分割したので，host1とhost2，また，host3とhost4同士はパケットの送受信ができない．これを以下のコマンドで確認する．

```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema send_packets --source host1 --dest host3
$ ./bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 2 packets
  192.168.0.1 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.4 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host2
Packets sent:
  192.168.0.2 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
 ./bin/trema show_stats host3
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.3 = 1 packet
  192.168.0.1 -> 192.168.0.3 = 1 packet
```
このようにhost1とhost2，また，host3とhost4同士はパケットの送受信ができないので，ネットワークのスライスが正しく行えていることが確認できた．

#### STEP.4 スライスの結合
スライスの結合と確認を以下のコマンドで実行した．

```
$ ./bin/slice join slice_b slice_c --into slice_a
$ ./bin/slice list
slice_a
  0x1:1
    11:11:11:11:11:11
  0x5:1
    33:33:33:33:33:33
  0x4:1
    22:22:22:22:22:22
  0x6:1
    44:44:44:44:44:44
```
先程分割されたスライスが正しく結合されている事がわかる．
以下にブラウザでの可視化の図を示す．

![](./step4.png)

#### STEP.5 パケットの送信
スライスが結合されたので，host1とhost2，host3とhost4の間でパケットの通信が行えるはずである．その確認を以下のコマンドで行った．

```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema send_packets --source host1 --dest host3
$ ./bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 3 packets
  192.168.0.1 -> 192.168.0.3 = 2 packets
Packets received:
  192.168.0.4 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host2
Packets sent:
  192.168.0.2 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 2 packets
$ ./bin/trema show_stats host3
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.3 = 1 packet
  192.168.0.1 -> 192.168.0.3 = 2 packets
```

正しくパケットの送受信が行われており，スライスの結合が正常に行えていることが分かる．


### 4.2 REST API
今回実装した，スライスの分割・結合機能のREST APIが正常に動作するかどうかについて確認を行った．  
まず，REST APIを起動するためにスライス機能付きスイッチを起動し，その後```rackup```コマンドでWEBrickを起動する．
```
$ ./bin/trema run ./lib/routing_switch.rb -c trema.conf -d -- --slicing
$ ./bin/rackup
```

次に，以下に示すコマンドで2つのスライス```slice1```, ```slice2``` を作成し，それぞれのスライスに指定したdpid，ポート番号，MACアドレスを持つホストを1つずつ追加してそれぞれのスライスの様子を確認した．詳細な説明はテキストに記載されているためここでは省略する．

```
$ curl -sS -X POST -d '{"name": "slice1"}' 'http://localhost:9292/slices' -H Content-Type:application/json -v
$ curl -sS -X POST -d '{"name": "slice2"}' 'http://localhost:9292/slices' -H Content-Type:application/json -v
$ curl -sS -X POST -d '{"name": "11:11:11:11:11:11"}' 'http://localhost:9292/slices/slice1/ports/0x1:1/mac_addresses' -H Content-Type:application/json -v
$ curl -sS -X POST -d '{"name": "44:44:44:44:44:44"}' 'http://localhost:9292/slices/slice2/ports/0x6:6/mac_addresses' -H Content-Type:application/json -v
```

上記コマンド実行後のブラウザの様子を以下の図に示す．このとき，ホスト間ではパケットの送受信を行うことはできなかった．

![](./initial.png)


そして，先ほど作成した2つのスライス```slice1```と```slice2```を結合し，新たに```slice3```を生成することを試みる．```curl```コマンドを用いて，```slice1```と```slice2```の結合を行うメッセージをGETメソッドでサーバに対して送った．
```
curl -sS -X GET 'http://localhost:9292/base_slices_id/slice1,slice2/into_slice_id/slice3'
```
このとき，スライスの結合が正しく動作したことが，以下の図の色分けより確認できる．実行後，異なるスライス(異なる色分け)だったものが同一のスライス(同じ色分け)へブラウザ上で遷移している．また，このときホスト間でパケットの送受信を行うことができた．

![](./join2.png)


次に，只今作成した```slice3```(ホストが2つ繋がっている状態)を2つのスライス```slice4```，```slice5```に分割することを試みる．```curl```コマンドを用いて，上記の処理を行うメッセージをGETメソッドでサーバに対して送った．

```
curl -sS -X GET 'http://localhost:9292/base_slice_id/slice3/into_slices_id/slice4:11:11:11:11:11:11,slice5:44:44:44:44:44:44'
```

このとき，正しくスライスの分割が動作したことが，以下の図の色分けよりわかる．実行後，同一のスライス(同じ色分け)から異なるスライス(異なる色分け)へとブラウザ上で遷移している．また，このときホスト間でパケットの送受信を行うことはできなかった．

![](./split2.png)


## 5. 実機での動作検証
### 5.1 実機で検証する際のメモ
実機でsliceable-switchを動作させる上で、発見した課題とその解決策を述べる．
#### フローエントリの重複
　実機を用いた動作検証において、ping等で発信元と送信先の同じパケットを複数回送信すると、発信元と送信先の同じのフローエントリが複数生成される事を確認した．  
　この現象は/lib/path.rbがpacket_inを処理する際に、フローエントリへ書き加える処理のマッチングルールをpacket_inの発生したパケットと"全く同じパケットであった場合"としたためと考えられる．pingのように同じパスを経由するパケットであっても、各パケットはICMPが異なるので、初めに到着したパケットによって追加されたフローエントリは次のパケットにマッチしないとハードウェアが判断し、再びpacket_inが発生した結果と考えられる．そこで、新しく追加するフローエントリのマッチフィールドをpacket_inの発生するパケットと全く同じパケットではなく、宛先ipアドレスが同じであることを条件とした．  
　この際、OpenFlowの仕様として宛先IPのみをマッチングルールにできないので、IPv4パケットとARPパケットのイーサタイプを同時にマッチングルールの条件として指定した．

#### ARP解決について
　実機の検証ではARP解決ができないという課題が発生した．そこで、検証を行う際はパケットの送信元ホストのターミナルで送信先ホストのIPアドレスとMACアドレスをARPテーブルに予め登録した．

### 5.2 ネットワークの構成
　実機を用いた検証では，スイッチとしてVLANが６個（dpid:0x1..6），ホストとしてPCを４台接続したネットワークを構築した．  
　各ホストに割り当てたipとMACアドレスの関係は以下の通り．

||IPアドレス|MACアドレス|接続するVLANのdpid|機種|
|:--:|:--|:--:|:--:|:--:|
|host1|192.168.0.100|34:95:db:11:e6:14|0x2|MacBookPro|
|host2|192.168.0.11|00:25:4b:fd:d8:0d|0x5|MacBookAir|
|host3|192.168.0.50|74:03:bd:3d:34:9d|0x1|MacBook|
|host4|192.168.0.4|34:95:db:2c:67:53|0x11|MAcBookPro|

### 5.3 検証した内容
#### STEP.1 スライスの作成
　まず，以下のコマンドを実行し，全てのホストが属するスライスslice_a作成した．

```
$ ./bin/slice add slice_a
$ ./bin/slice add_host --mac 34:95:db:11:e6:14 --port 0x2:5 --slice slice_a
$ ./bin/slice add_host --mac 00:25:4b:fd:d8:0d --port 0x5:14 --slice slice_a
$ ./bin/slice add_host --mac 74:03:bd:3d:34:9d --port 0x1:2 --slice slice_a
$ ./bin/slice add_host --mac 34:95:db:2c:67:53 --port 0x4:11 --slice slice_a
```
　しかし，このままではホストの位置をコントローラが把握することが出来ない．そこで、host2のターミナルから残りの3つのホストに対し以下のようにpingを送信して各ホストの位置を検出した．

```
$ ping 192.168.0.4
PING 192.168.0.4 (192.168.0.4): 56 data bytes
64 bytes from 192.168.0.4: icmp_seq=0 ttl=64 time=222.026 ms
64 bytes from 192.168.0.4: icmp_seq=1 ttl=64 time=218.299 ms
64 bytes from 192.168.0.4: icmp_seq=2 ttl=64 time=0.851 ms
64 bytes from 192.168.0.4: icmp_seq=3 ttl=64 time=0.935 ms
^C
--- 192.168.0.4 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.851/110.528/222.026/109.643 ms

$ ping 192.168.0.100
PING 192.168.0.100 (192.168.0.100): 56 data bytes
64 bytes from 192.168.0.100: icmp_seq=0 ttl=64 time=239.911 ms
64 bytes from 192.168.0.100: icmp_seq=1 ttl=64 time=250.474 ms
64 bytes from 192.168.0.100: icmp_seq=2 ttl=64 time=1.019 ms
64 bytes from 192.168.0.100: icmp_seq=3 ttl=64 time=1.010 ms
64 bytes from 192.168.0.100: icmp_seq=4 ttl=64 time=0.963 ms
^C
--- 192.168.0.100 ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.963/98.675/250.474/119.677 ms

$ ping 192.168.0.50
PING 192.168.0.50 (192.168.0.50): 56 data bytes
64 bytes from 192.168.0.50: icmp_seq=0 ttl=64 time=421.629 ms
64 bytes from 192.168.0.50: icmp_seq=1 ttl=64 time=1.073 ms
^C
--- 192.168.0.50 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 1.073/211.351/421.629/210.278 ms
```
　各ホストに対して複数回送信したpingのうち，2〜3個目以降の応答時間は．1〜2個目のものと比較すると，かなり短くなっていることが確認できる．これは，最初のpacket_inによってフローエントリが作成され，以降のパケットの転送はコントローラの判断を経由せずにハードウェアによって処理されるようになったことが高速化の要因と考えられる．  
　pingによって他のホストの接続状況が把握でき,結果として得られたトポロジ図は以下のようになった．
![](./real_step1.png)


#### STEP.2 スライスの分割
　続いて，以下のコマンドによりスライスslice_aを、host1とhost2が属するslice_b、host3とhost4の属するslice_cの2つのスライスに分割した．
```
$ ./bin/slice split slice_a --into slice_b:34:95:db:11:e6:14,00:25:4b:fd:d8:0d slice_c:74:03:bd:3d:34:9d,34:95:db:2c:67:53
$ ./bin/slice list
slice_b
  0x2:5
    34:95:db:11:e6:14
  0x5:14
    00:25:4b:fd:d8:0d
slice_c
  0x1:2
    74:03:bd:3d:34:9d
  0x4:11
    34:95:db:2c:67:53
```

　スライスのlistコマンドでスライスが分割出来ていることが確認できる．この時ブラウザ上に表示されたトポロジ図を以下に示す．
![](./real_step2.png)
　この図から，スライスの分割が成功し，色分けして表示できている事が確認できる．

#### STEP.2-2 パケットの送信による確認
　ここで，host2のターミナルから残りの3つのホストに対し，以下のようにpingを送信することで，スライスによってパケットの送受信が管理できているかを確認した．ここでは，host1とhost2をslice\_bに，host3とhost4をslice\_cに分割したので，
host2からhost1(192.168.0.100)へはパケットの送信が成功するが，host3(192.168.50)とhost4(192.168.0.4)にはパケットの送信が失敗することを予期している．
```
$ ping 192.168.0.100
PING 192.168.0.100 (192.168.0.100): 56 data bytes
64 bytes from 192.168.0.100: icmp_seq=0 ttl=64 time=432.455 ms
64 bytes from 192.168.0.100: icmp_seq=1 ttl=64 time=0.820 ms
64 bytes from 192.168.0.100: icmp_seq=2 ttl=64 time=0.964 ms
64 bytes from 192.168.0.100: icmp_seq=3 ttl=64 time=0.949 ms
^C
--- 192.168.0.100 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.820/108.797/432.455/186.864 ms

$ ping 192.168.0.50
PING 192.168.0.50 (192.168.0.50): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
^C
--- 192.168.0.50 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss

$ ping 192.168.0.4
PING 192.168.0.4 (192.168.0.4): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
^C
--- 192.168.0.4 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss
```

　以上の結果より，host2からhost1へのパケットは送信できている一方で、slice_cに属するホストへの送信は失敗しており、スライスの管理が正しく行われていることを確認した．

#### STEP.3 スライスの結合
　STEP2で分割したスライスを，以下のコマンドで再び一つのスライスslice_aに統合した．
```
$ ./bin/slice join slice_b slice_c --into slice_a
$ ./bin/slice list
slice_a
  0x2:5
    34:95:db:11:e6:14
  0x5:14
    00:25:4b:fd:d8:0d
  0x1:2
    74:03:bd:3d:34:9d
  0x4:11
    34:95:db:2c:67:53
```
　この時のブラウザでの可視化の図を以下に示す．
![](./real_step4.png)

#### STEP.3-2 パケットの送信による確認
　スライスが結合後は、全てのホストが同じスライスに属している．そこで、以下のコマンドでhost2のターミナルから他の全てのホストにパケットを送信できるか確認した．
```
$ ping 192.168.0.100
PING 192.168.0.100 (192.168.0.100): 56 data bytes
64 bytes from 192.168.0.100: icmp_seq=0 ttl=64 time=223.546 ms
64 bytes from 192.168.0.100: icmp_seq=1 ttl=64 time=1.036 ms
64 bytes from 192.168.0.100: icmp_seq=2 ttl=64 time=0.857 ms
64 bytes from 192.168.0.100: icmp_seq=3 ttl=64 time=0.668 ms
^C
--- 192.168.0.100 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.668/56.527/223.546/96.429 ms

$ ping 192.168.0.50
PING 192.168.0.50 (192.168.0.50): 56 data bytes
64 bytes from 192.168.0.50: icmp_seq=0 ttl=64 time=422.579 ms
64 bytes from 192.168.0.50: icmp_seq=1 ttl=64 time=0.953 ms
64 bytes from 192.168.0.50: icmp_seq=2 ttl=64 time=0.987 ms
^C
--- 192.168.0.50 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.953/141.506/422.579/198.748 ms

$ ping 192.168.0.4
PING 192.168.0.4 (192.168.0.4): 56 data bytes
64 bytes from 192.168.0.4: icmp_seq=0 ttl=64 time=224.851 ms
64 bytes from 192.168.0.4: icmp_seq=1 ttl=64 time=0.622 ms
64 bytes from 192.168.0.4: icmp_seq=2 ttl=64 time=1.001 ms
^C
--- 192.168.0.4 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.622/75.491/224.851/105.613 ms
```
　実際に、パケットの送受信は全てのホストに対して成功しており、スライスの結合が正しくできていることが確認できた．