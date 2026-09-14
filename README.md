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
- Embedding候補: `bge-m3:latest`, `nomic-embed-text:latest`, `nomic-embed-text-v2-moe`
- Ollama: `0.33.3`

## 現在の状態

検索プロトタイプを検証中。2026-09-14時点では `nomic-embed-text-v2-moe` が有力候補だが、採用は未確定。

実Obsidianデータから7チャンクを取得済み。次の検証は「北海道で大きな魚を釣りたい」をqueryとして、7チャンクをv2-moeでランキングし、釣り関連チャンクが適切に上位へ来るか確認する。

`search-brain.ps1` / `ask-brain.ps1` はモデル選定が確定するまで変更しない。

## 旧 Brain Twin 2 から引き継いだ設計原則

- Markdown / Obsidian Vaultを永続的なMemory Source of Truthとする。
- SQLite / embedding cache / ANN sidecarは再構築可能な派生状態とする。
- 生の入力を保持し、AI整理で破壊的に置換しない。
- Organizer LLM / embedding / reranker / ANN backendは交換可能にする。
- 実データ、synthetic benchmark、CI、Windows実機の証拠を混同しない。
- Formal Blind / held-out evidenceはopen-development evidenceと分離する。
- 評価完了とProduction Vector Search activationを別ゲートとして扱う。
- embeddingのabsolute similarityだけを固定閾値にして採否を決めない。

旧PA1評価基盤では、evidence-integrity修正についてCritical=0 / Major=0の独立レビューと、exact-SHA CI 617 passed、Windows実機 616 passed / 1 skipped が確認された。詳細は [`docs/legacy-brain-twin-2-history.md`](docs/legacy-brain-twin-2-history.md)。

## 次の検証

1. v2-moeで実Obsidian 7チャンクをランキング
2. 同条件で過去候補（特にbge-m3）と比較
3. Top-K順位と検索品質を記録
4. 必要な追加候補を同一条件で比較
5. 採用モデル決定
6. 採用確定後に検索スクリプトを整理
7. Qwenによる検索結果の回答生成を接続
8. セットアップ手順・トラブルシューティングを完成させる

## 開発記録

- 最新のembedding / 実Obsidian検証: [`docs/embedding-diagnosis.md`](docs/embedding-diagnosis.md)
- 旧Brain Twin 2からの設計・評価・検証履歴: [`docs/legacy-brain-twin-2-history.md`](docs/legacy-brain-twin-2-history.md)
