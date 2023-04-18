---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

# TypeScript 5.0

---

# 本日のひとこと

<div style="display: grid; grid-template-columns: 1fr 1fr">
  <div>
    <h5>『ノック 恐怖の訪問者』を観た</h5>
    <img src="/knock_at_the_cabin.jpeg" style="width: 300px" alt="">
  </div>
  <div>
    <ul style="margin-top: 20px;">
      <li>M・ナイト・シャマラン監督作</li>
      <li>前作『オールド』に続いて面白い</li>
      <li>間違いなく現行アメリカ映画のトップ</li>
      <li>黒沢清との関連性</li>
    </ul>
  </div>
</div>


---

# Agenda

- Decorators
- `const` Type Parameters
- Supporting Multiple Configuration Files in `extends`
- Exhaustive `switch/case` Completions
- Breaking Changes

---

# Decorators

- 以前からプロポーザルに存在していた "デコレータ" が Stage 3 と安定したため実装
- これまでの実装は古い仕様 "Legacy Decorators" として継続サポート
- `--experimentalDecorators` というコンパイルフラグ
  - true: "Legacy Decorators"
  - false: 今回の実装

---

# Decorators

どんな機能か？

- JavaScript のクラスを拡張するための提案
- クラスやクラス要素、その他の JavaScript 構文の定義時に呼び出される関数のこと

---

# Decorators

例えば...

`HTMLElement` クラスを拡張した `c` クラスの例

```js
@defineElement("my-class")
class C extends HTMLElement {
  @reactive accessor clicked = false;
}
```

- `clicked` プロパティがデフォルトで `false` 
- `@reactive` デコレータは、 `clicked` にリアクティブ性を追加するカスタムデコレータ
  - `clicked` の値が変更されたらそれに依存した部分が更新されるイメージ
- さらに `@defineElement` デコレータでカスタム要素として登録

```html
<!-- このように使用できる -->
<my-class></my-class>
```

---

# Decorators

TypeScript の場合

単純な `greet()` メソッドが定義された `Person` クラス

```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ray");
p.greet();
```

---

# Decorators

TypeScript の場合

非同期、再帰的処理、副作用などで複雑になった場合デバッグするのが大変になる

```ts
class Person {
  name: string;
  constructor(name: string) {
    this.name = name;
  }

  greet() {
    console.log("LOG: Entering method.");
    console.log(`Hello, my name is ${this.name}.`);
    console.log("LOG: Exiting method.")
  }
}
```

---

# Decorators

TypeScript の場合

ここで便利になるのがデコレータ

```ts
function loggedMethod(originalMethod: any, _context: any) {

    function replacementMethod(this: any, ...args: any[]) {
        console.log("LOG: Entering method.")
        const result = originalMethod.call(this, ...args);
        console.log("LOG: Exiting method.")
        return result;
    }

    return replacementMethod;
}
```

- `originalMethod` を受け取り、二つのログと `this` と `...args` をメソッドに渡すロジックを持っている関数を返す

---

# Decorators

TypeScript の場合

この `loggedMethod` をデコレータとして `Person` クラスへ渡す

```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    @loggedMethod
    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ray");
p.greet();

// Output:
//
//   LOG: Entering method.
//   Hello, my name is Ray.
//   LOG: Exiting method.
```

---

# Decorators

TypeScript の場合

- `@foo` というデコレータを記述することで、メソッドのターゲットとコンテキストオブジェクトが受け取れる
- コンテキストでは、デコレータのターゲットであるメソッドが private か static か、メソッド名などを取得できる

メソッド名を表示する例

```ts
function loggedMethod(originalMethod: any, context: ClassMethodDecoratorContext) {
    const methodName = String(context.name);

    function replacementMethod(this: any, ...args: any[]) {
        console.log(`LOG: Entering method '${methodName}'.`)
        const result = originalMethod.call(this, ...args);
        console.log(`LOG: Exiting method '${methodName}'.`)
        return result;
    }

    return replacementMethod;
}
```

---

# Decorators

複数のデコレータを付与することもできる

- 先ほどの `greet` 関数を独立した関数として呼んだり、コールバックとして渡したりすると再 bind しないので起こられてしまう

```ts
const greet = new Person("Ron").greet;

// これができるようにしたい
greet();
```

---

# Decorators

複数のデコレータを付与することもできる

- `context` には `addInitializer` という関数が用意されており、 `bind` を呼び出すよう記述することが可能
- コンストラクタの始まりに渡した関数を実行できる

```ts
function bound(originalMethod: any, context: ClassMethodDecoratorContext) {
	const methodName = context.name;
	if (context.private) {
		throw new Error(`'bound' cannot decorate private properties like ${methodName as string}.`);
	}
	context.addInitializer(function() {
		this[methodName] = this[methodName].bind(this);
	})
}
```

---

# Decorators

複数のデコレータを付与することもできる

- 実行される順番は逆順
  - `@loggedMethod` => `@bound`
```ts
class Person {
	name: string;
	constructor(name: string) {
		this.name = name;
	}

	@bound // ここに追加
	@loggedMethod
	greet() {
		console.log(`Hello, my name is ${this.name}.`)
	}
}

const p = new Person("Ron");
const greet = p.greet;

// Works!
greet();
```

