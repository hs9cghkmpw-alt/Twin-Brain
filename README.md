# Twin-Brain

「第二の脳」プロジェクト。

Obsidian に保存した自分の情報を、ローカル AI / embedding / 検索によって再利用できる仕組みを構築する。

## 現在の構成

- OS: Windows
- Knowledge base: Obsidian
- Vault: `C:\Users\user\Documents\第二の脳`
- Inbox: `C:\Users\user\Documents\第二の脳\00_Inbox`
- Embedding API: Ollama `http://localhost:11434/api/embed`
- LLM候補: `qwen2.5:1.5b-instruct`, `qwen2.5:7b-instruct`
- Embedding候補: `bge-m3:latest`, `nomic-embed-text:latest`
- Ollama: `0.33.3`

## 現在の状態

検索プロトタイプを検証中。embedding モデルの日本語検索性能について比較テストを実施している。

現時点では `bge-m3` の方が、実際の3ノート検索において `nomic-embed-text` より良い順位を返した。ただし、bge-m3にも日本語フレーズで異なる入力が完全に同一ベクトルになる現象が確認されており、最終採用モデルは未確定。

## 次の検証

1. 日本語・多言語embeddingモデル候補を追加比較
2. 同一質問・同一ノートセットでベンチマーク
3. 検索精度とTop-K順位を評価
4. 採用モデル決定
5. 検索スクリプトを最終整理
6. Qwenによる検索結果の回答生成を接続
7. セットアップ手順・トラブルシューティングを完成させる

## 開発記録

詳細な今回のセットアップ・診断・テスト結果は [`docs/embedding-diagnosis.md`](docs/embedding-diagnosis.md) を参照。
