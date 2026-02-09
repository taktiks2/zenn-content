---
title: "JavaScriptを安全に書くTips"
emoji: "🧷"
type: "tech"
topics:
  - "javascript"
  - "tips"
  - "safe"
published: true
published_at: "2026-01-18 14:21"
---

## はじめに

JavaScriptは柔軟で自由度の高い言語です。しかし、その自由さゆえに「同じことを実現する方法が複数存在する」という特徴があります。

この柔軟性は時にバグの温床となり、コードの可読性や保守性を損なう原因にもなります。

この記事では、より安全で堅牢なJavaScriptを書くためのTipsを紹介します。

---

## 1. `let`を避けて`const`を使う

### ⚠️ 注意が必要な書き方

```javascript
// letを使う（再代入しないのに）
let userName = 'Taro';
let maxRetry = 3;
```

### ❌ 起こりうる問題

```javascript
let total = 100;
// ... 100行後 ...
total = 'エラー';  // 意図せず型が変わってしまう
// ... さらに後 ...
console.log(total * 2);  // NaN になる
```

### ✅ こう書くとより安全

```javascript
// constを使う
const userName = 'Taro';
const maxRetry = 3;
const config = { timeout: 5000 };
```

### 💡 メリット

- **意図の明確化** — 「この値は変更されない」という意図がコードから読み取れる
- **再代入バグの防止** — 誤って値を上書きしようとするとエラーになる
- **コードレビューの効率化** — `let`を見たら「どこかで再代入される」と予測できる

---

## 2. 即時実行関数(IIFE)でスコープを限定

こちらは `const` を使いたいが、条件によって初期値を変えたい場合の発展テクニックです。