---

# Decorators

デコレータ関数を返す関数も作成できる

`loggedMethod` がデコレータを返し、メッセージをどのように記録するかをカスタマイズ

```ts
function loggedMethod(headMessage = "Log:") {
	return function actualDecorator(originalMethod: any, context: ClassMethodDecoratorContext) {
		const methodName = String(context.name);
		
		function replacementMethod(this: any, ...args: any[]) {
			console.log(`${headMessage} Entering method '${methodName}'.`)
			const result = originalMethod.call(this, ...args);
			console.log(`${headMessage} Exiting method '${methodName}'.`)
			return result
		}
		return replacementMethod;
	}
}
```

---

# Decorators

デコレータ関数を返す関数も作成できる

```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    @loggedMethod("") // 任意の文字列を渡すことができる
    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ron");
p.greet();

// Output:
//
//    Entering method 'greet'.
//   Hello, my name is Ron.
//    Exiting method 'greet'.
```

---

# Decorators

可読性と型付けのトレードオフになる場合もある

- 厳密に型定義しようとするとこうなるので、なるべくデコレータはシンプルにしましょう

```ts
function loggedMethod<This, Args extends any[], Return>(
    target: (this: This, ...args: Args) => Return,
    context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => Return>
) {
    const methodName = String(context.name);

    function replacementMethod(this: This, ...args: Args): Return {
        console.log(`LOG: Entering method '${methodName}'.`)
        const result = target.call(this, ...args);
        console.log(`LOG: Exiting method '${methodName}'.`)
        return result;
    }

    return replacementMethod;
}
```

---

# `const` Type Parameters

オブジェクトの型推論の話

- このような場合、具体的な型ではなく一般的な型が推論される

```ts
type HasNames = { readonly names: string[] };
function getNamesExactly<T extends HasNames>(arg: T): T["names"] {
  return arg.names;
}

// 推論される型 => string[]
const names = getNamesExactly({ names: ["Alice", "Bob", "Eve"] });
```

---

# `const` Type Parameters

オブジェクトの型推論の話

- `getNamesExactly` が具体的に何をするか、どう使われるかによっては、具体的な型推論が欲しい場合も
- 特定の場所で `as const` を追加して対応できるが...

```ts
// The type we wanted:
//    readonly ["Alice", "Bob", "Eve"]
// The type we got:
//    string[]
const names1 = getNamesExactly({ names: ["Alice", "Bob", "Eve"]});

// Correctly gets what we wanted:
//    readonly ["Alice", "Bob", "Eve"]
const names2 = getNamesExactly({ names: ["Alice", "Bob", "Eve"]} as const);
```

---

# `const` Type Parameters

オブジェクトの型推論の話

- `const` モディファイアを追加して `as const` のような推論がデフォルトで可能になった

```ts
type HasNames = { names: readonly string[] };
function getNamesExactly<const T extends HasNames>(arg: T): T["names"] {
//                       ^^^^^
  return arg.names;
}

// Inferred type: readonly ["Alice", "Bob", "Eve"]
// Note: Didn't need to write 'as const' here
const names = getNamesExactly({ names: ["Alice", "Bob", "Eve"] });
```

---

# Supporting Multiple Configuration Files in `extends`

複数プロジェクトでの管理の話

- TypeScript ではコンパイラが使用する設定ファイルとして `tsconfig.json` を使用する
- 元々存在していた `extends` フィールドの機能が拡張された
  - 今までは `extends` には一つのフィールドのみ指定できた
  - 5.0 からは配列で複数ファイルを指定できるようになった

```json
{
    "extends": ["a", "b", "c"],
    "compilerOptions": {
        // ...
    }
}
```

---

# Exhaustive `switch/case` Completions

https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/#exhaustive-switch-case-completions

---

# Breaking Changes

- ランタイム要件
  - ターゲットが ECMAScript 2018 になった
  - Node ユーザーは Node.js 10以降のバージョンが最低条件になる

---

# Breaking Changes

- 関係演算子に関する型エラー
  - `<` や `>=` などに対するエラーが追加

```ts
function func1(ns: number | string) {
  // string である可能性もあるためエラー
	return ns * 4;
}

function func2(ns: number | string) {
  // pass
  return nus > 4;
}
```

```ts
// 5.0 からは
function func3(ns: number | string) {
	// Operator '>' cannot be applied to types 'string | number' and 'number'.
	return nus > 4;
}
```

---

# Breaking Changes

- Enum に関する型エラー
  - ドメイン外のリテラルを Enum へ代入

```ts
enum SomeEvenDigit {
    Zero = 0,
    Two = 2,
    Four = 4
}
// Now correctly an error
let a: SomeEvenDigit = 1;

// OK
let b: SomeEvenDigit = 2
```

---

# Breaking Changes

- Enum に関する型エラー
  - 型が混在した Enum

```ts
enum Letters {
	A = "a"
}

enum Numbers {
	one = 1,
	two = Letters.A
}

// Now correctly an error
const t: number = Numbers.two;
```