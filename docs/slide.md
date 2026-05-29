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

## 自己紹介

- 西濃 拓郎（にしの たくろう）/ @nishitaku
- フリーランス
- フロントエンドエンジニア（Angular）
- 奈良 > 神戸 > 川崎 > 名古屋 > 岐阜


<img class="profile-icon" src="../images/icon_square.jpg" alt="nishitaku icon">


<!-- 
西濃拓郎といいます。
nishitakuのHNと、このメガネのアイコンでやらせてもらってます。
普段はAngularを使った開発をメインに
今日は岐阜からきました。初登壇です。
-->

---
<!-- _class: lead  -->
<!-- _paginate: false -->

# Signals 知ってる人〜 :raised_hand:

<!-- 
Deep Diveのセッションなので、さすがに知らない人はいないと思うが・・・
知らない人が多かったら、Signalsをざっくり解説するつもり
 -->

---

## これから話すこと

- Signals の本質は依存関係の追跡にある
- Signals がどうやって依存関係の追跡を実現しているか

---

## Signals とは

- 状態と**依存関係**を効率的に扱うリアクティブモデル
- 各種フレームワーク、ライブラリで採用
- TC39 proposal-signals (Stage 1)

<!-- 
まずは、Signalsについてざっくり紹介。
Signals は状態と依存関係を効率的に扱う、プリミティブなリアクティブモデル。
依存関係というのがこれから何度もでてくる重要なキーワード。
フロントエンドのフレームワークでも採用されているため、知っている人も多いはず。
例えば、Angular、Vue.js、Solid.js、Preact、Svelte、Qwik。
まだstage1だが、TC39のプロポーザルがあがっている。
つまり、将来的にJavaScriptの標準APIとして実装される可能性がある。
既存のFWの実装を取り入れて進めていく方針。
-->

---

## Signalsは何ができるのか？

### 例：カウンターと偶奇判定

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
これはtc39のプロポーザルでも紹介されているサンプルコードです。
counter > isEven > parity > render > DOM という依存関係がある。
重要なのはruntimeがこの依存関係を知らないということ。
つまり、counterが変更されたら、render()を呼ぶ必要がある、と開発者が知っておく必要がある。
 -->

---

### 依存関係を知っておく必要がある

```js
let counter = 0;
const setCounter = (value) => {
  counter = value;
  renderParity();
  renderColor(); // colorも変更しないとだめ
}

const parity = () => (counter % 2) == 0 ? "even" : "odd";
const renderParity = () => element.innerText = parity();

const color = () => (counter % 2) == 0 ? "blue" : "red";
const renderColor = () => counterEl.innerText = color();

setInterval(() => setCounter(counter + 1), 1000);
```

<!-- 
もしcounterに依存しているUIが増えた場合、counterが変更されたときに、そのrenderを呼び出す必要がある。
これは見るからに大変で、counterに依存する状態が増えたびに、setCounterが大きくなる。
さらに、実際のプロダクションコードでは、もっと状態が複雑に絡み合うため、破綻することは目に見えている。
また、別の問題として、parityは偶数であれば"even"となるため、例えばcounterが2 > 4 となっても再計算され、無駄なrenderがはしってしまう。
-->

---

### Signals の場合

- **State**: 手動で設定される値
- **Computed**: Stateに依存して計算される値
- **Effect**: StateやComputedに依存して実行されるコールバック

```js
const counter = new Signal.State(0);

const parity = new Signal.Computed(() => (counter() % 2) == 0 ? "even" : "odd");

effect(() => {
  element.innerText = parity.get();
});

setInterval(() => counter.set(counter.get() + 1), 1000);
```

<!-- 
Signalsの基本概念として、State、Computed、Effectの3要素がある。
基本となる状態を表すState、Stateに依存して計算されるComputed。
Effectは、StateやComputedの変化を検知して実行されるコールバック。

先ほどまでのカウンターの例をSignalsで書き換えるとこうなる。
まずcounterはsignalというStateで定義する。
parityはcounterに依存するComputed。
effectではparityに依存してレンダリングが行われる。

