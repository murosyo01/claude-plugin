---
name: network-config-validation
description: このスキルは、ユーザーが「設定を変更前にレビューして」「このコンフィグに危険なコマンドがないか確認して」「ACL や route-map の参照が壊れていないか調べて」「VTY のセキュリティ設定を確認して」「ネットワーク自動化の事前チェックをして」と尋ねたときに使用する。ルーター・スイッチの設定をデプロイ前に検証し、危険コマンド・重複アドレス・サブネット重複・古い参照・管理プレーンリスクを検出する。
argument-hint: "[レビュー対象の設定ファイル・スニペット]"
allowed-tools: ["Read", "Grep", "Glob"]
---

# ネットワーク設定検証スキル

変更ウィンドウまたは自動化ランが本番機器に触れる前にネットワーク設定をレビューするときに使用する。

## 使用するタイミング

- Cisco IOS または IOS-XE スタイルのスニペットをデプロイ前にレビューする。
- スクリプトやテンプレートから生成された設定を監査する。
- 危険コマンド・重複 IP アドレス・サブネット重複を探す。
- ACL・route-map・prefix-list・line ポリシーが参照されているが定義されていないか確認する。
- ネットワーク自動化向けの軽量プレフライトスクリプトを構築する。

## 動作原理

設定検証は完全なパーサーではなく、積み重ねられたエビデンスとして扱う。正規表現チェックはプレフライトの警告として有用だが、最終承認にはネットワークエンジニアが意図・プラットフォーム構文・ロールバック手順をレビューする必要がある。

以下の順序で検証する:

1. 破壊的コマンド。
2. 認証情報と管理プレーンの露出。
3. 重複アドレスとサブネット重複。
4. ACL・route-map・prefix-list・インターフェースへの古い参照。
5. NTP・タイムスタンプ・リモートログ・バナーなどの運用衛生。

## 危険コマンド検出

```python
import re

DANGEROUS_PATTERNS: list[tuple[re.Pattern[str], str]] = [
    (re.compile(r"\breload\b", re.I), "reload はダウンタイムを引き起こす"),
    (re.compile(r"\berase\s+(startup|nvram|flash)", re.I), "永続ストレージを消去する"),
    (re.compile(r"\bformat\b", re.I), "デバイスファイルシステムをフォーマットする"),
    (re.compile(r"\bno\s+router\s+(bgp|ospf|eigrp)\b", re.I), "ルーティングプロセスを削除する"),
    (re.compile(r"\bno\s+interface\s+\S+", re.I), "インターフェース設定を削除する"),
    (re.compile(r"\baaa\s+new-model\b", re.I), "認証動作を変更する"),
    (re.compile(r"\bcrypto\s+key\s+(zeroize|generate)\b", re.I), "デバイス SSH キーを変更する"),
]

def find_dangerous_commands(lines: list[str]) -> list[dict[str, str | int]]:
    findings = []
    for line_number, line in enumerate(lines, start=1):
        stripped = line.strip()
        for pattern, reason in DANGEROUS_PATTERNS:
            if pattern.search(stripped):
                findings.append({
                    "line": line_number,
                    "command": stripped,
                    "reason": reason,
                })
    return findings
```

## 重複 IP とサブネット重複

```python
import ipaddress
import re
from collections import Counter

IP_ADDRESS_RE = re.compile(
    r"^\s*ip address\s+"
    r"(?P<ip>\d{1,3}(?:\.\d{1,3}){3})\s+"
    r"(?P<mask>\d{1,3}(?:\.\d{1,3}){3})\b",
    re.I | re.M,
)

def extract_interfaces(config: str) -> list[dict[str, str]]:
    results = []
    current = None
    for line in config.splitlines():
        if line.startswith("interface "):
            current = line.split(maxsplit=1)[1]
            continue
        match = IP_ADDRESS_RE.match(line)
        if current and match:
            ip = match.group("ip")
            mask = match.group("mask")
            network = ipaddress.ip_interface(f"{ip}/{mask}").network
            results.append({"interface": current, "ip": ip, "network": str(network)})
    return results

def find_duplicate_ips(config: str) -> list[str]:
    ips = [entry["ip"] for entry in extract_interfaces(config)]
    counts = Counter(ips)
    return sorted(ip for ip, count in counts.items() if count > 1)

def find_subnet_overlaps(config: str) -> list[tuple[str, str]]:
    networks = [ipaddress.ip_network(entry["network"]) for entry in extract_interfaces(config)]
    overlaps = []
    for index, left in enumerate(networks):
        for right in networks[index + 1:]:
            if left.overlaps(right):
                overlaps.append((str(left), str(right)))
    return overlaps
```

