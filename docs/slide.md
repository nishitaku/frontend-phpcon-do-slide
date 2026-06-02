---
marp: true
theme: my-theme
html: true
paginate: true
size: 16:9
---

<script type="module">
import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
mermaid.initialize({
  startOnLoad: true,
});
</script>

<!-- _class: lead invert -->
<!-- _paginate: false -->
# Signals Deep Dive

フロントエンド・PHPカンファレンス北海道 2026

nishitaku

---
<style scoped>
section h1 {
  margin-bottom: 70px;
}
</style>

# 初めまして

<div class="profile">
<img class="profile-icon" src="../images/icon_square.jpg" alt="nishitaku icon">

<div>

#### 西濃 拓郎（にしの たくろう）

#### [@nishitaku](https://x.com/nishitaku_dev)

#### フリーランスエンジニア（10年目）

<img class="angular" src="../images/angular_wordmark_gradient.png" alt="nishitaku icon">
</div>

</div>


<!-- 
初めまして、西濃拓郎といいます。。
nishitakuのハンドルネームでフリーランスのエンジニアとして活動しています。今年で10年目になりました。。
普段はAngularをつかったフロントエンドの開発を仕事にしています。
-->

---
<style scoped>
section h2 {
  margin-bottom: 100px;
  margin-left: 40px
}
</style>

# これから話すこと

## Signals がどういう仕組みで動いているか

# 持ち帰ってもらいたいこと

## Signals の内部実装を理解して<br>リアクティブシステムの勘所を掴む

<!-- 
本トークはDeep Diveです。
Deep Diveの価値は、ブラックボックスを減らして、なぜそう動くのかを理解することで、不具合調査やパフォーマンス改善の精度をあげることにあると思っています。
これから私はsignalsがどういう仕組みで動いているか、どうやって効率的に必要な箇所だけを更新しているのかを解説します。
signalsは各種フロントエンドFWも採用されていて、モダンフロントエンドのリアクティブシステムの基盤になっています。
つまり、みなさんには、signalsの内部実装を少しでも理解していただくことで、リアクティブシステムの勘所を掴んでもらいたいと思っています。
 -->

---

# 目次

### 1. Signals 概要
### 2. 特徴① 依存関係の自動追跡
### 3. 特徴② Push-Pull ハイブリッド方式
### 4. 特徴③ メモ化
### 5. どうやって依存関係を追跡しているのか

---

# Signals 概要

```ts
const counter = signal(0);
```

- 状態と**依存関係**を効率的に扱うリアクティブモデル
- 各フレームワーク(Angular/Preact/Svelte/Vue ...etc)で採用
- 基本3要素
  - **State**: 手動で設定される値
  - **Computed**: Stateに依存して計算される値
  - **Effect**: StateやComputedに依存して実行されるコールバック
- TC39 proposal-signals (Stage 1)

<!-- 
まず、知らない人もいると思うので、Signalsについてざっくり紹介します。
signalsは状態と依存関係を効率的に扱う、プリミティブなリアクティブモデルです。この依存関係というのがこれから何度もでてくる重要なキーワードになります。
フロントエンドの各種フレームワークでも採用されているので、現場で使っている人もいると思う。自分もAngularで使っています。
Signalsの基本概念として、State、Computed、Effectの3要素があります。後からでてくるコードを見た方がわかりやすいので、ここでの説明は割愛します。
また、TC39のプロポーザルがあがっている。まだstage1だが、将来的にJavaScriptの標準APIとして実装されて、各フレームークの独自実装が置き換わることを期待されています。
-->

---
<!-- _class: lead invert -->
<!-- _paginate: false -->

# Signals の特徴①

# 依存関係の自動追跡

<!-- 
まず、signalsの1つ目の特徴として依存関係の自動追跡について解説していきます。
 -->

---

## Vanilla JSの場合

