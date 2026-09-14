# Embedding / 検索プロトタイプ診断記録

## 1. 目的

Obsidian に保存した第二の脳の情報を、embedding 検索で過去のメモから取り出せるか検証する。

今回の検証では、Ollama の embedding API を PowerShell から直接呼び出し、embedding の生成と検索順位を確認した。

## 2. 環境

- Ollama: `0.33.3`
- API: `http://localhost:11434/api/embed`
- Vault: `C:\Users\user\Documents\第二の脳`
- Inbox: `C:\Users\user\Documents\第二の脳\00_Inbox`
- Embedding model: `bge-m3:latest`
- 比較用 embedding model: `nomic-embed-text:latest`
- 新規比較 model: `nomic-embed-text-v2-moe`
- LLM: `qwen2.5:1.5b-instruct`, `qwen2.5:7b-instruct`

確認済みモデル一覧:

```text
nomic-embed-text:latest    0a109f422b47    274 MB
qwen2.5:1.5b-instruct      65ec06548149    986 MB
bge-m3:latest              790764642607    1.2 GB
qwen2.5:7b-instruct        845dbda0ea48    4.7 GB
```

追加インストール済み:

```text
nomic-embed-text-v2-moe    475.29M parameters    768 dimensions    context 512    F16
```

## 3. Obsidian テストノート

`00_Inbox` に以下の3ファイルを用意した。

- `第二の脳テスト.md`
- `釣りテスト.md`
- `睡眠テスト.md`

主な内容:

- 第二の脳: 情報保存、過去情報検索、AIによる読み取り、関連情報の接続
- 釣り: 北海道、小樽、大きな魚、サビキ、ウキ釣り
- 睡眠: 睡眠時間、朝の目覚め、睡眠の質、日中の疲れ

## 4. 最初に発見したPowerShellの問題

`Get-Content -Raw` で取得した値に PowerShell の provider NoteProperties が残っている状態があり、`ConvertTo-Json` すると `input` が文字列ではなくオブジェクトとして送信され、Ollama が以下を返した。

```text
{"error":"invalid input type"}
```

対策:

```powershell
$noteText = Get-Content "..." -Raw -Encoding UTF8
$noteText = [string]::Copy($noteText)
```

実際の検索処理でも同様に、ファイル本文を `[string]::Copy(...)` してから API に渡す。

## 5. bge-m3 の基本確認

`ollama show bge-m3` の結果:

```text
architecture        bert
parameters          566.70M
context length      8192
embedding length    1024
quantization        F16
Capabilities
  embedding
```

生成された Modelfile は以下の構成だった。

```text
FROM C:\Users\user\.ollama\models\blobs\sha256-daec91ffb5dd0c27411bd71f29932917c49cf529a641d0168496c3a501e3062c
TEMPLATE {{ .Prompt }}
```

## 6. bge-m3 embedding 異常の検証

異なる入力に対して、ベクトル全1024要素を比較した。

### 日本語の単語

```text
魚
睡眠
```

結果:

```text
1024 / 1024 要素が異なる
```

→ 単語レベルでは入力によって embedding が変化する。

### 日本語フレーズ

```text
魚を釣る
本を読む
```

結果:

```text
0 / 1024
```

→ 完全に同一ベクトル。

### 日本語の短文

```text
北海道で釣りをする
睡眠時間を記録する
```

結果:

```text
0 / 1024
```

### 長めの日本語文

```text
第二の脳で過去の情報を検索したい
最近、睡眠について記録している。
```

結果:

```text
0 / 1024
```

### 英語

```text
I like fishing.
I like sleeping.
```

結果:

```text
1024 / 1024
```

→ 英語では正常に異なる embedding が生成された。

### API の配列入力

`input` を1要素配列、2要素配列としても確認したが、日本語フレーズでは同一ベクトルになった。

Batch:

```text
魚を釣る
本を読む
```

結果:

```text
Embedding数: 2
次元数1: 1024
次元数2: 1024
Batch内で異なる要素数: 0 / 1024
```

### トークン数確認

完全一致の原因が単純な入力未認識ではないか確認するため `prompt_eval_count` を確認した。

```text
魚             -> 3
魚を釣る       -> 4
```

したがって、少なくとも入力が全く同一として扱われているという単純な説明ではない。

## 7. bge-m3 の実検索テスト

質問:

```text
北海道で大きな魚を釣りたい
```

