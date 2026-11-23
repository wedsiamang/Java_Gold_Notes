# ジェネリクス

> **全体を通すルール**
>
> コンパイラは**変数の宣言型だけ**を見て判断する。実際に何が入っているかは見ない。

- ジェネリクスとは、クラスやインタフェースの定義時点で型をパラメータ化しておき、利用時に具体的な型を指定する
- ジェネリクスを指定したクラスをジェネリッククラス、インタフェースをジェネリックインタフェースと呼ぶ
- クラス名やインタフェース名にあたる識別子の後ろに<>を記述して型パラメータを宣言する
- 型パラメータはカンマで区切り複数指定ができ、これを型パラメータリストと呼ぶ

---

class クラス名 <型パラメータリスト> {}
interface インタフェース名 <型パラメータリスト> {}

---

```
class Foo<T> {}
interface Bar<T,R> {}
```

- 型パラメータに割り当てられる具体的な型はわからないため、型パラメータを使用したインスタンス化はできない
- 型パラメータは非static であり、staticコンテキストからのアクセスができないため、staticメンバに型パラメータを指定することができない

## ダイヤモンド演算子
- ダイヤモンド演算子<>を使うと型推論される

## ジェネリックメソッド
- クラスをジェネリクス指定せずメソッドのみにジェネリクスを指定できる. 


## 原型（raw type）

> **要点**
>
> `<>` を省略してジェネリクスのクラスを使うと、T はすべて **Object** として扱われる。
> 中身が String でも、String では受け取れない。

- **原型（raw type）**：型パラメータを持つクラスを `<>` なしで使う書き方。コンパイル時に unchecked 警告が出るが、エラーではない。
- **型消去（type erasure）**：型のチェックはコンパイル時に行われる。実行時には型パラメータの情報は消え、Object として扱われる。

```java
public class Box<T> {
    private T val;
    public Box(T val) { this.val = val; }
    public T get() { return val; }
}
```

原型の `Box b = new Box("Hello");` から `b.get()` を受け取る場合：

| 受け取る型 | 判定 | 理由 |
|---|---|---|
| Object | ✅ | 戻り値が Object 扱いなので、そのまま受け取れる |
| String | ❌ | `Object cannot be converted to String`。中身が String でも不可 |
| Box | ❌ | 同上 |
| T | ❌ | `cannot find symbol`。T は Box クラスの中だけで有効な名前 |

`Box<String> b = new Box<>("Hello");` と宣言すれば、String でも Object でも受け取れる。

### キャストは変換ではない

- **基本型のキャスト**：`(double) 100` → `100.0`。値を作り変える。
- **参照型のキャスト**：`(String) obj`。何も作り変えない。中身がその型かどうかを確認するだけ。違えば実行時に `ClassCastException`。

```java
Box raw = new Box(100);
Integer x = (Integer) raw.get();   // ✅
String  s = (String)  raw.get();   // コンパイルは通る → 実行時 ClassCastException
```

数値を文字列にしたいときは、キャストではなく `String.valueOf(x)` や `x.toString()` を使う。

<details>
<summary><code>Box&lt;Integer&gt; b = new Box&lt;&gt;(100);</code> のとき <code>/* ここ */ a = b.get();</code> に入るのは？（Integer / Number / Object / String / int）</summary>

Integer・Number・Object・int。Number と Object は親の型として受けられる。int はオートアンボクシング。String は親子関係がないので不可。

</details>

<details>
<summary><code>Box b = new Box(100); String s = (String) b.get();</code> はどうなる？</summary>

コンパイルは通る（Object → String のキャストは文法上ありえる）。実行時に `ClassCastException`。

</details>

## 非変（invariance）

> **要点**
>
> `A a = new B();` は通るが、`Item<A> a = new Item<B>();` は**通らない**。
> `<>` の中は、親子関係があっても**同じ型どうししか代入できない**。

