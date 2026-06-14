---
name: netmiko-ssh-automation
description: このスキルは、ユーザーが「Netmiko で show コマンドを複数台から収集したい」「SSH 自動化スクリプトにタイムアウトと例外処理を追加したい」「TextFSM でコマンド出力をパースしたい」「Netmiko で設定変更を安全に行うパターンを教えて」「ネットワーク自動化スクリプトをレビューして」と尋ねたときに使用する。読み取り専用収集・バッチ SSH・TextFSM パース・保護付き設定変更・タイムアウト・エラーハンドリングの安全な Netmiko パターンを提供する。
argument-hint: "[作成・レビュー対象の自動化スクリプトまたは自動化したい操作]"
allowed-tools: ["Read", "Grep", "Glob"]
---

# Netmiko SSH 自動化スキル

Netmiko でネットワーク機器に接続する Python 自動化を書く・レビューするときに使用する。デフォルトのパスは読み取り専用にする。設定変更には別途変更ウィンドウ・ピアレビュー・ロールバック計画が必要。

## 使用するタイミング

- ルーター・スイッチ・ファイアウォール全体で `show` コマンド出力を収集する。
- インターフェース・ルーティング・設定エビデンスの小規模監査スクリプトを作成する。
- ネットワーク SSH スクリプトにタイムアウトと例外処理を追加する。
- テンプレートが存在する場合に TextFSM でコマンド出力をパースする。
- 本番機器に触れる前に自動化をレビューする。

## 安全のデフォルト

- 読み取り専用の `send_command()` 収集から始める。
- インベントリは小さく明示的に保つ。アドレス範囲全体をスキャンしない。
- 環境変数・ボールト・`getpass` を使用する。認証情報をソースにハードコードしない。
- 接続タイムアウトと読み取りタイムアウトを設定する。
- 古い機器と AAA システムへの過負荷を防ぐため並行数を制限する。
- `send_config_set()` の前に明示的なオペレーターフラグを要求する。
- 変更の確認と承認が完了するまで `save_config()` を呼ばない。

## 読み取り専用接続パターン

```python
import os
from getpass import getpass
from netmiko import ConnectHandler
from netmiko.exceptions import (
    NetmikoAuthenticationException,
    NetmikoTimeoutException,
    ReadTimeout,
)

device = {
    "device_type": "cisco_ios",
    "host": "192.0.2.10",  # ドキュメント用プレースホルダー
    "username": os.environ.get("NETMIKO_USERNAME") or input("Username: "),
    "password": os.environ.get("NETMIKO_PASSWORD") or getpass("Password: "),
    "secret": os.environ.get("NETMIKO_ENABLE_SECRET"),
    "conn_timeout": 10,
    "auth_timeout": 20,
    "banner_timeout": 15,
    "read_timeout_override": 30,
}

try:
    with ConnectHandler(**device) as conn:
        if device.get("secret") and not conn.check_enable_mode():
            conn.enable()
        output = conn.send_command("show ip interface brief", read_timeout=30)
        print(output)
except NetmikoAuthenticationException:
    print("認証に失敗しました")
except NetmikoTimeoutException:
    print("SSH 接続がタイムアウトしました")
except ReadTimeout:
    print("コマンド読み取りがタイムアウトしました")
```

例ではドキュメント用アドレス範囲のプレースホルダーを使用する。実際のインベントリは gitignore 済みのローカルファイルまたはシークレット管理システムに保存する。

## バッチ収集

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Any

def collect_show(device: dict[str, Any], command: str) -> dict[str, Any]:
    host = device["host"]
    try:
        with ConnectHandler(**device) as conn:
            output = conn.send_command(command, read_timeout=45)
        return {"host": host, "ok": True, "output": output}
    except (NetmikoAuthenticationException, NetmikoTimeoutException, ReadTimeout) as exc:
        return {"host": host, "ok": False, "error": type(exc).__name__}

results = []
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(collect_show, device, "show version") for device in devices]
    for future in as_completed(futures):
        results.append(future.result())
```

機器台数と AAA システムが高い接続量を処理できることがわかっている場合を除き、`max_workers` は低く保つ。

## 構造化パース

Netmiko は TextFSM・TTP・Genie にサポートされたコマンド出力のパースを依頼できる。パーサー出力は最適化として扱い、唯一のエビデンスパスにしない。

```python
with ConnectHandler(**device) as conn:
    parsed = conn.send_command(
        "show ip interface brief",
        use_textfsm=True,
        raise_parsing_error=False,
        read_timeout=30,
    )

if isinstance(parsed, str):
    print("パーサーテンプレートがマッチしなかった。生の出力をレビュー用に保存する")
else:
    for row in parsed:
        print(row)
```

パースがブロッキング判断を行う場合は、オペレーターがミスマッチを確認できるよう生コマンド出力もパース結果と並べて保存すること。

## 保護付き設定変更パターン

```python
import os

commands = [
    "interface GigabitEthernet0/1",
    "description CHANGE-1234 UPLINK-TO-CORE",
]

# 環境変数で明示的に有効化されない限り変更を適用しない
apply_changes = os.environ.get("APPLY_NETWORK_CHANGES") == "1"

if not apply_changes:
    print("ドライランのみ。候補コマンド:")
    print("\n".join(commands))
else:
    with ConnectHandler(**device) as conn:
        conn.enable()
        # 変更前の状態を記録
        before = conn.send_command("show running-config interface GigabitEthernet0/1")
        output = conn.send_config_set(commands)
        # 変更後の状態を記録
        after = conn.send_command("show running-config interface GigabitEthernet0/1")
        print(before)
        print(output)
        print(after)
        print("startup-config への保存前に動作を確認すること。")
```

設定の保存は別途の承認ステップ。本番環境ではロールバックスニペットを用意し、変更記録に変更前後のエビデンスを残す。

## レビューチェックリスト

- スクリプトは明示的なインベントリソースを識別しているか?
- ソース・ログ・例外メッセージから認証情報が除外されているか?
- `conn_timeout`・`auth_timeout` とコマンドの `read_timeout` が設定されているか?
- 1 台の失敗でバッチ全体が止まらず、デバイスごとにエラーが報告されるか?
- 広いスキャンと無制限の並行接続を避けているか?
- 設定変更はドライランまたは明示的なオペレーターフラグの後ろに隠れているか?
- `save_config()` は初回プッシュと分離され、検証に紐付けられているか?

## アンチパターン

- パスワード・enable シークレット・秘密鍵をソースにハードコードする。
- デフォルトのコードパスで設定コマンドを送信する。
- レビュー済みインベントリではなく CIDR 範囲に対して自動化を実行する。
- サニタイズなしに running-config 全体を共有システムにログ出力する。
- パーサー成功を機器の状態が正しい証拠とする。

## 参照

- エージェント: `network-architect`
- スキル: `cisco-ios-patterns`
- スキル: `network-config-validation`
- スキル: `network-interface-health`
