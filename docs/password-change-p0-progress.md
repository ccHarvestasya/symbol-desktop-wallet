# P0 パスワード変更整合性対応の進行記録

## 目的

パスワード変更時に、現在プロファイルに属する秘密情報を一括して新しいパスワードで再暗号化する。すべての変換が完了するまで保存を開始せず、保存途中の失敗では保存前の状態へ復元する。

P0では旧形式暗号文の厳格な形式検証や完全性検証を追加しない。`symbol-sdk` の現行 `Crypto.decrypt()` の返り値・例外挙動を維持し、厳格な拒否はP1以降の課題とする。

## 対応状況

| 項目 | 状態 |
| --- | --- |
| 対象範囲と保存構造の調査 | 完了 |
| Harvesting鍵4項目とアカウント内リモート鍵の再暗号化実装 | 完了 |
| 全件変換後に保存を開始する処理 | 完了 |
| 保存前スナップショットによる復元 | 完了 |
| 保存後の読戻し確認 | 完了 |
| 中断時の永続スナップショット復旧 | 完了（スキーマ版不一致時は安全側に停止） |
| P0の回帰テスト | 追加済み・対象テスト完走 |
| 型チェック | 通過 |
| 全体Lint・整形確認 | 通過（既存警告5件） |
| 対象テスト | 完了（16件成功） |
| Webビルド | 完了（環境制限を回避して成功） |
| `desktop-wallet` へのコミット | 完了（`fa057406`） |

## 実装方針

- `FormProfilePasswordUpdate` から変換・保存処理を `PasswordChangeService` へ分離する。
- 再暗号化対象は、プロファイルの `seed`、プロファイルが参照する通常アカウントの秘密鍵、アカウントモデルに残る `encRemoteAccountPrivateKey`、Harvestingモデルの次の4項目とする。
  - `encRemotePrivateKey`
  - `newEncRemotePrivateKey`
  - `encVrfPrivateKey`
  - `newEncVrfPrivateKey`
- Harvestingモデルは全プロファイル共通の配列であるため、現在プロファイルのアカウントアドレスに一致するモデルだけを処理する。
- 変換中は保存済みオブジェクトを変更せず、複製したコレクションを作る。
- 変換完了後、`profiles`、`accounts`、`harvestingModels` をコレクション単位で保存する。
- 保存前の3コレクションを暗号文の状態のままスナップショットとして保持し、保存失敗・読戻し不一致時に復元する。
- プロセス中断に備え、保存完了確認まで永続スナップショットを残し、次回アプリケーション初期化時に復元する。
- 永続スナップショットには各コレクションのスキーマ版を記録し、起動時に版が一致しない場合は旧形式データを現行形式として保存せず停止する。
- 永続スナップショットの書込み失敗時は、完全な復旧用記録を残すか、記録を削除してから処理を失敗させる。
- 空文字、`null`、`undefined` のHarvesting項目は鍵がない状態としてそのまま保持する。
- 旧形式暗号文の新たな形式判定、認証タグ検証、秘密鍵の妥当性検証はP0へ追加しない。

## レビューでの修正事項

1. 初回レビューで、復元失敗を握り潰していたため、復元後の読戻しを確認し、復元不能時は安全側のエラーにするよう修正した。
2. 保存データに存在しない参照アカウントを黙って処理対象から外さず、保存前に失敗させるよう修正した。
3. 同一Harvestingアドレスが複数プロファイルから参照される場合、所有者を区別できないため保存前に失敗させるよう修正した。
4. 保存後の読戻し不一致を検出し、成功通知・ログアウト・画面遷移へ進まないよう修正した。
5. プロセス中断後も旧データへ戻せるよう、永続スナップショットと起動時復旧を追加した。
6. アカウントモデルに残る `encRemoteAccountPrivateKey` も再暗号化対象へ追加した。
7. スナップショットの部分書込みに対する後始末、モデル必須項目、レコードキー、参照関係、スキーマ版の検証を追加した。

## 検証記録

### 成功

- `npm exec -- tsc --noEmit --pretty false`
- 対象テストおよびテスト補助ファイルのESLint
- 対象ファイルのPrettier確認
- `npm run lint`（既存の警告5件を含むが、終了コード0）
- `git diff --check`
- `./node_modules/.bin/jest --runInBand __tests__/services/PasswordChangeService.spec.ts --testTimeout=30000 --no-cache`
  - `PasswordChangeService` 16件成功
- `./node_modules/.bin/jest --runInBand __tests__/components/AppLogo.spec.ts --testTimeout=30000 --no-cache`
  - 1件成功
- `npm run build:web`（Node 16、制限外実行、Browserslist警告のみ）
- `canvas@2.8.0` のNode 16向けプリビルド再構築（`mise`でNode 16・Python 3.10を指定）

### 未完了・阻害要因

- 全Jestは、`canvas.node` の再構築後に起動できることを確認した。ただし、全体実行では既存のWebSocket・MSW依存テストが長時間化し、`FormPersistentDelegationRequestTransaction.spec.ts` の失敗出力を確認した後、上限時間内の完了を優先して手動停止した。全体の成功は未確認であり、今回のP0対象外として切り分ける。
- `canvas` のソースビルドは、Python 3.12およびCairo/Pango/Pixman開発ライブラリ不足では実行できない。Node 16向けプリビルドを取得できる環境では再構築に成功したため、クリーンなPR環境で同じ依存導入手順を確認する必要がある。
- JestのNode環境では、アプリ設定が参照する `navigator`、`window`、`localStorage` をテストセットアップで補い、`symbol-sdk` と `js-sha3` のcross-realm `ArrayBuffer`差異をJest専用アダプターで吸収した。本番暗号処理は変更していない。

## 次の作業

1. PR環境で全Jestを実行し、`FormPersistentDelegationRequestTransaction.spec.ts` の既存失敗原因を切り分ける。
2. 実ユーザーデータを使わない永続スナップショット復旧の再起動相当試験を追加・確認する。
3. PR資料としてこの記録を参照し、全体Jestと実環境確認の結果を更新する。

## リポジトリ状態（記録時点）

- 親リポジトリHEAD: `4e99ad5d2374a08f2c5239c610f0507470b591c1`（この記録のコミット前）
- `desktop-wallet` HEAD: `fa05740683a98f7cfd117ad3806abc7d92d2957a`
- `desktop-wallet` の既存変更: `_symbol` の変更、`mise.toml` の未追跡ファイル。今回の実装では変更していない。