```js
let counter = 0;
const setCounter = (value) => {
  counter = value;
  render();
};

const isEven = () => (counter & 1) == 0;
const parity = () => isEven() ? "even" : "odd";
const render = () => element.innerText = parity();

setInterval(() => setCounter(counter + 1), 1000);
```

#### 開発者が依存関係を知っておく必要がある<br>→ 変更時の対応漏れや過剰更新が問題になる

<!-- 
まず、VanillaJSでカウンターと偶奇判定を実装するケースを考えてみます。counterという変数があり、その値が偶数か奇数かをDOMにレンダリングしたいとします。counterが変化するたびに、最新の値を反映してDOMを更新する必要があります。
このコードにはいくつか問題がありますが、一番大きな問題は、依存関係を開発者が知っておく必要がある、ということです。状態や計算、UIが複雑になっていくと、知っておくべき依存関係が増えるため、変更時の対応漏れや過剰な更新が発生し、アプリケーションがスケールしません。

(補足)例えば、別の開発者がisEvenの値を表示したい、とします。その人はisEvenがcounterに依存していること知らないため、setCounterで、そのisEvenを表示する関数を呼び出すのを忘れてしまい、結果、counterが更新されてもisEvenの表示が更新されない、という問題が発生する
 -->

---

## Signals の場合

```js
const counter = new Signal.State(0);
const isEven = new Signal.Computed(() => (counter.get() & 1) == 0);
const parity = new Signal.Computed(() => isEven.get() ? "even" : "odd");

effect(() => {
  element.innerText = parity.get();
});

setInterval(() => counter.set(counter.get() + 1), 1000);
```

#### runtimeが依存関係を把握してくれる<br>→ 依存関係が増えてもスケールしやすい

<!-- 
先ほどの例をsignalsで書き換えるとこうなります。
まずcounterはStateで定義する。
isEvenはcounterに依存するComputed、parityはisEvenに依存するComputedになります。こうすることで、counterが変更されたときに、isEventが自動で計算され、isEvenが計算されたときにparityが自動で計算されます。
さらに、effectはparityに依存します。こうすることで、parityが変更されたときに、effect内のコールバックが呼ばれ、レンダリングされます。
このように依存関係をruntimeが把握できる仕組みにすることで、変更伝搬を自動化できるため、依存関係が増えてもスケールしやすいのがsignalの大きなメリットの１つと言えます。

-->

---

## 宣言的UIの場合

```js
// JavaScript
let name = "太郎";
let age = 20;
const validatedName = () => f(name);
const parity = () => (age % 2 === 0 ? '偶数' : '奇数');

// HTML
<p>名前は{{ validatedName() }}です</p>
<p>年齢は{{ parity() }}です。</p>
```

#### 依存関係がわからないと、`age`が変更されたときに<br>`parity`だけでなく`validatedName`も再計算する必要がある

<!-- 
次に、フロントエンド開発でおなじみの宣言的UIの場合を考えてみます。
nameとageという状態が定義されていて、nameに依存するvalidatedNameとage依存するparityがあって、それぞれHTMLにバインドします。
例えばageが変更された場合、本来であれば依存しているparityだけを再計算すればいいのですが、runtimeが依存関係をしらないと全て再計算して再描画する必要があります。もしvalidatedNameで呼び出されている関数fが重たい処理だった場合、無駄が大きいです。

補足：
全部更新(Component-based reactivity)の代表がReact。
富豪的プログラミング。コンポーネント全体を再評価して、仮想DOMとの差分比較でどこを更新するか決める。無駄は多いが、React CompilerやTransitionでケアしてる
 -->

---

## Signals の場合

```js
// JavaScript
const name = signal("太郎");
const age = signal(20);
const validatedName = computed(() => f(name()));
const parity = () => (age() % 2 === 0 ? '偶数' : '奇数');

// HTML
<p>名前は{{ validatedName() }}です</p>
<p>年齢は{{ parity() }}です。</p>
```

#### runtimeが依存関係を把握してくれる<br>→ 必要な計算だけを再実行できる＝無駄な計算を減らせる