この仕組みであれば先ほどのようにcounterに依存するUIが増えた場合も、Computedとeffectを追加するだけで済む。さらにSignalsはメモ化もしてくれるため、偶奇が変わらない場合はparityが変更されず、無駄なレンダリングがはしらない。

-->

---

## Signalsは何ができるのか？

### 例：宣言的UI

```js
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
次に、フロントエンド開発でおなじみの宣言的UIの例を考える。
このように、nameに依存するvalidatedNameとage依存するisEvenを、HTMLにバインドしている場合。
 -->

---

### 依存関係がわからない場合

```js
const name = "太郎";
const age = 21; // 20から変更された場合

const validatedName = () => `${f(name)}`;
const isEven = () => `${age} % 2 === 0 ? '偶数' : '奇数'`; // 再計算したい
```

```html
<p>名前は{{ validatedName }}です</p>
<p>年齢は{{ isEven }}です。</p>
```

- `isEven`だけではなく`validateName`も再計算する必要がある

<!-- 
ageが変更された場合を考える。
validatedNameは依存しているのがnameだけなので、再計算したくない。
一方で、isEvenはageに依存しているため、再計算が必要。

しかし、HTML側は何が変更されたかわからないため、validatedNameとisEvenの両方の計算をする必要がある。
もしvalidatedNameで呼び出されている計算処理が重たい処理だった場合、無駄が大きい。

補足：
全部更新(Component-based reactivity)の代表がReact。
富豪的プログラミング。コンポーネント全体を再評価して、仮想DOMとの差分比較でどこを更新するか決める。無駄は多いが、React CompilerやTransitionでケアしてる
 -->

---

### Signals の場合

```js
const name = signal("太郎");
const age = signal(21); // 変更された場合

const validatedName = computed(() => `${f(name)}`);
const isEven = computed(() => `${age} % 2 === 0 ? '偶数' : '奇数'`);
```

```html
<p>名前は{{ validatedName() }}です</p>
<p>年齢は{{ isEven() }}です。</p>
```

- `isEven`だけ再計算される(**自動依存関係トラッキング**)
- 呼ばれてる場合だけ計算される(**遅延評価**)
- 同じ値なら再評価しない(**メモ化**)

<!-- 
angular signalsで書き換えるとこうなる。
変更されたageに依存するisEvenだけが再計算され、htmlの表示が更新される。
さらに
-->

---
<!-- _class: lead -->
<!-- _paginate: false -->

# Push型 or Pull型 ?

<!-- 
データの通信方式にはPush型とPull型があります。
Signalsはどちらになるのでしょうか？
 -->

--- 

## Push型<span class="reset">：状態変更時に、依存先へ即座に通知・再計算する方式</span>
例：Pub/Sub、WebSocket、プッシュ通知
- 常に最新状態
- 無駄な再計算が発生しやすい

## Pull型<span class="reset">：値が必要になった時に取得して計算する方式</span>

例：ポーリング、関数呼び出し
- 必要なときに取得して計算
- 変更されていなくても取得してしまう


<!-- 
まずは、それぞれの方式の例と、特徴を。
Push型は状態変更時に、依存先へ即座に通知、再計算する方式。つまり変更時に頑張る方式。
ネットワークの文脈だとPub/Subが代表例です。あとはイベントや、コールバックなど。
リアルタイム同期に使われる方式で、常に最新状態を維持できるが、通知された値がそのタイミングで必要かどうかを考えずに通知するため、無駄な再計算が発生しやすい

一方で、Pull型は値が必要になった時に取得して計算する方式。つまり値を読む時に頑張る方式。
ネットワークの文脈だとポーリングになります。あとは通常の関数呼び出しなど。
値が変更されていなくても取得しにいく必要がある、という無駄がある。

どちらの方式も一長一短。場合によって使い分ける必要がある。
ではSignalsはどちらか。

-->

---

## Signals は Push-Pull ハイブリッド方式

- Computedの評価は**Pull型**
- Stateの変更は即座に通知される**Push型**

<!-- 
Computedの評価はPull型。元となるStateの値がずっと前に変更されていても、getされたタイミングで取得し、再計算する。一方で、Stateの変更は即座に通知されるPush型。したがって、SignalsはPush-Pullのハイブリット型といえる。
 -->

---
<!-- _class: lead -->
<!-- _paginate: false -->

# Signals はどうやって依存関係を解決しているのか？

<!-- 
このDeep Diveセッションの本題。
 -->

---

## 例:カウンターと偶奇判定

```js
const counter = signal(0);

