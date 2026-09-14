# 旧 Brain Twin 2 からの移行記録

この文書は、旧リポジトリ `hs9cghkmpw-alt/-` の `brain-twin-2/` で積み上げた設計・評価・検証結果のうち、現在の `Twin-Brain` でも保持すべき履歴を移行したもの。

旧リポジトリを今後の作業先として使うのではなく、**歴史的な設計判断・評価ゲート・検証結果の根拠として参照するための記録**である。

## 1. 移行元と移行先

- 移行元: `hs9cghkmpw-alt/-` / `brain-twin-2/`
- 旧開発ブランチ: `brain-twin-dev`
- 移行先: `hs9cghkmpw-alt/Twin-Brain`
- 現行開発基準: `Twin-Brain`
- 2026-09-14 時点の現行作業到達点: `32c3318afbf187d3608e5f304303ec5aa7ea8f6b`

## 2. 旧プロジェクトで確立した基本設計

- Markdown / Obsidian Vault = 永続的な Memory Source of Truth
- SQLite / embedding cache / ANN sidecar = 再構築可能な派生状態
- 生の入力は保持し、AI整理で破壊的に置換しない
- Organizer LLM、embedding provider、reranker、ANN backend は交換可能な構成にする
- 通常の `reindex` は provider / network に依存しない
- テスト・評価fixtureは実ユーザーVaultに触れない
- production code は `brain_twin_eval` に依存しない
- 評価完了と Production Vector Search activation は別ゲートとして扱う

## 3. 旧フェーズの到達点

旧 `CURRENT_STATE.md` では以下を完了扱いとしていた。

- Phase 1 — Memory Foundation: COMPLETE
- Phase 2 — Automatic Memory Worker / Entity / Link generation: COMPLETE
- Phase 3 — Retrieval: COMPLETE
- Phase 4 — Vector Retrieval Core (4A–4D): GO / COMPLETE
- Production Vector Search activation: PENDING

PA1（日本語retrieval / model acceptance）は、実モデル選定やproduction activationとは分離した証拠ゲートとして設計されていた。

## 4. PA1で確立した評価原則

### Open benchmark

- 360 synthetic Memories
- 120 queries
- 80 dev / 40 blind-labelled pipeline-test queries
- semantic / paraphrase / spelling-transliteration / proper-noun / mixed JP-EN / hard-negative / short-query / long-Memory slices

重要: repository内の `blind` label は正式なheld-out evidenceではない。正式なheld-out corpusはtuning workspaceの外に置く。

### Formal blind

正式なblind評価では、以下を分離・固定する設計を採用した。

- held-out / public package separation
- judge comparison / adjudication
- frozen retrieval-config SHA
- Launch Envelopeによるdataset / policy / config / evaluator / runtime identity binding
- clean exact Git HEAD verification
- private scoring
- redacted final acceptance evidence
- critical-slice aggregate gates

正式なheld-out corpusによるblind runは、旧 `CURRENT_STATE.md` の時点では未実施だった。

## 5. 旧評価で修正した重大な証拠整合性問題

### Formal Blind predicate

Formal Blind の判定は、held-out judgement と `split == "blind"` の組み合わせを要求する単一のauthoritative predicateに統一した。

### Ranking drift / reproducibility

warm ranking drift が 0 でない場合、cold metricsを捨てず診断情報として保持しつつ、型付き状態として

- `reproducible=false`
- `selection_eligible=false`

を伝播させ、matrix selection、paired metric comparison、ANN comparison、critical slices、formal acceptance、blind scoringでfail closedする設計にした。

### Open matrixへのblind / held-out evidence混入防止

最終修正 `a2af959f84b3ca7853f11c3f5e6c64b05c61a412` では、open-development matrix summarizerが各入力について

- `judgement_visibility == "open"`
- `split == "dev"`

を要求するようにした。条件を満たさない証拠はentry生成・winner selectionより前に拒否する。

