---
title: "Cloudflareでads.txt を成立させる手順"
date: 2026-01-28T00:00:00+09:00
draft: false
slug: "adsense-ads-txt-apex-cloudflare"
tags: ["Cloudflare", "AdSense", "DNS"]
description: "arinkolab.com/ads.txt がパスを失ってトップに飛ぶ問題を、Cloudflare Redirect Rules でパス保持して解決する手順メモ。"
---

## 背景
Google AdSense の審査では `ads.txt` を正しく取得できることができなかった。

今回の構成は次の通りです。

- 審査対象のサイト：`arinkolab.com`（Cloudflare へ NS 移管済み）
- アプリ本体：`sqlformatter.arinkolab.com`（Cloudflare Pages / カスタムドメイン / SSL 有効）
- 目的：`arinkolab.com` → `sqlformatter.arinkolab.com` にリダイレクトしつつ、`/ads.txt` は **パスを保持して** `sqlformatter.arinkolab.com/ads.txt` に到達させる

問題は、以下の状態になっていたことです。

- `https://sqlformatter.arinkolab.com/ads.txt` は表示できる（OK）
- `https://arinkolab.com/ads.txt` が `https://sqlformatter.arinkolab.com/`（トップ）へ飛び、`/ads.txt` が失われる（NG）

---

## 解決方針
優先する方針は **Redirect Rules でパスを保持する**ことです。

- 目標：`https://arinkolab.com/*` → `https://sqlformatter.arinkolab.com/$1`
- クエリ（`?a=b` 等）も保持する

もし Redirect Rules で実現できない場合は、代替として Worker で `arinkolab.com/ads.txt` を直接返す案もありますが、まずは Redirect Rules で成立させるのが運用上シンプルです。

---

## 前提（DNS / SSL）
- Cloudflare 側で `arinkolab.com` がプロキシ（オレンジ雲）になっていること
- `sqlformatter.arinkolab.com` は Cloudflare Pages のカスタムドメインとして有効で、HTTPS が有効であること

また、HTTP で来たアクセスを HTTPS に統一するために、Cloudflare の設定で以下を ON にします。

- **SSL/TLS → エッジ証明書 → Always Use HTTPS：ON**

---

## 手順（Cloudflare Redirect Rules）
Cloudflare ダッシュボード（日本語UI）で次を設定します。

1. 対象ゾーン：`arinkolab.com`
2. **ルール → リダイレクト ルール** を開く
3. ルールを作成し、以下を設定

### マッチ条件（ワイルドカード）
- 種別：ワイルドカード パターン
- リクエストURL：  
  `https://arinkolab.com/*`

※ `http://` も拾いたい場合、Always Use HTTPS を ON にするか、別ルールで `http://` を同様に扱います。

### アクション（ターゲットURL）
- ターゲットURL：  
  `https://sqlformatter.arinkolab.com/${1}`
- ステータスコード：`301`
- **クエリ文字列を保存する：ON**

これで、以下が成立します。

- `https://arinkolab.com/ads.txt` → `https://sqlformatter.arinkolab.com/ads.txt`
- `https://arinkolab.com/` → `https://sqlformatter.arinkolab.com/`
- `https://arinkolab.com/some/path?a=1` → `https://sqlformatter.arinkolab.com/some/path?a=1`

---

## つまづきポイント（キャッシュ）
設定を変えた直後、ブラウザのキャッシュにより「古いリダイレクト先」に飛ぶことがあります。

- シークレットモードで確認する
- 可能なら別ブラウザでも確認する

この確認を通して、`/ads.txt` が正しく保持されていることを確認しました。

---

## 確認方法
以下を直接アクセスして確認します。

- `https://arinkolab.com/ads.txt`
  - 最終的に `sqlformatter.arinkolab.com/ads.txt` の内容が表示されること
- `https://sqlformatter.arinkolab.com/ads.txt`
  - 直接アクセスでも内容が表示されること

---

## 代替案（Workerで ads.txt を直返し）
将来的に `arinkolab.com` をポータル化し、apex で 200 を返したい場合は、
`arinkolab.com/ads.txt` を Worker で直接返す構成も検討できます。

ただし当面は、Redirect Rules でパス保持してサブドメイン側の `ads.txt` を参照する方が実装が軽く、運用も簡単です。

---

## まとめ
- AdSense 用に `ads.txt` を成立させるには、apex 側でも `/ads.txt` が取得できる必要がある
- Cloudflare Redirect Rules で `https://arinkolab.com/*` → `https://sqlformatter.arinkolab.com/${1}` を設定し、パス保持で解決できた
- 反映確認はキャッシュ影響を避けるためシークレットモード推奨