- **変性**：親子関係のある型どうしを、どこまで代入可能とするかという性質。
- **共変（covariance）**：子の型を親の型の変数に代入できる。**配列**はこれ。
- **非変（invariance）**：子の型の代入も許さない。**ジェネリクス**はこれ。
- **反変（contravariance）**：親の型の代入を許す。`<? super B>` がこの性質を持つ。

B extends A のとき：

| 代入 | 判定 |
|---|---|
| `Item<A> a = new Item<B>();` | ❌ `Item<B> cannot be converted to Item<A>` |
| `Item<A> a = new Item<A>();` | ✅ |
| `Item<Object> a = new Item<A>();` | ❌ |
| `Item<? extends A> a = new Item<B>();` | ✅ ワイルドカードで緩めれば可 |

### なぜ非変なのか

配列は共変なので、次のコードはコンパイルが通り、実行時に壊れる。

```java
String[] s = {"A", "B"};
Object[] o = s;              // 共変なので代入できる
o[0] = Integer.valueOf(1);   // コンパイルは通る → 実行時 ArrayStoreException
```

もし `List<Object> list = new ArrayList<String>();` が許されると、コンパイラは宣言型 `List<Object>` しか見ないので、`list.add(100)` を止められない。そのため、**入口の代入そのものを禁止した**。

<details>
<summary><code>List&lt;Number&gt; list = new ArrayList&lt;Integer&gt;();</code> は通る？</summary>

通らない。非変なので、Integer が Number の子でも代入不可。通すなら `List<? extends Number>`。

</details>

## ワイルドカード

非変は厳しすぎるので、制限を緩める手段としてワイルドカードがある。その代わり、get / set に制限がかかる。

### 非境界ワイルドカード `<?>`

> **要点**
>
> `<?>` は「何でもいい」ではなく「**何かに決まっているが、分からない**」。
> 取り出しは **Object** で受ける。入れられるのは **null だけ**。

- **CAP#1**：コンパイラが「正体不明の型」に付ける仮の名前。エラーメッセージに出てくる。

| | `Test<Object>` | `Test<?>` |
|---|---|---|
| 意味 | Object 用の箱と**決まっている** | 何かの箱だが**分からない** |
| `= new Test<String>()` | ❌ 非変 | ✅ |
| get | Object | Object |
| set | 何でも入る | **null だけ** |

```java
Test<?> test = new Test<>("Hello");
Object o   = test.getValue();     // ✅
String msg = test.getValue();     // ❌ CAP#1 cannot be converted to String
test.setValue("sample");          // ❌ String cannot be converted to CAP#1
test.setValue(new Object());      // ❌ Object cannot be converted to CAP#1
test.setValue(null);              // ✅ null はどの参照型にも代入できる
```

Object を入れられないのは、箱の正体が `Test<Integer>` かもしれないから。Object を入れられたら、むしろ一番危険になる。

<details>
<summary><code>Test&lt;?&gt; t = new Test&lt;&gt;(10);</code> のとき、通るのは？ (ア) <code>Integer a = t.getValue();</code> (イ) <code>Object b = t.getValue();</code> (ウ) <code>t.setValue(20);</code> (エ) <code>t.setValue(null);</code></summary>

(イ)と(エ)。右辺の 10 はコンパイラに見えていない。`t` には後から `Test<String>` を代入することもできるので、20 は入れられない。

</details>

### 上限境界ワイルドカード `<? extends>`

> **要点**
>
> `<? extends Number>` ＝「Number **か、その子の何か**用の箱。どれかは分からない」。
> 取り出しは **Number** で受けられる。入れられるのは **null だけ**（`<?>` と同じ）。

- **上限境界ワイルドカード**：`<? extends B>`。B か、B の子だけを型パラメータとして受け付ける。B 自身も含む。

```java
List<? extends Number> list = new ArrayList<Integer>();
list.add(10);                 // ❌ int cannot be converted to CAP#1
list.add(null);               // ✅
for (Number n : list) { }     // ✅ Number で受けられる
Integer i = list.get(0);      // ❌ CAP#1 cannot be converted to Integer
```

