---
name: cisco-ios-patterns
description: このスキルは、ユーザーが「Cisco IOS の設定を確認したい」「安全な show コマンドを教えて」「ワイルドカードマスクの書き方を教えて」「ACL の向き（in/out）を確認したい」「設定モードの使い方を教えて」「変更後に running-config に反映されているか確認する方法を教えて」と尋ねたときに使用する。Cisco IOS および IOS-XE の設定レビュー・show コマンド・ワイルドカードマスク・ACL 配置・インターフェース衛生・安全な変更ウィンドウ検証のパターンを提供する。
argument-hint: "[レビュー対象のIOS設定スニペットまたは確認したい操作]"
allowed-tools: ["Read", "Grep", "Glob"]
---

# Cisco IOS パターンスキル

Cisco IOS または IOS-XE のスニペットをレビューする場合、変更ウィンドウのチェックリストを作成する場合、またはインシデントを悪化させずに機器からエビデンスを収集する方法を説明する場合に使用する。

## 使用するタイミング

- 計画された変更の前に IOS または IOS-XE の設定をレビューする。
- トラブルシューティング用の読み取り専用 `show` コマンドを選択する。
- ACL のワイルドカードマスクとインターフェースの向きを確認する。
- グローバル・インターフェース・ルーティングプロセス・ライン設定モードを説明する。
- 変更が running-config に反映されており、意図的に保存されていることを確認する。

## 動作原則

IOS の例はパターンとして扱い、本番機器への貼り付け可能な変更としては扱わない。実機への変更前に、プラットフォーム・インターフェース名・現在の設定・ロールバックパス・アウトオブバンドアクセスを確認すること。

推奨するワークフロー:

1. 読み取り専用コマンドで現在の状態を取得する。
2. 候補の正確な設定をレビューする。
3. 管理アクセスがロックアウトされないことを確認する。
4. メンテナンスウィンドウで最小限の変更を適用する。
5. 状態を再読し、ベースラインと比較してから、検証後にのみ保存する。

## モードリファレンス

```text
Router> enable
Router# show running-config
Router# configure terminal
Router(config)# interface GigabitEthernet0/1
Router(config-if)# description UPLINK-TO-CORE
Router(config-if)# no shutdown
Router(config-if)# exit
Router(config)# end
Router# show running-config interface GigabitEthernet0/1
```

`running-config` はアクティブメモリ。`startup-config` はリロードを生き延びるもの。コマンドが受け付けられたからといって変更を保存しないこと。動作を検証してから、変更が承認された場合に `copy running-config startup-config` を実行する。

## 読み取り専用コレクション

```text
show version
show inventory
show processes cpu sorted
show memory statistics
show logging
show running-config | section line vty
show running-config | section interface
show running-config | section router bgp
show ip interface brief
show interfaces
show interfaces status
show vlan brief
show mac address-table
show spanning-tree
show ip route
show ip protocols
show ip access-lists
show route-map
show ip prefix-list
```

設定にシークレット・顧客名・プライベートトポロジーが含まれている可能性があるため、チケットにフル設定をダンプせず、必要なセクションのみを収集する。

## ワイルドカードマスク

IOS の ACL と多くのルーティングステートメントはサブネットマスクではなくワイルドカードマスクを使用する。

```text
サブネットマスク       ワイルドカードマスク
255.255.255.255   0.0.0.0
255.255.255.252   0.0.0.3
255.255.255.0     0.0.0.255
255.255.0.0       0.0.255.255
```

デプロイ前にワイルドカードマスクをレビューする。サブネットマスクを誤ってワイルドカードとして使用すると、意図より大幅に多いトラフィックにマッチする可能性がある。

```text
ip access-list extended WEB-IN
  10 permit tcp 192.0.2.0 0.0.0.255 any eq 443
  999 deny ip any any log
```

すべての ACL には暗黙の deny がある末尾にある。ミスを観察することが運用目標に含まれる場合は明示的なログ付き deny を追加し、ログ量が安全かを確認する。

## ACL 配置レビュー

インターフェースに ACL を適用する前に以下の質問に答える:

- フィルタするトラフィックの向きは `in` か `out` か?
- 管理トラフィックは既知のジャンプホストまたは管理サブネットからソースされているか?
- 必要なルーティング・DNS・NTP・監視・アプリケーショントラフィックへの明示的な permit があるか?
- 安全なテストソースからのヒットカウンタが取得可能か?
- ロールバックコマンドとアクティブなコンソールまたはアウトオブバンドパスがあるか?

ファイアウォールや ACL の保護を削除して疎通をテストしないこと。カウンタ・ログ・ルート状態を先に読む。

## インターフェース衛生

```text
interface GigabitEthernet0/1
 description UPLINK-TO-CORE
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 999
 no shutdown
```

明確な description・明示的な switchport mode・文書化された native VLAN を使用する。ルーテッドインターフェースでは、リンク状態が転送の正しさを意味すると決めつける前に、マスク・ピアアドレッシング・ルーティングプロセスを確認する。

## 変更ウィンドウ検証

実際の変更に対応した前後チェックを使用する。

```text
show running-config | section interface GigabitEthernet0/1
show interfaces GigabitEthernet0/1
show logging | include GigabitEthernet0/1|changed state|line protocol
show ip route <prefix>
show ip access-lists <name>
```

ルーティング変更では変更前後にネイバー状態とルートテーブルも取得する。ACL 変更では汎用 ping ではなく、計画されたテストソースからのヒットカウンタを比較する。

## アンチパターン

- デバイス固有の差分なしに生成された設定を適用する。
- 変更後チェックが通る前に設定を保存する。
- IOS がワイルドカードマスクを期待する箇所にサブネットマスクを使用する。
- ACL を誤ったインターフェースの向きに適用する。
- ACL・ルートポリシー・認証を無効化してトラブルシューティングする。
- シークレットとトポロジーをサニタイズせずにフル設定をパブリックツールに貼り付ける。

## 参照

- エージェント: `network-architect`
- スキル: `network-config-validation`
- スキル: `network-interface-health`
- スキル: `network-bgp-diagnostics`
