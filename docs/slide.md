---
marp: true
theme: my-theme
paginate: true
size: 16:9
---

<!-- _class: lead invert -->
<!-- _paginate: false -->
# Signals Deep Dive

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

- 西濃 拓郎（にしの たくろう）/ @nishitaku
- フリーランス / フロントエンドエンジニア（Angular）
- 岐阜から来ました


<img class="profile-icon" src="../images/icon_square.jpg" alt="nishitaku icon">


<!-- 
西濃拓郎といいます。
nishitakuのHNと、このメガネのアイコンでやらせてもらってます。
普段はAngularを使った開発をメインに
今日は岐阜からきました。
-->

---

## 話すこと

- Signals の本質は依存関係の追跡にある
- Signals がどうやって依存関係の追跡を実現しているか

## 話さないこと

- Signals の API を網羅的には紹介しません
- 各種FWの実装について

---

# Signals とは

- 状態と依存関係を扱うリアクティブモデル
- fine-grained reactivity を実現
- Angular / Vue / Solid / Preact / Qwik などで採用
- TC39 proposal-signals (Stage 1)

<!-- 
まずは、Signalsの概論。
Signals は単なる状態管理APIではなく、リアクティブモデル
Reactで採用されてないので知名度は低い？
-->

---

```js
let counter = 0;
const setCounter = (value) => {
  counter = value;
  renderParity();
}

const parity = () => (counter % 2) == 0 ? "even" : "odd";
const renderParity = () => element.innerText = parity();

setInterval(() => setCounter(counter + 1), 1000);
```

<!-- 
これは後述するtc39のプロポーザルで紹介されているサンプルコードです。
counter > isEven > parity > render > DOM という依存関係がある。
重要なのはruntimeがこの依存関係を知らないということ。
つまり、counterが変更されたら、render()を呼ぶ必要がある、と開発者が知っておく必要がある。
 -->

---

```js
let counter = 0;
const setCounter = (value) => {
  counter = value;
  renderParity();
  
  // counterが変更された時に、colorも変更されることを知っている必要がある
  renderColor();
}

const parity = () => (counter % 2) == 0 ? "even" : "odd";
const renderParity = () => element.innerText = parity();

const color = () => (counter % 2) == 0 ? "blue" : "red";
const renderColor = () => counterEl.innerText = color();

setInterval(() => setCounter(counter + 1), 1000);
```

<!-- 
もしUIが増えた場合、counterに依存している場合、counterが変更されたときに、そのrenderを呼び出す必要がある。
これは見るからに大変で、counterに依存する状態が増えたびに、setCounterが大きくなる。
さらに、実際のプロダクションコードでは、もっと状態が複雑に絡み合うため、破綻することは目に見えている。
また、別の問題として、parityは偶数であれば"even"となるため、例えばcounterが2 > 4 となっても再計算され、無駄なrenderがはしってしまう。
-->

---

```js
const counter = signal(0);

const parity = computed(() => (counter() % 2) == 0 ? "even" : "odd");

effect(() => {
  element.innerText = parity();
});

setInterval(() => counter.set(counter() + 1), 1000);
```

<!-- 
Angular Signalsで書き換えた場合こうなる。
parityはcounterにのみ依存し、counterが変更されたら、parityも変更される。
effectでは副作用を扱うことができて、parityが変更された場合にのみ、レンダリングすることができる。
-->

---


```jsx
const name = "太郎";
const age = 20;

const validatedName = () => `${f(name)}`;
const isEven = () => `${age} % 2 === 0 ? '偶数' : '奇数'`;
```

```html
<p>名前は{{ validatedName }}です</p>
<p>年齢は{{ isEven }}です。</p>
```

<!-- 
フロントエンド開発でおなじみの宣言的UIでも同じことがいえます。
例えば、このように、nameとageに基づいて計算した値を、テンプレートにバインドしている場合。
 -->

---

```jsx
const name = "太郎";
const age = 21; // 変更された場合

const validatedName = () => `${f(name)}`;
const isEven = () => `${age} % 2 === 0 ? '偶数' : '奇数'`; // 再計算したい
```

```html
<p>名前は{{ validatedName }}です</p>
<p>年齢は{{ isEven }}です。</p> // 更新したい
```

<!-- 
ageが変更された場合を考える。
validatedNameは依存しているのがnameだけなので、再計算したくない。
一方で、isEvenはageに依存しているため、再計算が必要。

しかし、テンプレート側は何が変更されたかわからないため、validatedNameとisEvenの両方の計算をする必要がある。
もしvalidatedNameで呼び出されている処理が重たい処理だった場合、無駄が大きい。

補足：
全部更新(Component-based reactivity)の代表がReact。
富豪的プログラミング。コンポーネント全体を再評価して、仮想DOMとの差分比較でどこを更新するか決める。無駄は多いが、React CompilerやTransitionでケアしてる
 -->

---

