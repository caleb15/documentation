---
aliases:
  - /ja/9e5-69f-a68
  - /ja/security_monitoring/default_rules/9e5-69f-a68
  - /ja/security_monitoring/default_rules/nginx-ingress-controller-scanner-detected
disable_edit: true
integration_id: nginx-ingress-controller
kind: documentation
rule_category:
  - ログの検出
scope: nginx-ingress-controller
security: attack
source: nginx-ingress-controller
tactic: TA0001-initial-access
technique: T1190-exploit-public-facing-application
title: セキュリティスキャナからの NGINX Ingress Controller HTTP リクエスト
type: security_rules
---
### 目標
ウェブアプリケーションがスキャンされていることを検出します。これは、システムへの攻撃を隠そうとしない攻撃者の IP アドレスを識別します。より高度なハッカーは目立たないユーザーエージェントを使用します。

### 戦略
HTTP ヘッダーのユーザーエージェントを調べて、IP がアプリケーションをスキャンしているかどうかを判断し、`INFO` 信号を生成します。

### トリアージと対応
1. この IP がアプリケーションに認証済みのリクエストを行っているかどうかを確認します。
2. IP がアプリケーションに認証済みのリクエストを行っている場合
 * HTTP ログを調査し、ユーザーがアプリケーションを攻撃しているかどうかを判断します。

クエリの HTTP ヘッダーは [darkqusar][1] の [gist][2] からのものです

[1]: https://gist.github.com/darkquasar
[2]: https://gist.github.com/darkquasar/84fb2cec6cc1668795bd97c02302d380