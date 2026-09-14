# Embedding / 検索プロトタイプ診断記録

## 2026-09-14 作業再開ポイント

### 現在の結論

`nomic-embed-text-v2-moe` が現時点の有力候補。ただし採用確定はしていない。

### v2-moe で確認済み

モデル情報:

```text
architecture        nomic-bert-moe
parameters          475.29M
context length      512
embedding length    768
quantization        F16
Capabilities
  embedding
```

Direct curl で以下を確認:

```text
魚を釣る / 本を読む
次元数: 768
異なる要素数: 768 / 768
```

日本語カテゴリ分離ベンチマーク:

```text
釣り質問  -> 釣り記録 0.930114 / 睡眠記録 0.081754 / 第二の脳 0.141354
睡眠質問  -> 釣り記録 0.172251 / 睡眠記録 0.642216 / 第二の脳 0.289056
第二の脳質問 -> 釣り記録 0.196696 / 睡眠記録 0.310678 / 第二の脳 0.421447
```

3カテゴリすべてで意図したカテゴリがTop-1になった。

### 実Obsidianデータ

Vault:

```text
C:\Users\user\Documents\第二の脳\00_Inbox
```

見出しを除外して実ファイルから7チャンクを取得できることを確認済み。

```text
[0] 睡眠テスト.md
    最近、睡眠について記録している。

[1] 睡眠テスト.md
    - 睡眠時間を記録する
    - 朝の目覚めを確認する
    - 睡眠の質について考える
    - 日中の疲れとの関係を記録する

[2] 第二の脳テスト.md
    これは第二の脳のテストです。

[3] 第二の脳テスト.md
    - 自分の情報を保存する
    - 過去の情報を検索する
    - AIに自分の情報を読ませる
    - 関連する情報をつなげる

[4] 第二の脳テスト.md
    Obsidian + AIで第二の脳を作れるか試している。

[5] 釣りテスト.md
    北海道で釣りをして、大きな魚を釣りたい。

[6] 釣りテスト.md
    - 小樽で釣りをする
    - 大きな魚を狙う
    - サビキやウキ釣りを試す
    - 釣れた魚を記録する
```

### 次回やること

**ここから再開する。**

質問:

```text
北海道で大きな魚を釣りたい
```

v2-moe で上記7チャンクを direct curl でembeddingし、cosine similarityでランキングする。

目的:

1. 実Obsidianデータで釣りチャンクがTop-1になるか確認
2. 釣り関連チャンクが上位にまとまるか確認
3. その結果を bge-m3 と比較
4. 問題なければ v2-moe を検索モデルの第一候補として固める

### 重要な注意

- `search-brain.ps1` はまだ変更しない。
- `ask-brain.ps1` も変更しない。
- モデル選定が確定するまで既存の動作を壊さない。
- bge-m3 の過去の日本語embedding異常については、必要なら direct curl 条件で再検証する。

## 過去の環境・検証記録

- Ollama `0.33.3`
- API `http://localhost:11434/api/embed`
- Embedding候補: `bge-m3:latest`, `nomic-embed-text:latest`, `nomic-embed-text-v2-moe`
- LLM: `qwen2.5:1.5b-instruct`, `qwen2.5:7b-instruct`
- テストノート: `第二の脳テスト.md`, `釣りテスト.md`, `睡眠テスト.md`
- PowerShellでは `Get-Content -Raw` の値を `[string]::Copy(...)` してAPIへ渡す。
- JSONをdirect curlで送る場合はBOMなしUTF-8でファイルを書き出す。

## 設計候補

```text
質問
  ↓
Embedding検索
  ↓
Top-K候補取得
  ↓
必要ならQwenで再評価 / 文脈統合
  ↓
回答
```

embeddingの絶対スコアを固定閾値だけで判定する方式は採用しない。

このドキュメントは2026-09-14時点の途中経過。