```jsx
const name = signal("太郎");
const age = signal(21);

const validatedName = computed(() => `${f(name)}`);
const isEven = computed(() => `${age} % 2 === 0 ? '偶数' : '奇数'`);
```

```
<p>名前は{{ validatedName() }}です</p>
<p>年齢は{{ isEven() }}です。</p>
```

<!-- 
angular signalsで書き換えるとこうなる。
テンプレートから参照しているisEvenが変更された場合のみ、再レンダリングされるため、無駄がない
-->

---

# Signals のメリット

- 状態間の依存関係を意識する必要がなくなる
- 依存している計算だけが行われるため、パフォーマンスが上がる

<!-- 
ここまでを整理すると、Signalsの本質は状態間の依存関係を解決してくれるところにあります。
-->

---

# Runtime Dependency Tracking

- ビルド時ではなく、実行時に依存を解決している

```ts
const value = computed(() => {
  if (flag()) {
    return count()
  }

  return another()
})
```

<!-- 
Signalsは誰が誰に依存しているかをruntimeで記録している。
valueがcountに依存しているのか、anotherに依存しているのかは、flagの値によって動的に変わる。
つまり、実行しないとわからない。

Svelteもv3/v4はstatic dependency trackingだった
https://medium.com/%40vasanthancomrads/svelte-compiler-internals-for-performance-8fda982ea932

けどfunction callのdependencyが追跡しづらい問題がある。
例えば、double = getCount()とかだと、getCount()が何に依存しているか完全にはわからない。
つまり、dependency が coarse-grained になりやすい

Svelte5でruntimeよりになった。

runtimeで動的にdependency trackingすることで、fine-grained reactivityを実現している。
 
-->

---

# static vs runtime

## static tracking

- compile-time
- 速い
- runtime軽い
- dynamic dependency苦手

## runtime tracking

- 実行時tracking
- dynamic dependency OK
- fine-grained
- runtime graph必要

---

# Signals は observer pattern？

- 広義のobserver pattern を含んでいる
- しかし本質は dependency graph
- Signalsは「読むことで依存登録」が重要
- computed による派生状態を扱える
- Push/Pull Hybrid によって lazy に再計算する

<!-- 
広義のObserverパターンといえる。
しかし、本質はdependency graph
observerパターンはsubscribeして、observerがsubjectに依存を手動で登録する必要がある。
一方で、signalsはgetする際に、自動で依存が登録される=auto tracking
さらに、observerパターンは、通知された時に必ず処理が実行されるが、
signalsはdirtyのみ通知されて、必要な場合にのみ再計算される
 -->


---

# Push-based or Pull-based ?

<!-- 
データの通信方式にはPush型とPull型があります。
Signalsはどちらになるのでしょうか？
 -->

--- 
# Push-based

<style scoped>
.profile-icon {
  position: absolute;
  top: 50%;
  right: 120px;
  width: 280px;
  height: 280px;
  border-radius: 50%;
  object-fit: cover;
  transform: translateY(-50%);
}
</style>

状態変更時に、依存先へ即座に通知・再計算する方式。

例：Observer Pattern、Pub/Sub、EventEmitter、RxJS


<div class="columns">
  <div class="column">
  メリット

* 常に最新状態
* 読み取り時は速い
* モデルが分かりやすい
  </div>
  <div class="column">
  デメリット

* 使われない値まで再計算
* update storm が起きやすい
* dependency graph が大きいと propagation コストが高い
</div>
</div>



---

# signals のデメリット

## グリッチフリー vs lossy

Q: 「グリッチフリー（不整合のない）」実行とはどういう意味ですか？

A: 初期のプッシュ型リアクティブモデルでは、State が変更されるたびに Computed が即座に走り、UI を更新しようとしていました。しかし、次のフレームまでに複数の変更がある場合、中間の不正確な値が表示される「グリッチ」が発生することがありました。Signal はプル型を採用することで、フレームワークが UI を描画するタイミングで必要な更新だけを取りに行くため、無駄な計算や DOM 操作、そして表示の不整合を避けることができます。

Q: 「損失がある（lossy）」とはどういう意味ですか？

A: これはグリッチフリーの裏返しです。Signal はデータの「セル」であり、現在の最新値を表します。時間の経過に伴う「ストリーム」ではありません。State に2回連続で書き込んだ場合、1回目の値は Computed や Effect に観測されることなく「失われ」ます。これはバグではなく、ストリームが必要な場合は Async Iterable や Observable を使うべきという設計上の意図です。

---
# Agenda

1. Signalsとは何か
2. 依存関係はどう追跡されるのか
3. なぜ効率的なのか
4. signal-polyfillを触って見えたこと
5. まとめ

---

# 私とSignalsとの出会い

- Angular と zone.js
- v16 で 導入された Signals によって環境が激変
- Signals の魔法に感動して、仕組みが知りたくなった


<!-- 
本題に入る前に、私とSignalsとの出会いを話しておきたい。
私はAngularを使っています。
Signalsを知らない人はぜひ使ってみて欲しい
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