この修正の exact-SHA GitHub Actions run:

- Run `33850375512`
- Result: SUCCESS
- Tests: 617 passed

## 6. 旧Windows実機検証

PC134（Windows 11 Pro 64-bit）で、最終修正SHA `a2af959f84b3ca7853f11c3f5e6c64b05c61a412` をpullしてフルテストを実施。

- 616 passed
- 1 skipped
- 151.81s
- ExitCode 0

CIと実機Windowsの結果を分離して記録する運用も維持する。

## 7. 独立レビューの最終判定

2026-09-06、最終修正SHAについて独立レビューを実施。

- Critical: 0
- Major: 0
- Minor: 1
- Verdict: GO

レビューで確認された重要事項:

- 以前の「open matrixがheld-out / blind evidenceを誤ってwinner selectionへ取り込む」Majorは解消
- Formal Blind readinessに関するMajorは解消
- ranking drift / reproducibility / selection_eligibleのMajorは解消
- value-object construction boundaryでもfail-closedが機能
- adversarial negative testsが実際に失敗を拒否することを確認
- production `brain_twin/`、real Vault、production embedding configuration、PA2/PA3/PA4への変更はなし

Minor finding:

`summarize_payloads()` のtop-level scope / identityと、nested manifestの `dataset_judgement_visibility` / `dataset_sha256` / `dataset_version` の直接クロスチェックは追加のdefense-in-depthとして改善余地がある。ただし、通常生成される正規reportに対するbypassではなく、当時のGO判定を阻害するものではなかった。

## 8. 旧評価から現在へ引き継ぐアーキテクチャ候補

旧設計では次をpreferred targetとしていた。

- lexical recall: SQLite FTS / BM25
- semantic recall: Qwen3-Embedding-0.6B（evidence gate付き）
- post-retrieval relevance: Qwen3-Reranker-0.6B（OFF / ON比較）
- associative recall: Entity / Link one-hop expansion
- large-Vault ANN: FAISS HNSW（PA3 Windows / recovery gate付き）
- automatic organization: schema-constrained local instruction-following LLM

ただし、これは**候補・設計方針であり、現行の `nomic-embed-text-v2-moe` の実験結果より優先する決定ではない**。

## 9. 現行 Twin-Brain との関係

現在は、旧PA1評価基盤をそのままproduction仕様として扱うのではなく、そこで得た評価原則を現行の実Obsidian検索プロトタイプへ引き継ぐ。

現行 `Twin-Brain` では2026-09-14時点で `nomic-embed-text-v2-moe` を有力候補として実測中。

実Obsidianから7チャンクを取得済みで、次の実験は

> 「北海道で大きな魚を釣りたい」

をqueryとして、7チャンクをv2-moeでembeddingしcosine similarityでランキングすること。

この実験結果を、旧評価で確立した以下の原則に沿って記録する。

1. 同一query / 同一候補集合で比較する
2. Top-K順位を保存する
3. 実データ結果とsynthetic benchmark結果を混同しない
4. モデル採用確定前に `search-brain.ps1` / `ask-brain.ps1` を変更しない
5. absolute similarityの固定閾値だけで採否を決めない
6. 実機で確認した結果、CIで確認した結果、推論上の判断を分離する

## 10. 旧リポジトリの位置づけ

旧 `hs9cghkmpw-alt/-` の `brain-twin-2/` は、今後の通常作業先ではない。

必要になった場合のみ、以下のような歴史的根拠を確認するために参照する。

- PA1評価設計
- Formal Blind設計
- evidence-integrity修正
- Windows実機検証
- 独立レビュー
- production activation前のゲート設計

**現在の実装・実験・作業記録は `hs9cghkmpw-alt/Twin-Brain` に集約する。**

---

移行元の主要根拠: 旧 `brain-twin-2/docs/CURRENT_STATE.md`、PA1関連docs、Issue #2、独立レビュー、および各exact-SHA CI / Windows実機記録。
