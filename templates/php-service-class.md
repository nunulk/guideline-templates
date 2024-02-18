# Service クラスを使う際のガイドライン

# はじめに

## このドキュメントについて
Service クラス作成時および利用時のガイドラインについて記載します。サンプルコードは Laravel ですが、他のフレームワークを使う場合は適宜置き換えてください。

## このドキュメントが必要な理由
Service クラスは人によって解釈が違うことにより統一性が保ちにくかったり、導入することで却って不具合の原因になることがあるので、Service クラスを扱う際のガイドラインが必要と思われます。

## 前提
### Service クラスとは
Martin Fowler の「Patterns of Enterprise Application Architecture」と Eric Evans の「Domain-Driven Design」の定義にずれがあるため、利用者に混乱が生じているのだと思われます。本ドキュメントでは両方を取り込んだ上でガイドラインを定めます。

> アプリケーションの境界をサービス層を使って定義する。サービス層は利用可能な操作を定め、各操作へのアプリケーションレスポンスを取りまとめる。
> https://bliki-ja.github.io/pofeaa/ServiceLayer/

Service レイヤーにあるクラスは原則的には複数のクライアント（アクター）に対して共通のインタフェースを提供するために作成されるものです。
Martin Fowler's bliki には記載がないですが、「Patterns of Enterprise Application Architecture」にはビジネスロジックは Model に書けという注意書きがあります。

> The classes implementing the facades don’t implement any business logic. Rather, the Domain Model (116) implements all of the business logic.
>
> Fowler, Martin. Patterns of Enterprise Application Architecture

一方、Evans 本では以下のようにドメインロジック（ビジネスロジックと同義とします）を Service に切り出す際の指針が書かれています。

> ドメインで扱う概念の中には、1つの機能や処理が単体で存在していて、もの（オブジェクト）として扱うのが不自然なものもある。そうしたものは、サービスという形でユビキタス言語に組み込む。
> https://www.ogis-ri.co.jp/otc/hiroba/technical/DDDEssence/chap2.html#Services

Evans は「Domain-Driven Design」の中で良い Service の特徴を3つ挙げています。

> A good SERVICE has three characteristics.
> 1. The operation relates to a domain concept that is not a natural part of an ENTITY or VALUE OBJECT.
> 2. The interface is defined in terms of other elements of the domain model.
> 3. The operation is stateless.
>
> Eric, Evans. Domain-Driven Design

Entity は ID によって一意性を持つオブジェクトであり、Value Object はプリミティブ型の値をラップし、その値を用いて計算および等価性を判定できるオブジェクトであるため、Service はそのいずれでもないもの、という理解でいいでしょう。

Laravel において Service は ServiceProvider を経由して使うものと捉えられることがありますが、必ずしも必要ではありません。

### stateless とは
初期化以降変更が行われるプロパティを持たないこと。逆にいうと、すべてのプロパティは初期化されて移行変更してはいけません。

```php
class SomeService {
    // $state はミュータブルなのでこのクラスは stateless ではない
    private $state = null;
    // $options は初期化以降変更されないので「状態」ではない
    public function __construct(private array $options) {}
    public function doSomething(): void {
        $this->state = 1;
    }
}
```

### Domain Service と Application Service
Evans によると、Application レイヤーに置くべき Service と Domain レイヤーに置くべき Service にわけられるとされます。たとえば、以下のようにファイルへのエクスポート機能などが Application Service に該当します。

> Here we encounter a very fine line between the domain layer and the application layer. For example, if the banking application can convert and export our transactions into a spreadsheet file for us to analyze, that export is an application SERVICE.
>
> Eric, Evans. Domain-Driven Design

他にも、メールなどによる通知、外部 API との通信など、ドメインロジックを含まない処理が Application Service に該当します。

一方で、Domain Service はドメインロジックを含む Service となります。

### Model クラスとは
Fowler の定義は以下のとおりです。

> 振る舞いとデータをカプセル化した、ドメインのオブジェクトモデル。
> https://bliki-ja.github.io/pofeaa/DomainModel/

ドメインロジックの主な置き場所。Laravel において Model = Eloquent Model と捉えられがちですが、必ずしもそうではなく、POPO（Plain Old PHP Object）も含みます。

