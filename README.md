# Apex Map Rotation (Ranked)

Apex Legendsの**ランクマップ**ローテーションを確認できるWebツールです。
現在のマップや残り時間、今後の予測スケジュールをブラウザ上で簡単に確認できます。

## 🌟 特徴 (Features)

* **リアルタイム表示**: 現在プレイ可能なランクマップ、残り時間、進行状況（プログレスバー）をひと目で確認できます。
* **未来予測スケジュール**: マップローテーションが4時間30分ごとに切り替わるという前提のもと、今後のマップを予測してタイムライン形式で表示します。
* **日付選択・カレンダー機能**: タブやカレンダーから、特定の日付のスケジュールを確認可能です。
* **マップ絞り込み（フィルター）**: 特定のマップの予定だけを抽出して表示できます。
* **レスポンシブ対応**: PCだけでなく、スマートフォンからも快適に閲覧できるデザインです。

## 🔌 データソース (Data Source)

以前はGoogle Apps Script経由で手動メンテナンスしていたスプレッドシートを参照していましたが、
EA側の都合でマップローテーションが頻繁に変更され追従しきれなくなったため、
[tas.gg](https://tas.gg)（Twitch Apex Stats）が配信者向けに公開しているチャットコマンドAPI
`https://cmds.tas.gg/{twitchID}/maps` から「現在のランクマップ」「残り時間」「次のランクマップ名」を取得する方式に変更しました。

このAPIはプレーンテキストを返す形式（例:
`= 現在のマップスケジュール = | BRランク: E-District (残り38 分) - 次のマップ: ワールズエッジ | (- 運営元：TAS.gg -)`）
のため、GitHub Actions側でこのテキストを解析し、`{currentMap, nextMap, remainingMinutesAtFetch, fetchedAt}`という
構造化JSONに変換して保存しています。

その先の未来のマップ名はAPI側からは提供されないため、[data/mapcycle.json](data/mapcycle.json)に定義した
ランクマップの巡回順（`cycle`）とローテーション間隔（`rotationMinutes`、既定270分=4時間30分）を
組み合わせて予測生成しています。時刻は毎回APIから取得するため手動更新は不要ですが、
**シーズン切り替わりなどでEA側がランクマップのプールや巡回順、間隔を変更した場合のみ**、
`data/mapcycle.json`をGitHub上で直接編集してください（コードを触る必要はありません）。

```json
{
  "cycle": ["E-District", "ワールズエッジ", "ストームポイント"],
  "rotationMinutes": 270
}
```

tas.ggは`lang=jp`でもマップ名を必ずしも全て日本語化しないため（例: "ワールズエッジ"は日本語化されるが
"E-District"は英語のまま）、`cycle`には`data/maprotation.json`の`currentMap`/`nextMap`に
実際に入っている文字列をそのままコピーしてください。一致しない場合は自動的に「現在」「次」のみの
表示にフォールバックし、画面上に警告が表示されます。`data/mapcycle.json`自体の取得に失敗した場合は
`index.html`内のフォールバック値（現行のマッププール）で動作します。

### アーキテクチャ（twitchIDの秘匿）

このサイトはGitHub Pagesで公開する静的サイトのため、`index.html`（ブラウザ側のJS）に
twitchIDを直接書くとソースを見た誰でも分かってしまいます。そのため、tas.ggへのアクセスは
ブラウザからではなく [.github/workflows/update-map-rotation.yml](.github/workflows/update-map-rotation.yml) の
GitHub Actionsが15分おきに行い、解析結果を`data/maprotation.json`としてリポジトリにコミットします
（マップに変化がない場合はコミットしません）。`index.html`はこの静的JSONを読むだけなので、
ブラウザ側にtwitchIDは一切現れません。

```
GitHub Actions (15分おき, secrets.TAS_TWITCH_IDで認証)
  → https://cmds.tas.gg/{twitchID}/maps?br_ranked&shownext&lang=jp を取得
  → テキストを解析し、current/nextに変化があれば data/maprotation.json をコミット

index.html（ブラウザ）
  → data/maprotation.json を fetch するだけ（twitchID不要）
```

### 利用準備

1. https://tas.gg のダッシュボードでTwitchアカウントを連携し、自分のtwitchIDを確認する
2. GitHubリポジトリの Settings → Secrets and variables → Actions で、
   `TAS_TWITCH_ID` という名前のRepository secretを作成し、確認したIDを設定する
3. Actionsタブから「Update Map Rotation」ワークフローを一度手動実行（workflow_dispatch）し、
   `data/maprotation.json` を初期生成する

## 💻 使用技術 (Tech Stack)

* HTML5
* CSS3 (Custom Properties, Grid, Flexbox)
* Vanilla JavaScript
* [tas.gg](https://tas.gg) (Twitch Apex Stats) のマップローテーションコマンド
* ※外部ライブラリやフレームワーク（React, Vueなど）は使用していません。