## 管理プレーンチェック

関係のない行にチェックがはみ出さないよう、VTY ブロックをセクション単位でパースする。

```python
import re

def iter_blocks(config: str, starts_with: str) -> list[str]:
    blocks = []
    current: list[str] = []
    for line in config.splitlines():
        if line.startswith(starts_with):
            if current:
                blocks.append("\n".join(current))
            current = [line]
            continue
        if current:
            if line and not line.startswith(" "):
                blocks.append("\n".join(current))
                current = []
            else:
                current.append(line)
    if current:
        blocks.append("\n".join(current))
    return blocks

def check_vty_blocks(config: str) -> list[str]:
    issues = []
    for block in iter_blocks(config, "line vty"):
        if re.search(r"transport\s+input\s+.*telnet", block, re.I):
            issues.append("VTY が Telnet を許可している。SSH のみを要求すること。")
        if not re.search(r"\baccess-class\s+\S+\s+in\b", block, re.I):
            issues.append("VTY ブロックにインバウンド access-class による送信元制限がない。")
        if not re.search(r"\bexec-timeout\s+\d+\s+\d+\b", block, re.I):
            issues.append("VTY ブロックに明示的な exec-timeout がない。")
    return issues
```

## セキュリティ衛生チェック

```python
SECURITY_PATTERNS = [
    (re.compile(r"\bsnmp-server community\s+(public|private)\b", re.I),
     "デフォルト SNMP コミュニティ文字列が設定されている"),
    (re.compile(r"\bsnmp-server community\s+\S+", re.I),
     "SNMPv2 コミュニティ文字列が設定されている。SNMPv3 authPriv を推奨"),
    (re.compile(r"\bip ssh version 1\b", re.I),
     "SSH バージョン 1 が有効になっている"),
    (re.compile(r"\benable password\b", re.I),
     "enable password が存在する。enable secret を使用すること"),
    (re.compile(r"\busername\s+\S+\s+password\b", re.I),
     "ローカルユーザーが secret ではなく password を使用している"),
]

BEST_PRACTICE_PATTERNS = [
    (re.compile(r"\bntp server\b", re.I), "NTP サーバー"),
    (re.compile(r"\bservice timestamps\b", re.I), "ログタイムスタンプ"),
    (re.compile(r"\blogging\s+\S+", re.I), "ログ送信先またはバッファ"),
    (re.compile(r"\bsnmp-server group\s+\S+\s+v3\s+priv\b", re.I), "SNMPv3 authPriv グループ"),
    (re.compile(r"\bbanner\s+(login|motd)\b", re.I), "ログインバナー"),
]

def check_security(config: str) -> list[str]:
    return [message for pattern, message in SECURITY_PATTERNS if pattern.search(config)]

def check_missing_hygiene(config: str) -> list[str]:
    return [
        f"{description} が設定されていない"
        for pattern, description in BEST_PRACTICE_PATTERNS
        if not pattern.search(config)
    ]
```

## 適用例

### 変更ウィンドウのプレフライト

1. ペーストする予定の正確なスニペットに対して危険コマンドチェックを実行する。
2. 候補設定全体に対して重複 IP とサブネット重複チェックを実行する。
3. 参照されている ACL・route-map・prefix-list がすべて存在することを確認する。
4. 管理プレーンの変更を行う前に、ロールバックコマンドとアウトオブバンドアクセスを確認する。

### 自動化プレフライト

Netmiko・NAPALM・Ansible・ベンダー API 自動化が生成された設定をプッシュする前に、ブロッキングゲートとして検証を使用する。危険コマンドと認証情報は fail-close とする。変更スコープ外のベストプラクティスの欠如は警告扱いとする。

## アンチパターン

- 正規表現検証をデバイスパーサーとして扱う。
- ドライランの差分なしに生成された設定を適用する。
- 監視要件として SNMPv2 コミュニティ文字列を推奨する。
- 関係のないセクションにまたがる可能性がある正規表現で VTY ブロックをチェックする。
- ACL を無効化してファイアウォール動作をテストする（カウンタ・ログを読むこと）。

## 参照

- エージェント: `network-architect`
- スキル: `network-interface-health`
- スキル: `cisco-ios-patterns`
- スキル: `netmiko-ssh-automation`