<!-- 
先ほどの例をAngular signalsで書き換えるとこうなります。
runtimeが依存関係を把握しているため、ageが変更された場合は、ageに依存するparityだけが再計算されます。つまり無駄な計算を減らすことができます。
-->

---
<!-- _class: lead -->
<!-- _paginate: false -->

# Signals を使うことで

# 自動で依存関係を追跡できる

<!-- 
ここまでを一旦整理すると、signalsを使うことで、自動で依存関係を追跡してくれる、といえます。自動で追跡してくれるため、全体がスケールしますし、必要な計算だけを再実行できる、ということになります。

 -->

---
<!-- _class: lead invert -->
<!-- _paginate: false -->

# Signals の特徴②

# Push-Pull ハイブリッド方式

<!-- 
次に、signalsの2つ目の特徴として、Push-Pullハイブリッド方式によるデータの通信について解説していきます。
-->

--- 
<style scoped>
section ul {
  margin-top: 0px;
}
</style>

## データを誰が主導して渡すか

### Push型<span class="reset">：データを持っている側が通知する</span>

- 例）Pub/Sub、WebSocket、Observerパターン、EventEmitter
- 常に最新状態
- 無駄な再計算が発生しやすい


### Pull型<span class="reset">：データを使う側が取りにいく</span>

- 例）ポーリング、getter、関数呼び出し
- 必要なときに取得して計算
- 毎回取得コストが発生してしまう

<!-- 
Push型、Pull型というのは、さまざまな文脈で使われる概念ですが、データを誰が手動して渡すかを表す言葉として用いられます。
Push型はデータを持っている側が更新されたことを通知する、つまり変更時に頑張る方式です。Pub/SubやWebSocket、ObserverパターンやEventEmitterがPush型の代表例です。
リアルタイム同期によく使われる方式で、常に最新状態を維持できるますが、一方的に通知されるため、データを使う側で無駄な再計算が発生しやすいです。

一方で、Pull型はデータを使う側が取りにいく、つまり値を読む時に頑張る方式。ポーリングが代表例です。あとは通常の関数呼び出しもPull型になります。変更されていなくても取得するのでコストがかかります。
-->

---

## Push-Pull ハイブリッド方式

```js
const counter = new Signal.State(0);
const isEven = new Signal.Computed(() => (counter.get() & 1) == 0);
```

#### Stateの変更は即座に通知される**Push型**

#### Computedの評価は**Pull型**


<!-- 
signalsは先ほどのPush型とPull型のいいとこ取りをしたハイブリッド方式です。Stateの変更は即座に通知されるPush型です。この例だと、counterが変更されたときに、isEvenに即座に通知されます。
一方で、Computedの評価はPull型です。元となるStateの値がずっと前に変更されていても、getされたタイミングで取得し、再計算します。この例だとisEventが呼び出されたタイミングで再計算します。
 -->

---
<!-- _class: lead invert -->
<!-- _paginate: false -->

# Signals の特徴③

# メモ化（割愛）

<!-- 
次に、signalsの3つ目の特徴として、メモ化について解説していきます。
と言いたいところですが、メモ化自体は珍しくない機能なので、今回は割愛させていただきます。同じ入力であれば再計算せずにキャッシュしておいた値を返すやつです。
-->

---
<!-- _class: lead invert -->
<!-- _paginate: false -->

# Signals はどうやって
# 依存関係の追跡を実現しているのか

<!-- 
最後に、今までの特徴を踏まえて、signalsがどうやって依存関係の追跡を実現しているのか、を詳しく追いかけていきます。
-->

---

## 依存関係を追跡するための３ステップ

<div class="flow-image-container">
<img src="../images/flow.dio.svg">
</div>

