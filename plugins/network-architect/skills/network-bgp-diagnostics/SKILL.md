---
name: network-bgp-diagnostics
description: このスキルは、ユーザーが「BGP ネイバーが Established にならない」「BGP セッションがダウン・フラップしている」「期待しているルートが来ていない」「route-map や prefix-list でルートがフィルタされていないか確認して」「BGP 変更前後のエビデンスを収集して」と尋ねたときに使用する。BGP ネイバー状態・ルート交換・プレフィックスポリシー・AS パス検査と安全なエビデンス収集の診断専用パターンを提供する。
argument-hint: "[調査対象のBGPネイバーIPまたは事象の説明]"
allowed-tools: ["Read", "Grep", "Glob"]
---

# BGP 診断スキル

BGP セッションがダウン・フラップしている場合、Established だが期待したルートが見えない場合、または予期しないプレフィックスをアドバタイズしている場合に使用する。デフォルトのワークフローは読み取り専用のエビデンス収集であり、ポリシー変更やリセット操作はレビュー済みの変更ウィンドウで実施する。

## 使用するタイミング

- BGP ネイバーが Idle・Connect・Active・OpenSent・OpenConfirm で止まっている。
- セッションが Established だが期待するプレフィックスが見えない。
- route-map・prefix-list・max-prefix 制限・AS パスポリシーがルートをフィルタしている可能性がある。
- BGP 変更の前後エビデンスが必要。
- BGP サマリー出力をパースする自動化をレビューしている。

## 読み取り専用トリアージフロー

1. 正確なネイバー・アドレスファミリー・VRF・ローカル/リモート ASN を特定する。
2. サマリー状態と最後のリセット理由を取得する。
3. ピアのソースアドレスへの到達性を確認する。
4. トランスポート障害と決めつける前にルートポリシーの参照を確認する。
5. プラットフォームが対応している場合、アドバタイズ済み・受信済み・インストール済みルートを比較する。

```text
show bgp summary
show bgp neighbors <peer>
show ip route <peer>
show tcp brief | include <peer>|:179
show logging | include BGP|<peer>
show running-config | section router bgp
show ip prefix-list
show route-map
```

デバイスが VRF・IPv6・VPNv4・EVPN を使用している場合はプラットフォーム固有のアドレスファミリーコマンドを使用する。グローバル IPv4 ユニキャストを前提としない。

## 状態の解釈

| 状態 | 最初に確認すること |
| --- | --- |
| Established (プレフィックス数あり) | ルート交換は動作中。ポリシーとテーブル選択を確認 |
| Established (ゼロプレフィックス) | インバウンドポリシー・max-prefix・アドバタイズ済みルート・AFI/SAFI を確認 |
| Active | TCP セッションが完了していない。ルーティング・ソース・ACL・ピア到達性を確認 |
| Connect | TCP 接続が進行中。パスとリモートリスナーを確認 |
| OpenSent/OpenConfirm | TCP は動作中。ASN・認証・タイマー・ケーパビリティ・ログを確認 |
| Idle | ネイバーが無効化・設定欠落・ポリシーブロック・バックオフタイマーの可能性 |

## トランスポートチェック

```text
ping <peer> source <local-source>
traceroute <peer> source <local-source>
show ip route <peer>
show bgp neighbors <peer> | include BGP state|Last reset|Local host|Foreign host
```

ピアがループバックからソースされている場合、双方向でループバックアドレスへのルートが存在し、ネイバー設定が期待する update-source を使用していることを確認する。

診断のショートカットとして ACL やファイアウォールポリシーを無効化しないこと。まずヒットカウンタ・ログ・パス状態を読む。

## ルートポリシーチェック

```text
show bgp neighbors <peer> advertised-routes
show bgp neighbors <peer> routes
show ip prefix-list <name>
show route-map <name>
show bgp <prefix>
```

一部のプラットフォームでは `received-routes` が利用可能になる前に追加設定が必要。インシデントトリアージ中はオペレーターが変更を承認しない限りその設定を追加しない。

## AS パスとプレフィックスレビュー

```text
show bgp regexp _65001_
show bgp regexp ^65001$
show bgp <prefix>
show bgp neighbors <peer> advertised-routes | include Network|Path|<prefix>
```

AS パス正規表現は慎重に使用する。`_65001_` は AS 65001 をトークンとしてマッチする。素の `65001` は長い ASN や無関係なテキストにもマッチする可能性がある。

## パーサーパターン

```python
import re
from typing import Any

BGP_SUMMARY_RE = re.compile(
    r"^(?P<neighbor>\d{1,3}(?:\.\d{1,3}){3})\s+"
    r"(?P<version>\d+)\s+"
    r"(?P<remote_as>\d+)\s+"
    r"(?P<msg_rcvd>\d+)\s+"
    r"(?P<msg_sent>\d+)\s+"
    r"(?P<table_version>\d+)\s+"
    r"(?P<input_queue>\d+)\s+"
    r"(?P<output_queue>\d+)\s+"
    r"(?P<uptime>\S+)\s+"
    r"(?P<state_or_prefixes>\S+)$",
    re.M,
)

def parse_bgp_summary(raw: str) -> list[dict[str, Any]]:
    rows = []
    for match in BGP_SUMMARY_RE.finditer(raw):
        state_or_prefixes = match.group("state_or_prefixes")
        if state_or_prefixes.isdigit():
            state = "Established"
            prefixes_received = int(state_or_prefixes)
        else:
            state = state_or_prefixes
            prefixes_received = None
        rows.append({
            "neighbor": match.group("neighbor"),
            "remote_as": int(match.group("remote_as")),
            "state": state,
            "prefixes_received": prefixes_received,
            "uptime": match.group("uptime"),
        })
    return rows
```

構造化パーサー出力が利用できる場合は優先する。ただし BGP サマリーの書式はプラットフォームやアドレスファミリーによって異なるため、生の出力もインシデント記録と一緒に保存すること。

## 変更ウィンドウ限定の操作

以下の操作はルーティングに影響を与えるため、自動診断として提案しない:

- BGP セッションのクリア。
- ネイバーの認証・タイマー・update-source・route-map・prefix-list の変更。
- 追加の received-route ストレージの有効化。
- ファイアウォール・ACL・コントロールプレーンポリシーの緩和。

リセットが承認された場合は、プラットフォームがサポートする中で最も影響の小さいソフトまたはルートリフレッシュオプションを選び、なぜ安全かを変更記録に記載する。

## アンチパターン

- `Active` を常にリモート側がダウンしていると決めつける。
- VRF・アドレスファミリー・update-source の違いを無視する。
- トークン境界のない広い AS パス正規表現を使用する。
- 最後のリセット理由とログを読む前にピアをハードリセットする。
- `received-routes` の出力がないことをルートが届いていない証拠とする。

## 参照

- エージェント: `network-architect`
- スキル: `cisco-ios-patterns`
- スキル: `network-config-validation`
- スキル: `network-interface-health`
