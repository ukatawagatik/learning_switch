# 問題
OpenFlow1.3 版スイッチの動作について，trema dump\_flows の出力を混じえながら各ステップごとに説明せよ

# 解答
以下のコマンドで OpenFlow1.3 にて Trema を起動し，各ステップごとにスイッチの動作の説明を示す．

```bash
$ trema run lib/learning_switch13.rb --openflow13 -c trema.conf
```

## 1. スイッチ起動時
スイッチが起動すると，以下の `switch_ready` ハンドラが呼び出される．

```ruby
  def switch_ready(datapath_id)
    add_multicast_mac_drop_flow_entry datapath_id
    add_default_forwarding_flow_entry datapath_id
    add_default_flooding_flow_entry datapath_id
  end
```

呼び出される各メソッドについて以下で概要を解説する．

`add_multicast_mac_drop_flow_entry` では，宛先 MAC アドレスがマルチキャスト等の予約アドレスであるパケットに対し，「何もしない（ドロップする）」ルールを最も高いプライオリティ（ここでは 2）でフィルタリング用 FlowTable（Table ID=0）に追加する．

`add_default_forwarding_flow_entry` では，フィルタリングを通過したパケットを転送用 FlowTable（Table ID=1）にて処理するために，「参照先テーブルを転送用 FlowTable に変更する」ルールをフィルタリングのルールより低いプライオリティ（ここでは 1）でフィルタリング用 FlowTable に追加する．

`add_default_flooding_flow_entry` では，転送用 FlowTable にヒットするエントリがないパケットを処理するために，「全てのパケットに対し Packet In とフラッディングを発生させる」ルールを最も低いプライオリティ（ここでは 1）で転送用 FlowTable に追加する．

スイッチが起動した直後における当該スイッチの FlowTable を以下に示す．なお，FlowTable のステータス表示には ovs-ofctl の dump-flows 機能（trema dump\_flows が内部で実行している機能）を用いた．

```
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=0.629s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=0.592s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
 cookie=0x0, duration=0.592s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535,FLOOD
```

結果から，上記に解説した動作が正しく行われていることがわかる．

## 2. host1 から host2 にパケットを送る
host1 から host2 にパケットを送った際の FlowTable のステータスを以下に示す．

```
 OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=22.038s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=22.001s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
 cookie=0x0, duration=1.969s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,dl_dst=a3:07:bb:0a:95:b3 actions=output:1
 cookie=0x0, duration=22.001s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535,FLOOD
```

上記では，3 行目の n\_packets が 1 増えており，これはフィルタリング用 FlowTable にて当該パケットがフィルタリングされずに転送用 FlowTable に参照先が変更されたことを意味する．そして 5 行目の n\_packets が 1 増えていることから，当該パケットは転送用 FlowTable のルールによって Packet In され，さらにフラッディングされていることがわかる．したがって，コントローラにて以下の `packet_in` ハンドラが呼び出される．

```ruby
  def packet_in(datapath_id, message)
    add_forwarding_flow_entry(datapath_id, message)
  end
```

`packet_in` ハンドラで呼び出される `add_forwarding_flow_entry` では，PacketIn の送信元 MAC アドレスと受信ポートから Forwarding Table のエントリを作成し，その転送ルールをフラッディングルールよりも低いプライオリティ（ここでは 1）で転送用 FlowTable に追加する．FlowTableのステータスにおける 4 行目のルールはこの際に作成されたルールである．

## 3. host2 から host1 にパケットを送る

host2 から host1 にパケットを送った際の FlowTable のステータスを以下に示す．

```
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=40.425s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=40.388s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
 cookie=0x0, duration=20.356s, table=1, n_packets=1, n_bytes=42, idle_timeout=180, priority=2,dl_dst=a3:07:bb:0a:95:b3 actions=output:1
 cookie=0x0, duration=40.388s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535,FLOOD

```

これより，当該パケットはフィルタリング用 FlowTable から転送用 FlowTable に処理が移り，さらに host1 宛てのエントリが既にあることから host1 宛てのポートに転送されることがわかる．しかしながらこのケースでは Packet In が生じないため，本来であればこの段階で新たに Forwarding Table のルール（host2 宛てのエントリ）が追加されることが望ましいが，それがされていない．宛先が host2 のエントリを FlowTable に作成するには，host2 から host1 以外（FlowTable にまだ登録されていない宛先）にパケットを送る必要があり，これはパフォーマンスを低下させる潜在的なバグであるといえる．

## 4. 再び host1 から host2 にパケットを送る
再び host1 から host2 にパケットを送った際の FlowTable のステータスを以下に示す．

```
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=59.983s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=59.946s, table=0, n_packets=3, n_bytes=126, priority=1 actions=goto_table:1
 cookie=0x0, duration=1.986s, table=1, n_packets=1, n_bytes=42, idle_timeout=180, priority=2,dl_dst=a3:07:bb:0a:95:b3 actions=output:1
 cookie=0x0, duration=59.946s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535,FLOOD
```

これより，当該パケットは 2. と同様の動作をたどり，フラッディングされる．Packet In が発生するため `add_forwarding_flow_entry` が呼び出されるが，既に同じエントリがテーブルに存在するため何もしない．
