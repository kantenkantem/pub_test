# Entity設計・DB定義 ケアレスミス＆Gotchas一覧 (6月10日版)

チーム開発演習におけるデータベース定義および Entity クラス（作成済み）を精査したところ、実装時にバグや動作不良を引き起こしやすい**スペルミス**や**データ型の不整合**がいくつか確認されました。

メンバーA（テクニカルリーダー）として、開発中や他のメンバーのトラブルシューティング時に、本ドキュメントをチェックリストとして活用してください。

---

## 1. Plan Entity におけるカラム名のスペルミス

`Plan`（プラン）エンティティにおいて、カラム物理名（`@Column` で指定された名前）にタイポがあります。Java のフィールド名や Getter/Setter は正しい英単語ですが、アノテーション内は DB 定義に合わせる必要があります。

| Javaフィールド名 | アノテーション内のカラム指定 | 注意点（スペルミス箇所） |
| :--- | :--- | :--- |
| `taxExcludePrice` | `@Column(name = "tax_excluce_price")` | `exclude` の d が **`c`** になっている |
| `taxIncludePrice` | `@Column(name = "tax_incluce_price")` | `include` の d が **`c`** になっている |
| `isArchieved` | `@Column(name = "is_archieved")` | `archived` の h の後に **`e`** が入っている |

### 💻 Java実装時のコード例
アノテーションを記述する際は、以下のようにテーブル物理名に完全一致させてください。
```java
@Column(name = "tax_excluce_price")
private Integer taxExcludePrice;

@Column(name = "tax_incluce_price")
private Integer taxIncludePrice;

@Column(name = "is_archieved")
private Boolean isArchieved;
```

---

## 2. 郵便番号（Postal Code）の型不整合 (重要)

会員情報と宿情報で、郵便番号のデータ型が異なっています。

* **会員側 (`User.java`)**: `private Integer postalCode;` （数値型）
* **宿側 (`Accommodation.java`)**: `private String postalCode;` （文字列型）

### ⚠️ 発生する問題
1. **先頭の「0」消失バグ**: 数値型で管理すると、北海道などの郵便番号（例: `0600001`）を保存した際、先頭の `0` が消えて数値 `600001` に変換されてしまいます。
2. **バインドエラー**: 画面（HTML）から文字列として受け取り、そのまま `User` オブジェクトにバインドしようとすると、型ミスマッチ（`MethodArgumentTypeMismatchException` / 400エラー）が発生します。

### 🛠️ 対策・回避コード
会員登録処理コントローラーでは、引数を `Integer` で受け取るか、文字列で受けてからパースしてください。また、HTML側では数値入力制限を入れる等の対策を推奨します。
```java
// UserController.java での受信例
@PostMapping("/register")
public String register(
        @RequestParam("postalCode") Integer postalCode, // 最初からInteger型でバインドする
        ...
) {
    User user = new User();
    user.setPostalCode(postalCode);
    ...
}
```

---

## 3. 都道府県・カテゴリマスタの名称フィールド名

都道府県（`Prefecture`）および宿カテゴリ（`AcmCategory`）の名称を保持するフィールドです。

* **都道府県 (`Prefecture.java`)**: `private String name;`
* **宿カテゴリ (`AcmCategory.java`)**: `private String name;`

### ⚠️ 発生する問題
画面遷移図やテーブル定義の日本語名から連想して、Thymeleaf 上で以下のように書いてしまうメンバーが多発します。
* ❌ `th:text="${pref.prefectureName}"`
* ❌ `th:text="${category.categoryName}"`
この場合、画面表示時に **500 エラー（`PropertyNotFoundException`）** になります。

### 🛠️ 対策・回避コード
必ず Entity の定義である **`name`** を指定してください。
```html
<!-- 正しい表記 -->
<option th:each="pref : ${prefectures}" th:value="${pref.id}" th:text="${pref.name}"></option>
```

---

## 4. `PlanStock`（在庫）におけるリレーション定義ミス

通常、外部キーを持つテーブルはエンティティクラス間でオブジェクトとして結合（`@ManyToOne` 等）を定義しますが、`PlanStock` のみ定義が変則的になっています。

* **`PlanStock.java`**:
  ```java
  @JoinColumn(name = "plan_id")
  private Integer planId; // Planオブジェクトではなく、単なるInteger型になっている
  ```

### ⚠️ 発生する問題
JPAの自動マッピング機能で `PlanStock` から直接 `Plan` オブジェクトを取得（例: `stock.getPlan().getName()`）することができません。

### 🛠️ 対策・回避コード
在庫を新規作成する際や、在庫を検索する際は、オブジェクトではなく **`Integer` 型のプランID** を直接セット・利用してください。
```java
// PlanService.java での在庫登録の記述例
PlanStock stock = new PlanStock();
stock.setPlanId(savedPlan.getId()); // savedPlan.getId() の整数値を直接セットする
stock.setTargetDate(targetDate);
stock.setStock(initialStock);
planStockRepository.save(stock);
```

---

## 5. `Accommodation`（宿）のカテゴリ結合カラム名

宿とカテゴリの結合（ManyToOne）定義です。

* **宿 (`Accommodation.java`)**:
  ```java
  @ManyToOne
  @JoinColumn(name = "type_of_acm")
  private AcmCategory acmCategory;
  ```

### ⚠️ 注意点
他が `prefecture_id` や `user_id` と命名されているのに対し、ここだけ **`type_of_acm`** というカラム名になっています。
* JPAが内部的に外部キーとして処理するため、基本的には意識せずとも結合オブジェクトとして取得可能ですが、手動でSQL文を書く場合や、DB定義を直接デバッグする際はこの物理名に留意してください。
