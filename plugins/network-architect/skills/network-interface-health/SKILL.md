---
name: network-interface-health
description: このスキルは、ユーザーが「インターフェースで CRC やエラーが出ている」「パケットロスや断続的な疎通不良がある」「インターフェースがフラップしている」「ドロップやカウンタを確認したい」「デュプレックスミスマッチを疑っている」「機器交換前に両端のカウンタを比較したい」と尋ねたときに使用する。ルーター・スイッチ・Linux ホストのインターフェースエラー・ドロップ・CRC・デュプレックスミスマッチ・フラップ・速度ネゴシエーション問題の診断パターンを提供する。
argument-hint: "[調査対象のインターフェース名と症状の説明]"
allowed-tools: ["Read", "Grep", "Glob"]
---

# インターフェース健全性スキル

ネットワーク症状の原因が物理リンク・スイッチポート・ケーブル・トランシーバー・デュプレックス設定・輻輳したインターフェースにある可能性があるときに使用する。

## 使用するタイミング

- ホストまたは VLAN にパケットロス・レイテンシスパイク・断続的な疎通不良がある。
- スイッチまたはルーターのインターフェースで CRC・ランツ・ジャイアント・ドロップ・リセット・フラップが発生している。
- ハードウェアを交換する前にリンクの両端を比較する必要がある。
- 変更ウィンドウでインターフェースカウンタの前後エビデンスが必要。
- 監視から `ifInErrors`・`ifOutErrors`・`ifOutDiscards` の増加が報告されている。

## 動作原理

インターフェースカウンタはエビデンスだが、絶対値よりトレンドが重要。ベースラインを取得し、測定インターバルを待ち、再取得して増分を比較する。

```text
show interfaces <interface>
show interfaces <interface> status
show logging | include <interface>|changed state|line protocol
```

Linux ホストの場合:

```text
ip -s link show <interface>
ethtool <interface>
ethtool -S <interface>
```

## カウンタリファレンス

| カウンタ | 意味 | 主な原因 |
| --- | --- | --- |
| CRC | 受信フレームのチェックサム失敗 | 不良ケーブル・汚れたファイバー・不良オプティック・デュプレックスミスマッチ |
| input errors | 受信側エラーの集計 | サブカウンタを確認してから結論を出す |
| runts | Ethernet 最小フレームサイズ未満 | デュプレックスミスマッチ・コリジョンドメイン・NIC 障害 |
| giants | 想定 MTU を超えるフレーム | MTU ミスマッチまたはジャンボフレームの境界 |
| input drops | デバイスがインバウンドパケットを受け付けられない | バースト・オーバーサブスクリプション・CPU パス・キュー圧力 |
| output drops | 出力キューがパケットを破棄 | 輻輳・QoS ポリシー・不足したアップリンク |
| resets | インターフェースのハードウェアリセット | フラップ・キープアライブ・ドライバ・オプティック・電源 |
| collisions | Ethernet コリジョンカウンタ | ハーフデュプレックスまたはネゴシエーションミスマッチ |

## 診断フロー

### CRC またはインプットエラー

1. カウンタが増加中かを確認する（過去の履歴だけでないか）。
2. リンクの両端を確認する。受信側エラーは通常そのポートに届く信号を指し示す。エラーを報告しているポート自体が原因とは限らない。
3. パッチケーブルを交換するか、ファイバーとオプティックを清掃・交換する。
4. 両端の速度/デュプレックス設定が一致していることを確認する。
5. 同じタイムスタンプ周辺のフラップイベントをログで確認する。

### ドロップ

1. インプットドロップとアウトプットドロップを分ける。
2. インターフェースレートと帯域を比較する。
3. QoS ポリシー・キューカウンタ・リンクがオーバーサブスクライブのアップリンクかを確認する。
4. キューチューニングは二次対応として扱う。まずリンクが輻輳しているかを証明する。

### デュプレックスと速度

現代の Ethernet リンクで両端が対応している場合は自動ネゴシエーションを優先する。一方を固定にする必要がある場合は両端を明示的に設定し、理由を記録する。片方を固定・もう一方をオートに設定する「混在」は絶対に行わない。

```text
show interfaces <interface> | include duplex|speed
```

## 安全なパーサー例

各インターフェースブロックを次のヘッダーまでの範囲でスライスする。任意の文字数ウィンドウは使用しない。大きなインターフェースブロックでカウンタが欠落したり誤ったポートに割り当てられたりする可能性がある。

```python
import re
from typing import Any

HEADER_RE = re.compile(
    r"^(?P<name>\S+) is (?P<status>(?:administratively )?down|up), "
    r"line protocol is (?P<protocol>up|down)",
    re.I | re.M,
)
ERROR_RE = re.compile(r"(?P<input>\d+) input errors, (?P<crc>\d+) CRC", re.I)
DROP_RE = re.compile(r"(?P<output>\d+) output errors", re.I)
DUPLEX_RE = re.compile(r"(?P<duplex>Full|Half|Auto)-duplex,\s+(?P<speed>[^,]+)", re.I)

def parse_show_interfaces(raw: str) -> list[dict[str, Any]]:
    headers = list(HEADER_RE.finditer(raw))
    interfaces = []
    for index, header in enumerate(headers):
        end = headers[index + 1].start() if index + 1 < len(headers) else len(raw)
        block = raw[header.start():end]
        errors = ERROR_RE.search(block)
        drops = DROP_RE.search(block)
        duplex = DUPLEX_RE.search(block)
        interfaces.append({
            "name": header.group("name"),
            "status": header.group("status"),
            "protocol": header.group("protocol"),
            "duplex": duplex.group("duplex") if duplex else "unknown",
            "speed": duplex.group("speed").strip() if duplex else "unknown",
            "input_errors": int(errors.group("input")) if errors else 0,
            "crc_errors": int(errors.group("crc")) if errors else 0,
            "output_errors": int(drops.group("output")) if drops else 0,
        })
    return interfaces
```

## 適用例

### 1 つのスイッチポートで CRC が発生

1. ローカルポートのカウンタを取得する。
2. 接続先リモートポートのカウンタを取得する。
3. ルーティングやファイアウォールルールを変更する前にケーブルまたはオプティックを交換する。
4. ベースラインを記録した後にのみカウンタをクリアする。
5. 固定インターバル後に再確認する。

### インターネットが遅いが LAN は問題ない

1. WAN インターフェースのドロップ/エラーを確認する。
2. LAN アップリンクの使用率とアウトプットドロップを確認する。
3. WAN リンクはクリーンだがスループットが低い場合、ゲートウェイの CPU を確認する。
4. 上流サービスのせいにする前に有線と無線でテストを比較する。

## アンチパターン

- ベースラインを保存する前にカウンタをクリアする。
- リンクの片側だけを確認する。
- 時間窓なしに過去のすべての CRC を現在進行中の問題と見なす。
- 片方をオート、もう一方を固定速度/デュプレックスに設定する。
- ドロップを輻輳の確認前にケーブル問題として扱う。

## 参照

- エージェント: `network-architect`
- スキル: `network-config-validation`
- スキル: `cisco-ios-patterns`
- スキル: `network-bgp-diagnostics`
