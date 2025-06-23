# SGDC 向け実装ガイドライン(Backend Kotlin 編)

プログラミングの基本原則からBill Oneアーキテクチャのマクロまで

## 実装前の必読: ボーイスカウト精神

新規の実装であれば、この章は飛ばしても構いませんが、できれば一度目を通してください。

まずは、[Boy Scout Rule (EN)](<(https://www.oreilly.com/library/view/97-things-every/9780596809515/ch08.html)>) 、[ja](<(https://www.oreilly.com/library/view/97-things-every/9780596809515/ch08.html)>) を読みましょう。

すでに存在するコードを変更する場合は、対象コードが本ガイドラインに正しく対応しているかを確認しましょう。
適合していない場合は、本ガイドラインに基づきリファクタリングを行いましょう。
というか、機能改修を行う全ての開発案件において、最初にリファクタリングの工数を取っておきましょう（工数の%については有識者に相談してください）。

リファクタリングは社会的責任であるだけでなく、既存実装の理解を深める大きなチャンスでもあります。

ただし、リファクタリングは難易度が高い場合も多く、時間を消耗することもあります。 難しさを感じたり早期に手を打ちたい場合は、一人で抱え込まず有識者に相談しましょう。

## ミクロ

### クラス定義

`data class`や`data object`を使用しましょう。

`data`ではないクラスは`equals`や`toString`などの実装が異なる為、簡単なバグを生む原因になります。
なお、テストコードにおいて`class`を用いることはあります。

良い例：

```kotlin
data class A()
data object B
```

良くない例：

```kotlin
class A()
object B
```

### Interface定義

関連するクラスは Interface にまとめましょう。

また、実装を直接見るのではなく Interface を介して確認するようにしましょう。

`sealed interface`を使用すると、継承先が制限されます。それにより型安全性が増します。特に理由のない限りは`sealed interface`にしましょう。

例：

```kotlin
sealed interface User {
    val name: String
}

data class ActiveUser(override val name: String): User
data class DeletedUser(override val name: String): User

val user: User = UserRepository.get()

when (user) {
    is ActiveUser -> {}
    is DeletedUser -> {}
    // else は不要！ 新しい継承先(SUspendedUser等)が増えたらコンパイルエラーで注意を与えられる
}
```

### 関数定義

```kotlin
fun f(arg: T): R
```

#### シグネチャの原則

- 引数は "広く"、返り値は "狭く"
- シグネチャは、関数が「何を受け取り、何を作るか」を明示するもの

この原則に違反しているコードのことを未来予知コードと呼んでください。

本来知り得ない・知り得てはいけないはずのクライアント側の実装都合を知っているからです。つまり責務違反を意味しています。

#### 引数

例えばある永続化関数を考えます。

```kotlin
fun storeRows(rows: List<Row>): Unit
```

→ 本当に`List`である必要はあるのでしょうか？定義を広くしてみましょう。

```kotlin
fun storeRows(rows: Iterable<Row>): Unit
```

とすればクライアントは`List`や`Set`等を渡せます。

#### 返り値

例えばある読み出し関数を考えます。

```kotlin
fun getRows(): Iterable<Row>
```

→ 本当に`Iterable`である必要はあるのでしょうか？定義を狭くしてみましょう。

```kotlin
fun getRows(): List<Row>
```

とすればクライアントは返り値が具体的にどういう値なのかを考慮しなくて済みます。

#### 引数が関数

頻出ポイントはフロントエンドなのと、typescriptには便利な `unknown` 型があるのでここだけtypescriptで説明します。

引数に存在する関数の場合、引数の原則”より広く”に従うため、その関数に対して適用するシグネチャの原則は逆になります。

例えばあるボタンのcomponentの引数に `handleCreate` があると仮定しましょう。下の定義の方がより優れています（可能である限り）。

```typescript
handleCreate: (request: Json) => Promise<Result<void, void>>; // Promiseであり、かつResultであり、かつその中身に対して関心がある。すごい狭い。
handleCreate: (request: Json) => Promise<Result<unknown, unknown>>; // Promiseであり、かつResultであることには関心があるが、中身はどうでもいい
handleCreate: (request: Json) => Promise<unknown>; // Promiseであることに関心があるが、中身はどうでもいい
handleCreate: (request: Json) => unknown; // JSONを受け取ってくれる関数ならなんでも良い。すごい広い。

handleCreate: (request: any) => unknown; // anyからstringを取り出し、jsonに変換するという、過大な責務を押し付けることになる
handleCreate: (request: string) => unknown; // stringを取り出し、jsonに変換するという、過大な責務を押し付けることにになる
handleCreate: (request: Json) => unknown; // Jsonを受け取ってくれる関数ならなんでも良い。すごい狭い。
```

### コレクションの考え方

#### Nullable

```kotlin
val a: List<Int>?
```

Nullableなコレクション定義はおよそ間違いなく不適切です。
嫌がらせにPRレビューの際`Nullな時と空コレクションの違いは何ですか`と質問してみましょう。実装者の頭は爆発します。

#### Set(集合)

sortするのはやめましょう。ソートをすると偶然順番が保持されることはあります。しかし、仕様上は順序が無いためその動作は保証されていません。

#### List(リスト)

本当にそのタイミングで順番の情報が必要なのか、もしくはデータの重複が許されるのか考えましょう。基本的に`Set`の方がより強い制約を持つため安全です。

#### 使い分け

- 重複排除かつ順序保持したい場合: `LinkedHashSet`
- 重複を許すが順序は気にしない場合: 無い
- 重複排除・順序どちらも不要な場合: `Set`
- 重複を許し、順序も大事な場合: `List`

### Result の考え方

```kotlin
sealed interface Result<out A, out B>
data class Success<out A>(val value: A) : Result<A, Nothing>
data class Failure<out B>(val value: B) : Result<Nothing, B>
```

基本的に`Success`, `Failure`の両方の型にはより多くの情報を入れよう。

特に`Unit`の取り扱いには注意しましょう。

`Success(Unit)`はValidation等で用いることがあります。しかし、であれば`data object Valid`等でValidである状態のシングルトンを定義し、`Success(Valid)`としましょう。クライアントからしてみたら`Valid`から`Unit`への変換はいつでもできます。

`Failure(Unit)`は絶対に定義してはいけません。`Failure`になった時点で何らかの理由があるはずですが、その理由が消失しているためです。`Success`と同様、理由の抹消はクライアント側でもできますが、復元はできません。失敗理由の消失はその後のハンドリングなどに大きな影響を及ぼします。

その値を実際にクライアントコードが使うかは分かりません。しかし関数のシグネチャの原則にもあるように、その判断をResultを返す関数でやるべきことではありません。
また、情報を豊かにすることでテスト容易性も高まります。
2つの値を取りうることを表現するのがResultの役割。勝手に端折るのはやめましょう。Unitが使われているResultを見つけたら、その関数は未来予知をしていると考えよう。責務違反。

#### SuccessやFailureの値

基本的に`sealed interface`で構造化し、その親クラスを指定しましょう。

それによりクライアントコードは`when`等で安全に成功パターン・失敗パターンを処理できます。

### コメントの考え方

色々ベストプラクティスはありますが、ここでは*Why not*を説明するコメントについて説明します。

#### リファクタリングの敗北

時間・品質・スコープ何かしらの問題でリファクタリングを断念・途中したら、コメントを残そう

- なぜリファクタリングしなかったのか？
  - 現状における理想状態とは
  - その実現を阻む課題

このあたりを書いておけば、後続の開発者が読んでもしかしたら代わりにやってくれるかもしれないし、別の案をだしてくれるかもしれない。なにより同じ内容を検討するのを避けることができる。

#### 場当たり対応

特に場当たり的対応をした場合は _Why not_ を説明するためのコメントを残しましょう。
初めてそのコードを読んだ人が抱く、その時なぜ〇〇しなかったのか？という疑問に対して納得できるような文言が必要です。

実例1

- 読み手の疑問・・・なぜ発行機能から送られた請求書と説明されているのにクラス名が `IssuingInvoice` から始まらないの？
- コメント・・・**なぜ** `IssuingInvoice` という名前を使え **ない** のかというと、

```kotlin
        /**
         * ja: 発行機能からBill One受領ユーザーに送られた請求書のリマインドメール
         * 発行機能からBill One受領ユーザーへ送った請求書は、InvoiceサービスではSenderInvoiceとして紐づいているため、senderInvoiceRemindResourceとして定義している。
         * en: Remind email sent to the Bill One recipient user from the issuing function.
         * The invoices sent to the Bill One recipient user from the issuing function are linked as SenderInvoice in the Invoice service, hence defined as senderInvoiceRemindResource.
         * ...
         * */
        data class SenderInvoiceRemindResource(
            ...,
        ) : Resource {
            ...
        }
```

実例2

- 読み手の疑問・・・なぜ`distinct()`する必要があるのだろう？どのタイミングで重複するの？
- コメント・・・ **なぜ** rowsのinvoiceUUIDをそのまま使用でき **ない** のかというと、

```kotlin
            // rowsには費用按分の関係で同じ請求書で複数レコードが入る可能性があるので重複を削除する
            val invoiceUUIDs = rows.map { it.invoiceUUID }.distinct()
```

### 型安全性は正義

型安全性は実行時エラーを防止する安全装置です。
その安全装置を破壊する`as`の使用は重罪です。安易に`as`をせずどうすれば回避できるか、なぜそこで型キャストをしないといけないのかを考えましょう。
もしあなたがレビューしているコードで`as`を見つけたら脊髄反射的にRequest Changesを投げましょう。理由は要りません。

なおテストコードにおける`as`使用はOKです。そこで落ちるのであれば、何か考慮が足りていないことを意味するためです。

### Public と Private

Publicな関数には必ず返り値の型を明示しましょう。可能な限りkdocで説明を書きましょう。

Privateはご自由にどうぞ。

### 小さい関数

関数を定義するとき、以下のようにブロックではなく、`=`で定義できないかを考えよう（テクニック）。

小さい関数を定義すべきという原則は理解できていても関数は勝手に肥大化していってしまうもの。
その理由はブロックで関数定義をしてしまうと、実装を色々と付け加えることができてしまうから。
`=` なら相当な無茶をしない限りは複雑なことにならない。

なお、全部を `=` で定義できる訳ではないので、どうしても `=` で定義できなさそうであればブロックでもOK。

良い例：

```kotlin
fun f(): Int =
    1
```

良くない例：

```kotlin
fun f(): Int {
    UserRepository.removeAll()  // 極論だが、こういうことができてしまう
    return 1
}
```

### 副作用の取り扱い

考え中

### スコープ関数

`let` `run` `also` 等のスコープ関数は、多用しすぎると可読性に問題が生じます。

Bill Oneのプロダクションコードにおいては `let` と `also` でユースケースの99%をカバーできることを抑えておきましょう。

#### `let`

letに渡す関数は可能な限りPrivate関数を定義しよう。

そして純粋関数(※)にすること。

良い例：

```kotlin
someVariable.let(::whatYouWantToDo)

fun whatYouWantToDo(x: Int): String =
    // foobarbaz...
```

良くない例

```kotlin
someVariable.let { x ->
    // foobarbaz...
}
```

#### `also`

ログや永続化などの副作用を発生させる関数を入れよう。

この関数を見れば、あ、この内部で呼ばれているこれは副作用を持つのだな、と認識できるので読み手の思考負荷が減ります。

#### `run`

プロダクションコードで使用するのはやめましょう。

テストのみの使用を強く推奨します。
スコープが広がり読み手に過度な思考負荷を強いる為です。

## マクロ

TODO: 各レイヤにおいてどのようなバリデーションをするかを書く

### 思想・哲学

Bill OneにおけるDDDへのリンクを張る

### 実装の順番

Domain → Infrastructure → ApplicationService → Controllerの順で実装していく。

PRもその単位で作ろう。それぞれにテストコードも忘れずに。

### Domain

- 設計に一番時間を使うようにする
- Always Valid Modelとは・・・
  - →従って制約は `check` で表現するのが良い・・・
- null使わない（意味論）
- 謎のgetter生やさない（責務違反）
  - `isX()`
  - `getX()`
  - `findX()`
- copy()の制限(private constructorの話を書く・・・？)
- モデル
  - 遷移から切るパターン
    - InitialState -> State1 -> State2 -> Finalize
  - 概念から切るパターン
    - type X = A | B（必要に応じてA <-> Bと相互に切り替わるイメージ）
- 設定というモデルの考え方
- トリレンマの話(https://enterprisecraftsmanship.com/posts/domain-model-purity-completeness/)

#### Aggregate

- ListではなくSetを使おう
- ValidationはValidationObjectを作るかcompanion objectにするか
- First Class Collectionが必要になるケースはほぼ無い
- 構成するValue Object同士の整合を制約で
- methodは全てResultを返すようにする
  - 変更するときはproperty一つずつ
  - signatureは`Result<Pair<Aggregate, DomainEvent>, FailureReason>`で全部揃えればいい
- 吐くdomain eventは一つの操作につき1つだけ

#### Entity

- 注意事項はAggregateと同じ
- 本当にAggregateとして切り出せないか考える

#### Value Object

- `data class`使うこと
- 必ず制約を入れる
- 複数の値で構成されるVOはありうる

#### Domain Event

- `DomainEvent`を継承する・・・
- 要素は全てVOで構成・・・

#### Factory

- 命名規則
- `Result<Pair<Aggregate, DomainEvent>, FailureReason>`

### Infrastructure

#### Repository

- Aggregateと1:1で対応させる
- （90%のニーズにおいて）とりあえず`get`と`store`さえあればいい
  - 纏めて取得したいとかは全部QueryService

#### なんだか良く分からん奴

- 構造はInfrastructure内にDTOとして定義する

### ApplicationService

- 使うRepositoryは一つだけ
- 設定の扱い
- Validationの扱い
- 基本get->mutate->storeの流れで完結させる

### QueryService

- パフォーマンス最優先

### Controller

- ApplicationService/QueryServiceの返り値とControllerの値を変換するのが責務
- 基本parseだけする。Validateはしない。＝ValueObjectに直接ここで変換しない

## テスト

- inner classによる構造化
- 責務内のテストに限定する
  - 例えばApplicationServiceでやったテストをControllerでやらない