[IIFE: Immediately Invoked Function Expressionについて](https://developer.mozilla.org/ja/docs/Glossary/IIFE)

### ⚠️ 注意が必要な書き方

```javascript
// letで宣言して後から代入
let message;
if (score >= 80) {
  message = '優秀';
} else if (score >= 60) {
  message = '合格';
} else {
  message = '不合格';
}
```

### ❌ 起こりうる問題

```javascript
let message;
if (score >= 80) {
  message = '優秀';
}
// else ifを書き忘れると...
console.log(message);  // undefined になる可能性
```

### ✅ こう書くとより安全

```javascript
// IIFEでスコープを限定しconstを使う
const message = (() => {
  if (score >= 80) return '優秀';
  if (score >= 60) return '合格';
  return '不合格';
})();
```

### 💡 メリット

- **`const`が使える** — 条件分岐があっても再代入不要
- **スコープの汚染防止** — IIFE内の変数は外部に漏れない

---

## 3. switch文を避けてオブジェクトを使う

### ⚠️ 注意が必要な書き方

```javascript
// 長いswitch文
switch (statusCode) {
  case 200:
    message = '成功';
    break;
  case 400:
    message = 'リクエストエラー';
    break;
  case 404:
    message = '見つかりません';
    break;
  case 500:
    message = 'サーバーエラー';
    break;
  default:
    message = '不明なエラー';
}
```

### ❌ 起こりうる問題

```javascript
switch (type) {
  case 'A':
    result = 1;
    // breakを忘れた！
  case 'B':
    result = 2;
    break;
}
// type === 'A' のとき result は 2 になってしまう
```

### ✅ こう書くとより安全

```javascript
// オブジェクトを使う
const statusMessages = {
  200: '成功',
  400: 'リクエストエラー',
  404: '見つかりません',
  500: 'サーバーエラー',
};

const message = statusMessages[statusCode] ?? '不明なエラー';
```

```javascript
// 関数のマッピングにも使える
const handlers = {
  click: () => console.log('クリックされました'),
  hover: () => console.log('ホバーされました'),
  blur: () => console.log('フォーカスが外れました'),
};

handlers[eventType]?.();
```

### 💡 メリット

- **簡潔で読みやすい** — 対応関係が一目でわかる
- **break忘れの心配がない** — fall-throughバグを防止
- **`const`が使える** — switch文と違い宣言と同時に初期化できる

---

## 4. 条件分岐を簡潔に

### ⚠️ 冗長な書き方

```javascript
// 冗長なif文
let status;
if (isActive) {
  status = 'アクティブ';
} else {
  status = '非アクティブ';
}

// 冗長なnullチェック
let displayName;
if (userName !== null && userName !== undefined) {
  displayName = userName;
} else {
  displayName = 'ゲスト';
}
```

### ✅ こう書くとより安全

```javascript
// 三項演算子
const status = isActive ? 'アクティブ' : '非アクティブ';

// null合体演算子（??）
const displayName = userName ?? 'ゲスト';

// 論理OR演算子（||）- falsyな値全般
const count = inputCount || 1;
```

#### `??` と `||` の違い

[Falsyな値について](https://developer.mozilla.org/ja/docs/Glossary/Falsy)

```javascript
const value1 = 0 || 10;    // 10（0はFalsyなので）
const value2 = 0 ?? 10;    // 0（0はnullでもundefinedでもない）

const value3 = '' || 'default';   // 'default'
const value4 = '' ?? 'default';   // ''
```

### 📝 補足：結果がbooleanの場合はさらに簡潔に

#### ⚠️ 冗長な書き方


```javascript
const isActive = status === "active" ? true : false;
const hasItems = items.length > 0 ? true : false;
const isValid = value !== null && value !== undefined ? true : false;
```

#### ✅ こう書くとより簡潔

```javascript
// 比較演算子の結果はすでにboolean
const isActive = status === "active";
const hasItems = items.length > 0;
const isValid = value !== null && value !== undefined;

// 明示的にbooleanに変換したい場合は !! を使う
const hasText = !!inputText;  // falsyな値をfalseに、truthyな値をtrueに
```

### 💡 メリット

- **`const`が使える** — 1行で宣言と初期化ができる
- **コードが簡潔** — 意図が明確になり、レビュー時の認知不可も下がる
- **`??`は意図しない変換を防ぐ** — `0`や`''`を有効な値として扱える

---

## 5. 三項演算子のネストは避ける

### ⚠️ 注意が必要な書き方

```javascript
// ネストした三項演算子
const grade = score >= 90 ? 'S' 
            : score >= 80 ? 'A' 
            : score >= 70 ? 'B' 
            : score >= 60 ? 'C' 
            : 'D';
```

### ❌ 起こりうる問題

- 読みにくい、理解しにくい
- 条件の優先順位を誤解しやすい
- バグが入り込みやすく、発見しにくい

### ✅ こう書くとより安全

```javascript
// 関数に切り出す
const getGrade = (score) => {
  if (score >= 90) return 'S';
  if (score >= 80) return 'A';
  if (score >= 70) return 'B';
  if (score >= 60) return 'C';
  return 'D';
};

const grade = getGrade(score);
```

```javascript
// IIFEを使う
const grade = (() => {
  if (score >= 90) return 'S';
  if (score >= 80) return 'A';
  if (score >= 70) return 'B';
  return 'C';
})();
```

### 💡 メリット

- **可読性の向上** — 条件の流れが追いやすい
- **デバッグしやすい** — 各条件にブレークポイントを設定できる
- **保守性の向上** — 条件の追加・変更が容易

---

## 6. オプショナルチェイニングで安全にプロパティアクセス

### ⚠️ 注意が必要な書き方

```javascript
// 冗長なnullチェック
const city = user && user.address && user.address.city;

// 長い条件式
let city;
if (user !== null && user !== undefined) {
  if (user.address !== null && user.address !== undefined) {
    city = user.address.city;
  }
}
```

### ❌ 起こりうる問題

```javascript
// userがundefinedだと...
const city = user.address.city;
// TypeError: Cannot read property 'address' of undefined

// APIレスポンスが想定外の形式だと...
const name = response.data.users[0].name;
// どこかがundefinedならクラッシュする
```

### ✅ こう書くとより安全

```javascript
// オプショナルチェイニング
const city = user?.address?.city;
const firstItem = items?.[0];
const result = obj?.method?.();
```

### 💡 メリット

- **簡潔で安全** — ネストしたプロパティに安全にアクセス
- **エラー防止** — `Cannot read property of undefined`を防ぐ
- **コードの可読性向上** — 意図が明確

---

## 7. 早期リターンでネストを浅く

### ⚠️ 注意が必要な書き方

```javascript
// 深いネスト
const processUser = (user) => {
  if (user) {
    if (user.isActive) {
      if (user.age >= 18) {
        return {
          name: user.name,
          email: user.email,
        };
      }
    }
  }
  return null;
};
```

### ❌ 起こりうる問題

- コードの見通しが悪くなる
- どの条件でどの処理が実行されるか把握しにくい
- バグの原因となる「閉じ括弧の対応ミス」が起きやすい

### ✅ こう書くとより安全

```javascript
// 早期リターン（ガード節）
const processUser = (user) => {
  if (!user) return null;
  if (!user.isActive) return null;
  if (user.age < 18) return null;

  // メインロジック（ネストなし）
  return {
    name: user.name,
    email: user.email,
  };
};
```

### 💡 メリット

- **可読性の向上** — メインロジックが目立つ
- **認知負荷の軽減** — 条件を上から順に確認できる
- **保守性の向上** — 条件の追加・削除が容易

---

## 8. Object.freezeで定数オブジェクトを保護

### ⚠️ 注意が必要な書き方

```javascript
// constだけでは中身は変更可能
const CONFIG = {
  API_URL: 'https://api.example.com',
  TIMEOUT: 5000,
};

// これは可能（constは再代入を防ぐだけ）
CONFIG.TIMEOUT = 99999;  // 変更できてしまう
```

### ❌ 起こりうる問題

```javascript
const DEFAULT_OPTIONS = { retries: 3 };

const fetchData = (options = DEFAULT_OPTIONS) => {
  options.retries--;  // DEFAULT_OPTIONSも変更される！
  // ...
};

fetchData();
fetchData();
fetchData();
console.log(DEFAULT_OPTIONS.retries);  // 0 になっている
```

### ✅ こう書くとより安全

```javascript
// Object.freezeで不変にする
const CONFIG = Object.freeze({
  API_URL: 'https://api.example.com',
  TIMEOUT: 5000,
  MAX_RETRY: 3,
});

// 変更しようとしても無視される（strictモードではエラー）
CONFIG.TIMEOUT = 10000;  // 変更されない
```

```javascript
// 深いフリーズが必要な場合
const deepFreeze = (obj) => {
  Object.keys(obj).forEach(key => {
    if (typeof obj[key] === 'object' && obj[key] !== null) {
      deepFreeze(obj[key]);
    }
  });
  return Object.freeze(obj);
};

const NESTED_CONFIG = deepFreeze({
  server: { host: 'localhost', port: 3000 },
});
```

### 💡 メリット

- **不変性の保証** — 設定値の意図しない変更を防止
- **バグの早期発見** — strictモードでは変更時にエラー
- **予測可能性** — オブジェクトの状態が常に一定

---

## 9. Array.fromやスプレッド構文で安全にコピー

[シャローコピーとディープコピーについて](https://medium-company.com/%E3%83%87%E3%82%A3%E3%83%BC%E3%83%97%E3%82%B3%E3%83%94%E3%83%BC%E3%81%A8%E3%82%B7%E3%83%A3%E3%83%AD%E3%83%BC%E3%82%B3%E3%83%94%E3%83%BC%E3%81%AE%E9%81%95%E3%81%84/)

### ⚠️ 注意が必要な書き方

```javascript
// 配列のシャローコピー(実体は同じ)
const original = [1, 2, 3];
const notCopied = original;

notCopied.push(4);
console.log(original);  // [1, 2, 3, 4] 元の配列も変わる！

// オブジェクトのシャローコピー(実体は同じ)
const originalObj = { a: 1, b: 2 };
const notCopiedObj = originalObj;

notCopiedObj.b = 3;
console.log(originalObj);  // { a: 1, b: 3 } 元のオブジェクトも変わる！
```

### ❌ 起こりうる問題

```javascript
const defaultItems = ['item1', 'item2'];

const createList = () => {
  const items = defaultItems;  // 参照をコピー
  items.push('newItem');       // defaultItemsも変わる！
  return items;
};

createList();
createList();
console.log(defaultItems);  // ['item1', 'item2', 'newItem', 'newItem']
```

### ✅ こう書くとより安全

```javascript
// スプレッド構文でコピー
const original = [1, 2, 3];
const copied = [...original];

// Array.fromでコピー
const copied2 = Array.from(original);

// スプレッド構文でコピー
const originalObj = { a: 1, b: 2 };
const copiedObj = { ...originalObj };
```

### 📝 補足：階層が深くなる場合のスプレッド構文は注意が必要

```javascript
const original = [1, 2, [3, 4]];
const copied = [...original];

copied[2].push(5);
console.log(original) // [1, 2, [3, 4, 5]] 元の配列の3階層目も変わる！

// スプレッド構文でコピーが保証されるのは1階層目まで
const originalObj = {
  a: 1,
  b: { c: 2, d: 3 },
};
const copiedObj = { ...originalObj };

copiedObj.b.c = 99;
console.log(original) // { a: 1, b: { c: 99, d: 3 } } 元のオブジェクトのbも変わる！
```

#### 全階層のコピーが必要な場合

```javascript
// JSON経由（関数やundefinedは失われる点に注意）
const deepCopied = JSON.parse(JSON.stringify(original));

// Node v17~ 使える
const deepCopied2 = structuredClone(original);
```

### 💡 メリット

- **副作用の防止** — 元のデータを変更しない
- **予測可能な動作** — コピーへの変更が元に影響しない
- **デバッグが容易** — データの流れを追跡しやすい

---

## 10. 配列操作は非破壊メソッドを優先

### ⚠️ 注意が必要な書き方

```javascript
// 破壊的な操作
const numbers = [3, 1, 4, 1, 5];
numbers.sort((a, b) => a - b);  // 元の配列が変わる
numbers.reverse();              // 元の配列が変わる
numbers.splice(2, 1);           // 元の配列が変わる
```

### ❌ 起こりうる問題

```javascript
const renderList = (items) => {
  items.sort();  // 引数を破壊！
  return items.map(item => `${item}`).join('');
};

const myItems = ['banana', 'apple', 'cherry'];
renderList(myItems);
console.log(myItems);  // ['apple', 'banana', 'cherry'] 順番が変わった！
```

### ✅ こう書くとより安全

```javascript
// 非破壊的な操作
const numbers = [3, 1, 4, 1, 5];

// ソート
const sorted = [...numbers].sort((a, b) => a - b);

// リバース
const reversed = [...numbers].reverse();

// 要素の削除
const removed = numbers.filter((_, i) => i !== 2);

// 要素の更新
const updated = numbers.map((n, i) => i === 0 ? 10 : n);
```

### 💡 メリット

- **予測可能性** — 元のデータが変わらない安心感
- **デバッグの容易さ** — 各ステップの状態を確認できる
- **並行処理での安全性** — 複数箇所で同じ配列を参照しても安全

---

## まとめ

これらのTipsに共通するのは「**予測可能で副作用の少ないコード**」を書くということです。

| ポイント | 説明 |
|---------|------|
| 値の不変性 | 値が変わらないことを保証する |
| 変更の防止 | 意図しない変更を防ぐ |
| 意図の明確化 | コードの意図を明確にする |

JavaScriptの自由さを活かしつつ、これらのプラクティスを意識することで、バグが少なく保守しやすいコードを書くことができます。

---

:::message
本記事は生成AI（Claude Code）を使用して作成し、筆者が内容を確認・編集しています
:::