<!-- 
signalsは3ステップで依存関係を追跡しています。
まず、signalsはruntime経由でStateとComputed、ComputedとEffectの間に双方向の依存グラフを構築します。
次に、Pushフェーズ。Stateが更新されたときに、変更されて古くなった、というフラグだけを伝播します。このタイミングではまだ再計算しません。
最後に、Pullフェーズ。通知されたフラグを見て、必要であれば再計算します。
 -->

---

## 具体的例で解説していく

```js
const counter = new Signal.State(0);

const parity = new Signal.Computed(() => (counter.get() % 2) == 0 ? "even" : "odd");

effect(() => {
  element.innerText = parity.get();
});
```

`counter → parity → effect`の依存

<!-- 
カウンターと偶奇判定の例で、具体的に解説していきます。
parityがcounterに依存して、effectがparityに依存している。
 -->

---

### 依存グラフの構築

<img class="code-image" src="../images/code.svg">

<pre class="mermaid">
%%{
  init: {
    "theme": "base",
    "themeVariables": {
      "background": "#fff8e1",
      "actorBkg": "#E3F2FD",
      "actorBorder": "#0288d1",
      "actorTextColor": "#455a64",
      "signalColor": "#455a64",
      "signalTextColor": "#455a64",
      "labelBoxBkgColor": "#E1F5FE",
      "labelBoxBorderColor": "#0288d1",
      "labelTextColor": "#455a64",
      "activationBorderColor": "#0288d1",
      "activationBkgColor": "#BBDEFB",
      "primaryTextColor": "#455a64",
      "lineColor": "#455a64"
    }
  }
}%%

sequenceDiagram
    participant C as Computed(parity)
    participant R as runtime
    participant S as State(counter)

    C->>R: activeConsumer = parity

    C->>S: counter.get()

    S->>R: producerAccessed(count)

    R->>S: subscribers.add(parity)
    R->>C: dependencies.add(count)

    C->>R: activeConsumer = null
</pre>

<!-- 
まず、依存グラフを構築します。
実際はもう一つ左にeffectがいて、effectとComputedの間にも依存グラフが構築されるますが、複雑になるので省略しています。
StateとComputedはruntime経由で、activeConsumerという変数を共有しています。
Computedは実行時にactiveConsumerに自身を登録した上で、countを読みにいきます。
呼び出されたState側は、activeConsumerを読んで、登録されているconsumerを、subscribers、つまり自身を呼び出しているconsumer一覧に登録します。
これによってStateとComputedの間の依存グラフが構築される。
ちなみにComputedはnestするので、実際のactiveConsumerはstack構造になっています。
-->

---

### Push

<img class="code-image" src="../images/code.svg">

<pre class="mermaid">
%%{
  init: {
    "theme": "base",
    "themeVariables": {
      "background": "#fff8e1",
      "actorBkg": "#E3F2FD",
      "actorBorder": "#0288d1",
      "actorTextColor": "#455a64",
      "signalColor": "#455a64",
      "signalTextColor": "#455a64",
      "labelBoxBkgColor": "#E1F5FE",
      "labelBoxBorderColor": "#0288d1",
      "labelTextColor": "#455a64",
      "activationBorderColor": "#0288d1",
      "activationBkgColor": "#BBDEFB",
      "primaryTextColor": "#455a64",
      "lineColor": "#455a64"
    }
  }
}%%

sequenceDiagram
    participant E as effect
    participant C as Computed(parity)
    participant S as State(counter)
    participant App

    App->>S: counter.set(1)

    S->>C: mark dirty
    C->>C: dirty = true

    C->>E: mark dirty
    E->>E: dirty = true
</pre>

<!-- 
次にPushフェーズです。。
アプリケーションがcounterを変更した場合を考えます。
Stateはdirtyフラグとして、「変更されたこと」だけを伝播する。再計算はまだしない。
-->
---

### Pull

<img class="code-image" src="../images/code.svg">