### Service と Model の役割分担
たとえば、複数のモデルを順に呼び出す処理があった場合、各モデルの処理はモデルに持たせ、ワークフローを Service に持たせます。クライアントは Service のインタフェースを呼ぶだけで済みます。これにより共通化された複雑なロジックをシンプルなインターフェイス経由で利用できるわけです。

```php
class ModelA {
    // Model は状態を持つ
    private array $attributes;
    public function foo() {}
}
class ModelB {
    // Model は状態を持つ
    private array $attributes;
    public function bar() {}
}
class SomeService {
    // Service は状態を持たない
    public function baz() {
        (new ModelA())->foo();
        (new ModelB())->bar();
    }
}
// client
(new SomeService())->baz();
```
### Service に状態を持たせることの問題点
前述のとおり「共通化された複雑なロジックをシンプルなインターフェイス経由で利用できる」のが利点なので、基本的には API は一つであることが望ましいです。
が、ときどき、複数の API を持った Service クラスを見かけます（止むを得ずそうなる場合もあります）。
そのとき、状態を持っていると、どの時点で Service 内の状態がどうなっているかを理解しながら処理を組み立てる必要が生じるため、認知負荷が高まり、
ひいては、処理の順番と内部の状態が不整合を起こして予期せぬ不具合が発生します。
状態管理は Model 側で行い、それぞれのアトミックな処理のなかで、整合性を取るようにしてください。

## ガイドライン

### 注意制限事項

1. 適用範囲は本ガイドラインが運用開始された日以降に作成されるものとします（既存のものはそのままでよい）
   1. 既存のものをリファクタリングする場合は準拠した形に書き直しても構いません 
2. 本ガイドラインは必要に応じて加筆修正する 
3. 例外は常に許容するが、コードレビューでレビュアーの同意を得てください。また、コメントにて補足を書いてください

### 名前空間

1. `App\Services` 以下に配置してください
2. Application Service は `App\Services\Application` 以下に配置してください
    1. 例）`App\Services\Application\ExportInvoiceService`
3. Domain Service は `App\Services\Domain` 以下に配置してください
    1. 例）`App\Services\Domain\PaymentService`
4. 特定のアクター用のものは上の名前空間に加えてアクター識別子ごとにわけてください
    1. 例）`App\Services\Application\Manager\ExportInvoiceService`, `App\Services\Application\User\ExportInvoiceService`

### クラス名

1. 原則的に `動作名 + Service` としてください
    1. 悪い例）PrefectureService（接頭辞がモノ）, CalcFareService（単語は省略しない）
    2. 良い例）EstimateCongestionLevelService
    3. 特別な例）StripeService（特定のサービスのラッパーのケース。この例では PaymentService にできるならそれでもいいが、Stripe のインタフェースが露出しているなら StripeService としたほうがわかりやすい）
2. 外部サービスの API や SDK をラップしたクラスの場合は `サービス名 + Api` としてください
    1. 例）StripeApi, TwilioApi

### プロパティ

1. 状態を持たせないでください（状態に関しては、本記事の stateless とは の項を参照のこと）
2. 設定値などをプロパティに持つ場合は特別な理由がない限りコンストラクタで初期化してください。特例の場合はコメントで補足してください
3. コンストラクタに引数を渡すようにしてもいいですが、DI できるようにしてください

例）

```php
class SomeService
{
    private string $constantOption;
    public function __construct(private readonly string $dynamicOption) {
        $this->constantOption = 'constant option';
    }
}

// AppServiceProvider
$this->app->bind(SomeService::class, function ($app) {
    return new SomeService(config('is_hoge', false) ? 'dynamic option 1' : 'dynamic option 2');
});

// constructor injection
class Client {
    public function __construct(private SomeService $someService) {}
}

```

### メソッド

1. DI されることを考慮して、すべてのメソッドをインスタンスメソッドにしてください
2. メソッド数はできるだけ少なくしてください。1つだけが最も望ましいです
3. `__invoke` マジックメソッドは使わないでください（ただし、callable を引数に渡す必要があるようなケースは除く）
