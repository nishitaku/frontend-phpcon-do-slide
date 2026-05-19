---
marp: true
theme: default
paginate: true
size: 16:9
---

# Signals Deep Dive

## なぜ必要な箇所だけ更新できるのか

フロントエンドカンファレンス北海道 2026

nishitaku
<!-- 
このスライドで伝えたいこと

このトークは「Signals API紹介」ではなく、リアクティブシステムの内部構造を見る話

話す内容

* 最近いろんなフレームワークで Signals が出てきている
* 今日は API の使い方ではなく「なぜ必要な箇所だけ更新できるのか」を見る
* Angular 固有ではなく、リアクティブシステムの共通原理の話
 -->

---

# 自己紹介

- nishitaku
- フロントエンドエンジニア
- Angular を中心に開発
- Signals / 状態管理 / Reactive System に興味

<!-- 
このスライドで伝えたいこと

状態管理や Signals を追っている人間として話している

話す内容

* Angular中心に開発している
* 状態管理や Signals に興味がある
* Angular Signals や TC39 proposal-signals を追っている
* signal-polyfill を触りながら理解していた
 -->


---

# 今日のテーマ

## Signals はなぜ
## 「必要な箇所だけ更新」
## できるのか？

<!-- 
このスライドで伝えたいこと

このトークの問いを明確にする

話す内容

* Signals の便利さは知っている人も多い
* でも「なぜ必要な箇所だけ更新できるのか」は意外とブラックボックス
* 今日は内部で何が起きているかを見る
 -->

---

# Agenda

1. Signalsとは何か
2. 依存関係はどう追跡されるのか
3. なぜ効率的なのか
4. signal-polyfillを触って見えたこと
5. まとめ

---

# Signalsとは何か

- 状態と依存関係を扱うリアクティブモデル
- fine-grained reactivity を実現
- Angular / Vue / Solid / Preact / Qwik などで採用

<!-- 
このスライドで伝えたいこと

Signals は単なる状態管理APIではなく、リアクティブモデル

話す内容

* 状態と計算の依存関係を扱う
* fine-grained reactivity を実現する
* Angular / Vue / Solid などで採用されている
* 実は各フレームワークで共通する考え方がある
 -->

---

# TC39 proposal-signals

- Stage 1 proposal
- JavaScript 標準化提案として進行中

<!-- 
このスライドで伝えたいこと

Signals はフレームワーク固有ではなく、JavaScript標準化の流れにもある

話す内容

* proposal-signals が Stage1
* JavaScriptレベルでリアクティブプリミティブを考え始めている
* Signals 的なモデルの重要性が増している
 -->

---

# signal / computed / effect

```ts
const count = signal(0)

const double = computed(() => {
  return count() * 2
})

effect(() => {
  console.log(double())
})
```

<!-- 
このスライドで伝えたいこと

Signals は「状態」「計算」「副作用」で構成される

話す内容

* signal = state
* computed = derived state
* effect = side effect
* 今日は API 説明が主目的ではない
* この後、内部で何が起きているかを見る
 -->


---

# Signalsはどう依存関係を追跡するのか

## 「読むことで依存を記録する」

<!-- 
このスライドで伝えたいこと

Signals の本質は dependency tracking

話す内容

* 一番重要なのは dependency tracking
* 「何が何に依存しているか」を記録している
* これが partial update の前提になる
 -->

---

# 実行時に依存を収集

```ts
computed(() => {
  count()
})
```

<!-- 
このスライドで伝えたいこと

依存関係は実行時に自動収集される

話す内容

* computed 実行中に signal を読む
* 「この computed は count に依存している」を登録
* 手動で依存を書く必要がない
* auto-tracking がかなり重要
 -->

---

# なぜ効率的なのか

## Push/Pull Hybrid Model

- Push → 変更通知
- Pull → 必要時再評価

<!-- 
このスライドで伝えたいこと

Signals は push-only ではない

話す内容

* 変更通知は push
* 実際の値計算は pull
* dirty にするだけで即再計算しない
* 必要になった時だけ再評価する
* これが効率性につながる
 -->

---

# Signalsは

## 「再描画」ではなく

# 「再計算の最小化」

を重視している

<!-- 
このスライドで伝えたいこと

Signals の本質は再計算範囲の最小化

話す内容

* React 的な「再render」と少し感覚が違う
* Signals は dependency graph ベース
* 必要なノードだけ再評価する
* incremental computation に近い
 -->

---

# signal-polyfillを触ってみて

- 思ったよりシンプル
- 魔法ではない
- dependency graph の伝播アルゴリズム

<!-- 
このスライドで伝えたいこと

Signals は意外とシンプルなアルゴリズム

話す内容

* 最初は魔法っぽく見えた
* 実際は dependency graph の伝播
* 内部構造を見ると理解しやすい
* フレームワークごとの差より共通原理が見えてくる
 -->

---

# まとめ

- Signalsは dependency graph を構築する
- dependency tracking によって依存関係を収集する
- Push/Pull Hybrid により効率的に更新する

---

<!-- 
このスライドで伝えたいこと

Signals の本質を再定義する

話す内容

* Signals は dependency graph を構築する
* dependency tracking によって依存関係を収集する
* Push/Pull Hybrid によって効率的に更新する
* 「必要な箇所だけ更新」の正体はこれだった
 -->

# Thank you