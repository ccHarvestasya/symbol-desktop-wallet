---
name: delegated-workflow
description: Coordinate repository work by assigning task analysis to gpt-5.6-sol at low reasoning effort and implementation plus independent review to gpt-5.6-luna at xhigh reasoning effort. Use for non-trivial work in this repository that benefits from delegated execution and review; skip for simple questions or one-step read-only checks.
---

# 委任ワークフロー

このリポジトリでの非自明な作業を、次の役割に分けて進める。

1. 作業整理担当として `gpt-5.6-sol`、reasoning effort `low` のサブエージェントを起動し、対象範囲、実行手順、検証項目、注意点を簡潔に整理させる。
2. 整理結果を踏まえ、実作業担当として `gpt-5.6-luna`、reasoning effort `xhigh` のサブエージェントを起動する。コードや文書の変更、必要な検証を担当させる。
3. 実作業後、別の `gpt-5.6-luna`、reasoning effort `xhigh` のサブエージェントに独立レビューを依頼する。要件適合性、事実関係、回帰、安全性、テスト不足を確認させる。
4. レビュー指摘を自身で確認し、必要な修正を実作業担当へ戻すか、自身で最小限の修正を行う。重要な指摘が解消するまでレビューを繰り返す。

サブエージェントの起動時は、モデルとreasoning effortを確実に指定できるよう、必要最小限の会話だけをforkする。各担当には、ルートの `AGENTS.md` と対象ディレクトリに適用される指示を読むよう明示する。

単純な質問、短い状態確認、明白な一行修正では、このワークフローを自動適用しない。委任による確認価値が作業量を上回る場合に使用する。

このスキルは作業権限を拡張しない。外部への送信、破壊的操作、コミット、プッシュなどは、ユーザーの依頼と既存の承認範囲に従う。
