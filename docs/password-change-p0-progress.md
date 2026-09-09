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
| 対象テスト | 完了（17件成功） |
| Webビルド | 完了（環境制限を回避して成功） |
| `desktop-wallet` へのコミット | 完了（`45b92988`） |

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

## 継続作業（2026-09-09）

P0検証時に長時間化していた `FormPersistentDelegationRequestTransaction` の阻害要因を切り分け、対象テストと終了処理を安定化した。

### 実装・設計判断

- 対象テスト内の非同期UI操作、トースト確認、プロフィール解除、料金選択、マルチシグ選択を`await`し、操作完了後の画面状態を検証するようにした。
- 各テストで生成したストアを追跡し、DOMのアンマウント、アカウント・ネットワーク購読の解放、保留中Vuexアクションの排出、ストアの`uninitialize`をこの順序で実行するようにした。保留中アクションには実タイマーを含む待機を設け、終了時に残存アクション名を報告する。
- Jestで`LocalStorageBackend`をインメモリ実装へ差し替えているため、`localStorage`のキー削除だけでは不十分だった。Harvesting、Mosaic、Network currency、Network、Nodeの各シングルトンストレージを、テストで使用したネットワーク世代ハッシュ単位で消去するようにした。
- 未モックだった`/node/unlockedaccount`を追加し、ノード運用者テストは実際のNode監視サービスのメソッド呼出しで`B983...`から`05E5...`への対応を検証できる構成にした。
- アカウント情報が未初期化の場合のgetterを`null`へ正規化し、購読が登録されていない場合もlistenerを閉じるようにした。Harvestingの状態取得はネットワーク初期化前に終了し、Networkの購読解除完了を待つようにした。これらはテスト終了時の競合を防ぐ防御的変更である。

### レビューと対応

- 独立レビューで指摘された非同期操作、終了順序、キャッシュ実体、無効なリンク解除アサーション、ノード監視モック、getterのnull契約を反映した。
- 再レビューで、キャッシュ削除への世代ハッシュ引渡し、実ノード監視モック、非null待機、アカウント購読解放を確認し、重大な残指摘なしの承認を得た。

### 継続作業の検証

- `./node_modules/.bin/jest --runInBand __tests__/views/forms/FormPersistentDelegationRequestTransaction.spec.ts --testTimeout=30000 --silent`
  - 17件成功、終了コード0、197.253秒
- `./node_modules/.bin/jest --runInBand __tests__/services/PasswordChangeService.spec.ts --testTimeout=30000 --silent`
  - 16件成功、終了コード0、55.178秒
- `./node_modules/.bin/tsc --noEmit`
  - 成功
- `npm run eslint`
  - 終了コード0。既存の未使用変数警告5件のみ
- `./node_modules/.bin/prettier --check ./src ./__tests__ ./__mocks__`
  - 成功
- `git diff --check`
  - 成功
- `env NODE_OPTIONS=--max_old_space_size=3072 npm run build:web`
  - Node 16.20.2で成功。ビルド完了メッセージを確認。既知のBrowserslistおよびアセットサイズ警告のみ

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

- 全体Jestは未実行のため、全体の成功は未確認である。ただし、従来の阻害要因だった`FormPersistentDelegationRequestTransaction.spec.ts`は全17件を単独で完走し、今回のP0対象範囲では解消済みである。全体JestはWebSocket・MSW依存テストの実行時間を含め、PR環境で別途確認する。
- `canvas` のソースビルドは、Python 3.12およびCairo/Pango/Pixman開発ライブラリ不足では実行できない。Node 16向けプリビルドを取得できる環境では再構築に成功したため、クリーンなPR環境で同じ依存導入手順を確認する必要がある。
- JestのNode環境では、アプリ設定が参照する `navigator`、`window`、`localStorage` をテストセットアップで補い、`symbol-sdk` と `js-sha3` のcross-realm `ArrayBuffer`差異をJest専用アダプターで吸収した。本番暗号処理は変更していない。

## 次の作業

1. PR環境で全Jestを実行し、対象外テストを含む全体の成功を確認する。
2. 実ユーザーデータを使わない永続スナップショット復旧の再起動相当試験を追加・確認する。
3. PR資料としてこの記録を参照し、全体Jestと実環境確認の結果を更新する。

## リポジトリ状態（記録時点）

- 親リポジトリHEAD: `2562b3cf1547db6b205bc7cc0fa49e92a260c013`（この記録の更新前）
- `desktop-wallet` HEAD: `45b92988ed246dde6e2931bc43517ee08580e696`
- `desktop-wallet` の既存変更: `_symbol` の変更、`mise.toml` の未追跡ファイル。今回の実装では変更していない。
