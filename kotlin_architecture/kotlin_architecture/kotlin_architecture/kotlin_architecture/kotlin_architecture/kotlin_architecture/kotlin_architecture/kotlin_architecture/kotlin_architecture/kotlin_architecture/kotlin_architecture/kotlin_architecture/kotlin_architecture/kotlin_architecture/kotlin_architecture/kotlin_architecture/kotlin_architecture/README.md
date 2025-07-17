# 生成AIが保守しやすいサーバー実装 ― 基本設計ガイド（Model / UseCase / Layout 版）

* **package-by-feature × Unbalanced Tree 多層構造**  
* **1 ファイル = 150 行以内**  
* **デメテルの法則（LoD）厳守でコンテキスト最小化**  
* **すべてのディレクトリに `README.md` 必須**  
* **各ディレクトリは `Model.kt`・`UseCase.kt`・`Layout.kt` のみを持つ**  
  *下位層ほど “読むだけ” を徹底。動的プロパティ (`var`) は最上位層のみ許可*

---

## 1. ディレクトリ階層例

```text
src/
 ├── common/
 │   ├── Model.kt
 │   ├── UseCase.kt
 │   ├── Layout.kt
 │   └── README.md
 ├── user/                   # ≪ユーザー機能≫（中規模）
 │   ├── Model.kt
 │   ├── UseCase.kt
 │   ├── Layout.kt
 │   └── README.md
 ├── health/                 # ≪死活監視≫（小規模）
 │   ├── Model.kt
 │   ├── UseCase.kt
 │   ├── Layout.kt
 │   └── README.md
 ├── payment/                # ≪決済機能≫（大規模）
 │   ├── Model.kt
 │   ├── UseCase.kt
 │   ├── Layout.kt
 │   ├── refund/             # サブ機能（Unbalanced 深掘り）
 │   │   ├── Model.kt
 │   │   ├── UseCase.kt
 │   │   ├── Layout.kt
 │   │   └── README.md
 │   └── README.md
 └── README.md               # ルート概要
```

> **階層ポリシー**  
> - **上位層 → 下位層** への一方向利用のみ許可  
> - **Model → UseCase → Layout** の依存は *隣接層* のみ（深い呼び出し禁止）  
> - 下位層 (`health/` など) のファイルは **再代入不可 (`val`/`const`) とラムダ** のみを保持  

---

## 2. README.md テンプレート（全ディレクトリ共通）

```md
# <ディレクトリ名>

## 役割
<機能概要を 2〜3 行で説明>

## ファイル一覧
| ファイル | 責務 | LoD チェックポイント |
| -------- | ---- | -------------------- |
| Model.kt  | 不変データ / 値オブジェクト | - 外部の深い構造にアクセスしない |
| UseCase.kt| ユースケース / 処理フロー | - Model へのみ依存<br>- 動的プロパティ禁止 (`val` のみ) |
| Layout.kt | I/O レイヤ (Controller/UI) | - UseCase へのみ依存<br>- a.b.c チェーン禁止 |

## 公開インターフェース
| 種別 | エンドポイント / エントリ | 概要 |
| ---- | ----------------------- | ---- |
| HTTP | GET /users/{id}         | ユーザー詳細取得 |
| FUNC | UserUseCase.exec()      | アプリケーションサービス |

## 依存関係
- depends-on: `common`
- no-depends: `<下層>.*/Layout.kt` 以外

## 動的プロパティ規約
- **最上位層**（`src/` 直下）の `Model.kt` のみ `var` を許可  
- それ以外は **再代入禁止 (`val`)** + **ラムダ** で状態遷移を表現

## 生成AI へのヒント
1. この README が最上位の要約  
2. 1 ファイル 150 行 / 1 関数 30 行以内  
3. ディレクトリ外部へは公開 API だけを公開  
4. テストは `example/*.spec.kt` に配置
```

---

## 3. ファイル役割サマリ

| ファイル | 型／構造                     | 主な内容                          | ルール |
| -------- | --------------------------- | --------------------------------- | ------ |
| **Model.kt**  | `data class` / value class      | 不変エンティティ、DTO、Enum        | - `val`/`const` のみ<br>- 外部ライブラリ呼び出し禁止 |
| **UseCase.kt**| `class` / `object`             | ユースケース、CQRS、バリデーション | - Model にのみ依存<br>- 副作用はラムダ注入 |
| **Layout.kt** | `class` / controller / endpoint | I/O バインド (HTTP/GraphQL/UI)     | - UseCase にのみ依存<br>- 表示ロジックは ViewModel 等へ委譲 |

---

## 4. 実装 Tips

1. **ディレクトリ＝境界**。別 feature への参照は `common/` 経由か HTTP/Message 経由のみ  
2. **値オブジェクトでカプセル化** → LLM のコンテキスト縮小 & LoD 遵守  
3. **最上位層でのみ mutable**、他層は pure function 構成で “事故” を防止  
4. **README 生成をテンプレ化** → PR 時に漏れを CI で検出  

---

必要に応じて、ここから言語／フレームワーク固有の実装サンプルや CI 設定を拡張してください。