<pre class="mermaid">
%%{
  init: {
    "theme": "base",
    "themeVariables": {
      "background": "#fff8e1",
      "actorBkg": "#E3F2FD",
      "actorBorder": "#0288d1",
      "actorTextColor": "#455a64",
      "signalColor": "#455a64",
      "signalTextColor": "#455a64",
      "labelBoxBkgColor": "#E1F5FE",
      "labelBoxBorderColor": "#0288d1",
      "labelTextColor": "#455a64",
      "activationBorderColor": "#0288d1",
      "activationBkgColor": "#BBDEFB",
      "primaryTextColor": "#455a64",
      "lineColor": "#455a64"
    }
  }
}%%

sequenceDiagram
    participant E as effect
    participant C as Computed(parity)

    E->>C: parity.get()

    alt dirty
        C->>C: recompute
    end

    C-->>E: value
</pre>

<!-- 
最後にPullフェーズ。
effectが実行され、Computedが呼ばれたとき、先ほど通知されたdirtyの値をみる。
dirtyだったら再計算し、dirtyじゃなかったら、キャッシュしていた値を返します。
これにより、無駄な再計算が発生しない仕組みを実現している。
-->

---

## Signals Deep Dive のまとめ

- #### Signals は「誰が誰を読んだか」を runtime に記録する
- #### 依存グラフにより変更伝搬を自動化する
- #### Push-Pull ハイブリッド方式により必要時だけ再計算する

---

<!-- 
まとめです。
Signalsは「誰が誰を読んだか」という依存グラフを構築して、変更伝搬を自動化します。そして、その依存関係をもとに、Push-Pullハイブリッド方式で効率的に再計算する。
 -->

<!-- _class: lead invert -->
<!-- _paginate: false -->
# ご清聴ありがとうございました

nishitaku

---

# Why Signals Deep Dive？

- Angular と zone.js
- v16 で 導入された Signals によって環境が激変
- Signals の魔法に感動


<!-- 
私はAngularを使っています。
Signalsを知らない人はぜひ使ってみて欲しい
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

# グリッチフリー

```js
count = 1
count = 2
count = 3
```

- グリッチ：中間の不正確な値が表示されること
- Push型リアクティブモデルの場合、変更が即座にUIに反映される
- Signals はフレームワークが UI を描画するタイミングで必要な更新だけを取りに行くため、グリッチが発生しずらい
- 「損失がある」ことの裏返しでもある。

<!-- 
signalsはグリッチフリーという特徴がある。
グリッチとは、中間の不正確な値が表示されること。
このcountを更新する例の場合、最終的な3ではなく、途中の1や2の状態も表示されるということ。
初期のプッシュ型リアクティブモデルの場合、Stateが変更されるたびにComputedが即座にはしり、UIを更新するため、グリッチが発生することがあった。
一方で、signalsはpull型のデータ取得なので、フレームワークがUIを描画するタイミングで必要な更新だけを取りに行くため、グリッチが発生しずらい

これはメリットばかりではなく、データを損失していると捉えることもできる。
ストリームとして全てのデータが必要な場合は、Observableなどを使うべき。
 -->

---

# Signals のトレードオフ

- 「誰が誰を読むか」が実行時に決まる
- 更新の流れを追うのが難しいことがある
- 細かく分割しすぎると複雑になりやすい
- Signals を使わない方がシンプルな場合もある

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

# TC39 proposal-signals Stage 1

- フレームワーク共通のリアクティブモデルを目指している
- いくつかの課題をクリアしてから Stage 2 へ
  - 複数の production-grade polyfill
  - 複数 framework への統合検証
  - パフォーマンス検証

---

# References

## TC39 Signals

-  [tc39/proposal-signals](https://github.com/tc39/proposal-signals)
-  [signal-polyfill](https://github.com/proposal-signals/signal-polyfill)
-  [A TC39 Proposal for Signals](https://eisenbergeffect.medium.com/a-tc39-proposal-for-signals-f0bedd37a335)

## Framework Implementations

- Angular Signals
- Preact Signals
- SolidJS
- Vue Reactivity