`List<? extends Number>` の正体は `List<Integer>` かもしれないし、`List<Double>` かもしれない。`List<Double>` だった場合に 10 を入れると壊れるので、コンパイラは止める。

Number は Integer・Double などの親で、インタフェースではなく**抽象クラス**。

<details>
<summary><code>List&lt;? extends Number&gt; list = new ArrayList&lt;Double&gt;();</code> のとき、通るのは？ (ア) <code>Number n = list.get(0);</code> (イ) <code>Double d = list.get(0);</code> (ウ) <code>Object o = list.get(0);</code> (エ) <code>list.add(1.5);</code> (オ) <code>list.add(null);</code></summary>

(ア)(ウ)(オ)。(イ)は、右辺が Double でもコンパイラには見えていない。(エ)は null 以外なので不可。

</details>

### 下限境界ワイルドカード `<? super>`

> **要点**
>
> `<? super B>` ＝「B **か、B より上**」。B 自身も含む。
> 入れるときは **B と B の子**を入れられる。取り出しは **Object** で受ける。

UML のクラス図では、白抜き三角の矢印は **子 → 親** の向きに引く。

```
      A        ← 一番上の親
    ↗   ↖
   B     D     ← どちらも A の子。B と D は無関係（兄弟）
   ↑
   C           ← B の子
```

```java
Test<? super B> a = new Test<A>();   // ✅ 親
Test<? super B> b = new Test<B>();   // ✅ 自分自身
Test<? super B> c = new Test<C>();   // ❌ 子
Test<? super B> d = new Test<D>();   // ❌ 兄弟（無関係）

a.setValue(new B());                 // ✅
a.setValue(new C());                 // ✅ B の子
a.setValue(new A());                 // ❌ A cannot be converted to CAP#1
Object o = a.getValue();             // ✅
B bb = a.getValue();                 // ❌ CAP#1 cannot be converted to B
```

- **入れる（set）**：箱の正体は `Test<B>`・`Test<A>`・`Test<Object>` のどれか。どれでも B は入る（B の子の C も入る）。A は `Test<B>` だった場合に入らないので不可。
- **取り出し（get）**：中身は A かもしれないし Object かもしれないので、確実なのは Object だけ。

<details>
<summary><code>List&lt;? super Integer&gt; list = new ArrayList&lt;Number&gt;();</code> のとき、通るのは？ (ア) <code>list.add(10);</code> (イ) <code>list.add(3.14);</code> (ウ) <code>Number n = list.get(0);</code> (エ) <code>Object o = list.get(0);</code> (オ) <code>List&lt;? super Integer&gt; l2 = new ArrayList&lt;Object&gt;();</code></summary>

(ア)(エ)(オ)。(イ) Double は Integer の子ではない。(ウ) 取り出しは Object のみ。

</details>

## まとめ

| | 代入できる箱 | 取り出し（get） | 入れる（set） |
|---|---|---|---|
| `<T>`（ワイルドカードなし） | 同じ型だけ | T | T |
| `<?>` | 何でも | Object | null だけ |
| `<? extends B>` | B と B の子 | B で受けられる | null だけ |
| `<? super B>` | B と B の親 | Object | B と B の子 |

- **PECS**：Producer-Extends, Consumer-Super。値を出す側（取り出し専用）は `extends`、値を受け取る側（入れる専用）は `super`。

> **よくある誤解**
>
> - `<?>` は「何でも入る」ではない。入るのは null だけ
> - `<? extends>` に子の値を add できそうに見えるが、null しか入らない
> - `<? super B>` には B 自身も含まれる

確認環境：JDK 21。エラーメッセージは javac の実際の出力。

参考：[JLS §4.8 Raw Types](https://docs.oracle.com/javase/specs/jls/se17/html/jls-4.html#jls-4.8)