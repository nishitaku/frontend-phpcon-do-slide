---
marp: true
theme: default
paginate: true
---

# Signals Deep Dive

なぜ必要な箇所だけ更新できるのか

---

# 状態管理の問題

- 再描画範囲が広い
- 依存関係が見えない

---

# Signals

```ts
const count = signal(0)
const double = computed(() => count() * 2)