実ファイルを段落単位で分割し、見出しを検索対象から除外した。

以前の bge-m3 結果:

```text
釣りテスト.md       0.9959  北海道で釣りをして、大きな魚を釣りたい。
睡眠テスト.md       0.9710  最近、睡眠について記録している。
第二の脳テスト.md   0.9625  これは第二の脳のテストです。
釣りテスト.md       0.8767  釣り関連箇条書き
睡眠テスト.md       0.8633  睡眠関連箇条書き
第二の脳テスト.md   0.7925  第二の脳関連箇条書き
第二の脳テスト.md   0.6408  Obsidian + AIで第二の脳...
```

→ 最上位は期待どおり釣りテストだった。ただし無関係な文とのスコア差が小さいため、絶対値による単純な閾値判定は採用しない。

## 8. nomic-embed-text 比較

`nomic-embed-text:latest` を追加インストールした。

embedding length:

```text
768
```

日本語フレーズ:

```text
魚を釣る
本を読む
```

結果:

```text
0 / 768
```

その後、同じ質問と同じ3ファイルで実検索した。

質問:

```text
北海道で大きな魚を釣りたい
```

結果:

```text
第二の脳テスト.md   0.9997  これは第二の脳のテストです。
睡眠テスト.md       0.9978  最近、睡眠について記録している。
釣りテスト.md       0.9921  北海道で釣りをして、大きな魚を釣りたい。
釣りテスト.md       0.8189  釣り関連箇条書き
睡眠テスト.md       0.8078  睡眠関連箇条書き
第二の脳テスト.md   0.7103  第二の脳関連箇条書き
第二の脳テスト.md   0.5079  Obsidian + AIで第二の脳...
```

→ このテストでは、釣りテストが3位になったため、日本語検索用途では bge-m3 の方が良い結果だった。

## 9. nomic-embed-text-v2-moe 比較

`nomic-embed-text-v2-moe` を追加インストールした。

`ollama show nomic-embed-text-v2-moe` の結果:

```text
architecture        nomic-bert-moe
parameters          475.29M
context length      512
embedding length    768
quantization        F16
Capabilities
  embedding
```

日本語フレーズのモデル単体 sanity test:

```text
魚を釣る
本を読む
```

結果:

```text
Embedding数: 2
次元数: 768
異なる要素数: 0 / 768
```

→ `nomic-embed-text-v2-moe` でも、この2つの日本語フレーズは完全に同一ベクトルになった。

→ この時点では `bge-m3` と `nomic-embed-text` だけでなく、`nomic-embed-text-v2-moe` でも同じ現象が再現している。したがって、特定の1モデルだけの異常と断定せず、モデル・推論環境・入力処理を切り分けながら追加検証する。

## 10. 現時点の判断

### 採用確定ではない

現時点では `bge-m3` を暫定的に優位とするが、最終採用は保留。

理由:

1. bge-m3 は実検索で釣りノートを1位にできた。
2. しかし日本語フレーズ間で完全に同一のembeddingが生成される異常な挙動がある。
3. nomic-embed-text も日本語フレーズで同一ベクトルとなり、今回の実検索では bge-m3 より悪かった。
4. nomic-embed-text-v2-moe も `魚を釣る` / `本を読む` で 0 / 768 の差分となった。
5. したがって、現象の原因を特定する前に特定モデルを採用確定しない。

## 11. 重要な設計判断

検索では、embedding の絶対スコアを固定閾値で判定しない。

初期テストでは、釣り質問に対して睡眠や第二の脳の文章にも高いスコアが付いたため、以下の構成を候補とする。

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

## 12. 今後のTODO

- [ ] 日本語・多言語向け embedding モデルを追加比較
- [ ] 同一質問セットでモデル別ベンチマーク
- [ ] Top-1 / Top-K の検索精度を記録
- [ ] 文書チャンク方式を確定
- [ ] embedding キャッシュ方式を検討
- [ ] Qwenを使った再ランキング / 回答生成を検証
- [ ] `search-brain.ps1` の最終版を確定
- [ ] `ask-brain.ps1` と検索機能を統合
- [ ] セットアップ手順を別ドキュメントとして整理
- [ ] 最終構成を再現可能な形にする

## 13. 注意

このドキュメントは 2026-09-14 時点の途中経過。モデル選定・検索方式はまだ確定していない。