const parity = computed(() => (counter() % 2) == 0 ? "even" : "odd");

effect(() => {
  element.innerText = parity();
});
```

### 1. 依存関係の登録
### 2. Push
### 3. Pull
---

### 依存関係の登録

<pre class="mermaid">
sequenceDiagram
    participant C as computed(parity)
    participant R as runtime
    participant S as signal(counter)

    C->>R: activeConsumer = parity

    C->>S: count()

    S->>R: producerAccessed(count)

    R->>S: subscribers.add(parity)
    R->>C: dependencies.add(count)

    C->>R: activeConsumer = null
</pre>

<!-- 
dependency tracking phase
countのsignalをparityのcomputedが呼び出すとき。
本当はもう一つ左にeffectがいるが、複雑になるので割愛。
countとparityはruntime経由で、activeConsumerという変数を共有している。
computedは実行時にactiveConsumerに自身を登録した上で、count()をreadする。
呼び出されたcount側は、activeConsumerを読んで、登録されているconsumerを、subscribers、つまり自身を呼び出しているconsumer一覧に登録する。


ちなみにcomputedはnestするので、実際のactiveConsumerはstackになっている。
-->

---

### Push

<pre class="mermaid">
sequenceDiagram
    participant E as effect
    participant C as computed(parity)
    participant S as signal(counter)
    participant App

    App->>S: counter.set(1)

    S->>C: mark dirty
    C->>C: dirty = true

    C->>E: mark dirty
    E->>E: dirty = true
</pre>

<!-- 
push phase
アプリケーションがcounterを変更したケースを考える。
signalsは変更検知だけを伝播する。再計算はまだしない。
 -->
---

### Pull

<pre class="mermaid">
sequenceDiagram
    participant E as effect
    participant C as computed(parity)

    E->>C: parity()

    alt dirty
        C->>C: recompute
    end

    C-->>E: value
</pre>

<!-- 
pull phase
effectが実行され、computed parityが呼ばれたときdirtyの値をみる。
dirtyじゃなかったら、キャッシュしていた値を返し、dirtyだったら再計算して返す。
つまり、signalsはlazy incremental computation

 -->

---

## まとめ

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

<!-- _class: lead invert -->
<!-- _paginate: false -->
# ご清聴ありがとうございました

nishitaku

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

# signals のデメリット

## グリッチフリー vs lossy

Q: 「グリッチフリー（不整合のない）」実行とはどういう意味ですか？

A: 初期のプッシュ型リアクティブモデルでは、State が変更されるたびに Computed が即座に走り、UI を更新しようとしていました。しかし、次のフレームまでに複数の変更がある場合、中間の不正確な値が表示される「グリッチ」が発生することがありました。Signal はプル型を採用することで、フレームワークが UI を描画するタイミングで必要な更新だけを取りに行くため、無駄な計算や DOM 操作、そして表示の不整合を避けることができます。

Q: 「損失がある（lossy）」とはどういう意味ですか？

A: これはグリッチフリーの裏返しです。Signal はデータの「セル」であり、現在の最新値を表します。時間の経過に伴う「ストリーム」ではありません。State に2回連続で書き込んだ場合、1回目の値は Computed や Effect に観測されることなく「失われ」ます。これはバグではなく、ストリームが必要な場合は Async Iterable や Observable を使うべきという設計上の意図です。


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
