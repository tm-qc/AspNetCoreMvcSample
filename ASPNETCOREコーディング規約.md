# 初心者向け ASP.NET Core MVC コーディング規約

## はじめに
作者が、Visual StudioやC#、ASP.NET CORE作成経験なく、詳しくないですが、
今後できるだけ改修、分析で困らないように、共通認識で守った方が良さそうことを、簡単に記載しました。

簡単とはいえ、すべて守るのも難しいとは思います。
極力認識あわせて作成進めれたら幸いです。

一番大事なのは知らない人が見たときにも、わかるようにコードを記載する思いやりの意識かなと思います。

**何かおかしな点があれば、遠慮なく指摘・改善提案をお願いします。**  
**コード形成など、自動管理で便利なプラグインがあれば教えてください。**

## 0. 共通

### 命名の意識
- どんな機能か予測ができる名前にする
- マジックナンバーを使わない
そのうえで命名規則を守る

### コメントについて
多すぎるコメントは好まれませんが、読んだ人がわからないが一番よくありません。  
実装背景、補足、機能の説明など、コードを見てもパッとわからないことを中心に、
今後誰かが見たときに困らないように記載しておきましょう

基本は「 XMLドキュメントコメント（///）」を使うといいと思います。  
「///」で関数に対して以下のようなひな形ができます。

```

/// <summary>
/// 商品情報を取得します
/// </summary>
/// <param name="productId">商品ID</param>
/// <returns>商品情報。見つからない場合はnull</returns>
public async Task<Product> GetProductAsync(int productId)
{
    // 実装
}

```

### 1クラス1責務
コードが長すぎると以下の弊害が生まれます

- 仕様把握、解析、改修ができない
- コードレビューができずに案件が進まない
- テスト実装の範囲が広すぎてできない

人がパッと見てわかる程度の短いコード、分離を意識しましょう。

「1クラス1責務」

これを原則に1ファイルのコードが極力短く、依存性がなく、1クラスの役割を1つに絞る意識が大事です。
※1クラスに複数メソッドはOK。適切な分離で少ないのが一番べスト。

### var
型が明確な場合のみ使用

## 1. 命名規則（Naming Conventions）

### クラス
- **PascalCase** を使用
- モデル（Entity） → 単数形
  - 例：`Product`, `Order`, `Customer`
- Controller → クラス名 + `Controller`
  - 例：`ProductsController`
- Service → クラス名 + `Service`
  - 例：`OrderService`
- Repository → クラス名 + `Repository`
  - 例：`CustomerRepository`

### メソッド・プロパティ・定数
- **PascalCase** を使用
- 名前は **機能が予測できるもの** にする。
  - ❌ 悪い例：`DoWork()`, `Data1`,`public const int MaxRetryCount1 = 3;`
  - ✅ 良い例：`CalculateDiscount()`, `SaveOrder()`,`public const int MaxRetryCount = 3;`

### 変数・引数
- **camelCase** を使用
  - 例：`itemCount`, `customerName`

### モデル・DBの命名
- **PascalCase** を使用
- **主キー** → `Id`（例：`Product.Id`）
- **外部キー** → 親クラス名 + `Id`（例：`CustomerId`）
- **ナビゲーションプロパティ(リレーション)**
  - 親 → 単数形（例：`Customer`）
  - 子 → 複数形（例：`Orders`）

## 2. MVC基本フォルダ構成

```

/Controllers   → 画面遷移・入力受付
/Services      → ビジネスロジック
/Repositories  → データアクセス
/Models        → データモデル
/Views         → 画面(.cshtml)

```

- フォルダ名は基本的には複数形
- Controller は処理を持たず、Service を呼び出す
- レイヤーを分けて責務を明確化

### MVC+Service+Repository 流れのイメージ
```

Views
↓↑
Controllers(機能の呼び出し)
↓↑
Services (ビジネスロジック。業務処理、複雑な計算、トランザクション管理など)
↓↑
Repositories (データ取得・保存（DBアクセス、外部API連携など）)
↓↑
Models (モデル定義のみ)
↓↑
DB

```

## 3. 依存性注入（DI）
ServiceやRepositoryを追加する場合は、Program.csで登録してDIとして使います。

### Program.cs での登録
```csharp

services.AddScoped<IProductRepository, ProductRepository>();
services.AddScoped<IProductService, ProductService>();

```

### Controller でのコンストラクタでDI定義の例
```csharp

public class ProductsController : Controller
{
    private readonly IProductService _service;
    
    //コンストラクタでインターフェース経由で参照
    public ProductsController(IProductService service) 
        => _service = service;
}

```

### DIのメリット
- 変更に強い	
- インターフェース経由なので、メソッドの参照ミスや呼び出し、記載漏れを防げる	
- 依存関係がコンストラクタで明確	
- Program.csで定義一覧ができる	
- テストが簡単（Fakeテスト実装時の差し替えが簡単になる）	

## 4. Controller の設計

### 非同期アクション
```csharp
public async Task<IActionResult> Index()
{
    var products = await _service.GetAllAsync();
    return View(products);
}
```

### 設計原則
- Fat Controller を避け、処理は必ず Service に委譲
- ルーティングは属性ルートを使用（特に何も使わない？ほかにいい方法あったら教えてください）

```csharp
[Route("products/{id}")]
public async Task<IActionResult> Details(int id) 
{ 
    // ... 
}
```

## 5. Service / Repository の役割

- **Repository**：DB操作のみ
- **Service**：ビジネスロジック、Repository参照

```csharp

public class ProductService : IProductService
{
    private readonly IProductRepository _repo;
    
    public ProductService(IProductRepository repo) 
        => _repo = repo;
    
    public async Task<Product> GetProductAsync(int id)
        => await _repo.FindByIdAsync(id);
}

```

## 6. モデルとバリデーション
モデルは処理は書かない、モデル定義のみ記載する

### DataAnnotation を使用したバリデーション
（バリデーションはほかにいい方法あったら教えてください）

```csharp

public class Product
{
    public int Id { get; set; }
    
    [Required]
    public string Name { get; set; }
    
    [Range(1, 9999)]
    public decimal Price { get; set; }
}

```

### Controller 側でのチェック
- Controller で `ModelState.IsValid` をチェック

## 7. コーディングスタイル
- インデントには 4 つのスペースを使用します。 タブは使用しないでください。

## 8. エラーハンドリングとロギング

- Service 層で `try-catch` を行い、`ILogger` でログ出力
- Controller は原則 Service に委譲

## 9. テストしやすさ（Testability）

- Service / Repository はインターフェース経由で設計
- モック化、Fake差し替えを可能にしてユニットテストを容易に

### 補足
- Fakeクラス：自分で作成するテスト用クラス
- Mockクラス：モックフレームワークで使える。自動生成で簡単に実装できるテスト用のクラス(テスト不要の関連クラスをtrue/falseで返すなど)

## 10. コードレビュー
ASP.NET COREに慣れてない人もいるので、とりあえず以下の方針

- 動くか確認
- コードについてはわかる範囲で確認

## 11. 参考リンク

### 公式ドキュメント
- **C# コーディング規約（Microsoft公式）**
  - [https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
  
- **ASP.NET Core MVC 公式ガイド**
  - [https://learn.microsoft.com/ja-jp/aspnet/core/mvc/overview](https://learn.microsoft.com/ja-jp/aspnet/core/mvc/overview)