# CLAUDE.md — 時間簿プロジェクト（Web版 / v1.2）
最終更新：2026-06-29

## このプロジェクトについて
「時間簿（Jikanbo）」は、お金の家計簿を**時間**に応用した個人用Webアプリ。
固定費（睡眠・経常業務など）を把握して可処分時間を可視化し、余剰時間に年間目標やマイルストーンを割り当て、
「朝に開けば今日が決まる」状態を作る。さらに使った時間を振り返り、見積もりのズレから学習する。

**作業前に必ず `jikanbo-requirements-v1.2.md`（要件定義書）を読むこと。** これが仕様の正本。

## 3つのファイルの役割（重要）
- `jikanbo.html` … **ロジックの正本**。収支計算・繰り返し処理・ICS書き出し・localStorage永続化など、動く実装の母体。
- `jikanbo-clipboard-mock.html` … **UI/UXの目標形**。クリップボード型（固定盤面＋文脈ページャ）の見た目・操作感の見本。
- `jikanbo-requirements-v1.2.md` … **機能仕様**。何を作るかの正本。

**ゴール**：`jikanbo.html` の動くエンジンを土台に、UIを `jikanbo-clipboard-mock.html` の方向（クリップボード型）へ寄せながら、要件定義書 v1.2 の機能を足していく。ゼロから作り直すのではなく、既存資産を育てる。

## 現在の実装状態（2026-06-29時点）

### ✅ 実装済み
- **今日ビュー**：当日の予定一覧・余剰時間・朝サマリー（コピー／mailto）。今週切替（Mon-Sun）
- **共有ICS**：公開レベル3段階（公開/多忙/非公開）、自分用・共有用2モード書き出し
- **分析タブ**：カテゴリ別・象限別・タグ別・文脈別・週次トレンド（8週積み上げ棒）
- **見積もり精度タブ**：実績入力・タグ別見積÷実績・補正係数
- **C警告 月別閾値設定**（カレンダータブ内で1〜12月各月に設定可）
- **ホームタブ**：クリップボード型UI（Phase 1）。クリップ金具・文脈ページャ（state.contexts 正本）・スワイプ対応・今週＋年間締切の2カラム・文脈管理ダイアログ（追加/編集/削除）
- **繰り返しパターン拡張**：隔週・毎月第N曜日・N ヶ月ごと・毎年。メモ欄・手順書（チェックリスト）も追加
- **優先度マトリクス ABCD 表記**：全タブ統一（A=重要で緊急、B=重要・計画、C=緊急・非重要、D=どちらでもない）
- **4象限 文脈フィルタ**：タブ上部のピルボタンで文脈ごとに絞り込み。カードに「社内」「顧客名」バッジ表示
- **カレンダー日付タップ → クイック追加ダイアログ**
- **月間タブ廃止・カレンダー統合**：時間収支バー・稼働時間設定・C警告グリッドをカレンダータブ内に統合
- **クラウド同期**：GitHub Gist API 経由でPCとモバイルのデータ共有（書き出しタブ）

### ❌ 未実装（次の主要課題）
- **クリップボード型UI Phase 2**：既存タブの詳細ビューをクリップボード内ドリルダウンに統合
- **R02 季節別稼働時間設定**：C警告の月別設定とは別に、月ごとの稼働時間（収入）を変える機能
- **R23 繁忙期・閑散期のカレンダー視覚化**
- **items への contextId 対応**：state.items は現在 customerType/customerName で文脈を代替。正式に contextId を付けてフォームで選択できるようにする

### STORE_KEY
現行は `"jikanbo.all.v10"`。スキーマ変更時は `load()` に移行処理を追加してバージョンを上げる。

## 現在のタブ構成

```
ホーム → 今日 → 予定 → カレンダー → 年間 → 4象限 → 分析 → 見積もり → 書き出し
```

（月間タブは廃止。カレンダータブ内に統合済み）

| タブ | 主な内容 | 主な関数 |
|---|---|---|
| ホーム | クリップボード型UI・文脈ページャ・カレンダー・今週/年間・収支・ABCD象限ミニ | `renderHome()` / `renderQuadCtxFilter()` |
| 今日 | 当日予定・余剰時間・朝サマリー（今週切替あり） | `renderToday()` / `renderWeek()` |
| 予定 | 追加フォーム・予定一覧・タグ管理 | `renderSchedule()` |
| カレンダー | 月次カレンダー・D&D・時間収支バー・稼働時間・C警告設定 | `renderCalendar()` / `renderBudget()` / `renderQ3Monthly()` |
| 年間 | ガントチャート・マイルストーン | `renderAnnual()` |
| 4象限 | アイゼンハワーマトリクス・文脈フィルタ・カードD&D | `renderQuad()` / `renderQuadCtxFilter()` |
| 分析 | カテゴリ/象限/タグ/文脈/週次トレンド | `renderAnalysis()` |
| 見積もり | 実績記録・タグ別補正係数 | `renderEstimate()` |
| 書き出し | ICS書き出し（自分用／共有用）・クラウド同期 | `buildICS()` / `syncUp()` / `syncDown()` |

## 重要な実装メモ

### 文脈（Context）の管理
- `state.contexts = [{id, name, type, color}, ...]` が正本
- ホームタブの文脈ページャは `homeCtxList()` → `[{id:null,name:"全予定"}, ...state.contexts]`
- 予定（state.items）は `customerType/customerName` で文脈を代替（後方互換）
  - フィルタには `itemMatchesCtx(it, ctx)` を使う（contextId → custKey(it)===ctx.name の順で照合）
- タスク（state.tasks）・マイルストーン（state.events）は `contextId` フィールドで正式管理
- `state.activeContext` はナビゲーションバーとの連携用

### 4象限 D&Dとタイトル表示
- ドラッグは `registerDrag(el, payload, {handleSel:'.drag-handle'})` で Pointer Events 方式
- モバイルでは `@media(pointer:coarse)` でハンドルを 48px 以上に拡大
- `.qrow.top` は `display: grid; grid-template-columns: auto auto auto auto 1fr` で実装
  - flex ではなく grid にしないとタイトルが消えるバグが発生する（flex の min-content 問題）

### クラウド同期
- `SYNC_PAT_KEY = 'jikanbo.syncpat'`（PAT を localStorage に保存）
- `SYNC_GIST_KEY = 'jikanbo.syncgist'`（Gist ID を localStorage に保存）
- 書き出しタブの「クラウド同期」セクションから操作

## 厳守ルール
1. **単一ファイル構成**：`jikanbo.html` 1枚で完結。外部ライブラリ・CDN・npm依存・ビルド工程は導入しない。オフラインでブラウザで開けることを維持。
2. **データ永続化は localStorage**：キーは `STORE_KEY`。保存スキーマを変えたら、必ず `load()` に旧データからの移行処理を追加し、`STORE_KEY` のバージョン番号を上げる（例 `v10 → v11`）。既存ユーザーのデータを失わせない。
3. **ドラッグ＆ドロップは Pointer Events 方式**（マウス・タッチ・ペン統一）を維持。HTML5 の `draggable`/`ondragstart`/`ondrop` には戻さない（iPad/iPhoneで動かないため）。
4. **配色は `:root` のCSS変数（デザイントークン）を使う**。紙とインクを想起させる低彩度・落ち着いたトーンを維持。鮮やかなiOS標準色には戻さない。
5. **既存機能を壊さない**。とりわけ **ICS出力ロジックは要件が固まるまで原則変更しない**（当面の最重要）。
6. **大きな変更の前後で git commit する**。常に「動いていた状態に戻せる」ようにする。各ステップは"動く状態"で終える。
7. **実名を埋め込まない**：顧客名・取引先名・個人名をコードやサンプルデータに直書きしない。サンプルは「顧客A大学」等のダミーを使う。
8. **全件を一度に描画・計算しない**：表示・集計は「見えている月・週・文脈の範囲」に限定する。データ量自体は個人利用なら問題ない（数千件でも数MB）が、年間に数千ピンを一度に描く等の作りは避ける。必要なら事前集計をキャッシュ。

## 予定の統一データモデル（①②③を支える共通の器）
1つの予定（event / task）に共通属性を持たせ、3機能が同じ器を共有する。
```
title       … 名前
contextId   … どの文脈（クリップボードのページ）に属するか
category    … fixed | recurring | growth | project | travel（内部区分。画面表示はアイコン）
tags[]      … タグID参照（各タグ＝アイコン＋色＋任意ラベル）
importance  … 重要度（★）
urgency     … 緊急度（⚡）
estimate    … 見積もり時間（分）
actual      … 実績時間（分・終了後に記録）
visibility  … public | busy | private（公開レベル）
date / time … 日付・時刻
memo        … メモ・備考（任意）
checklist   … 手順書ステップ string[]（任意）
```
- ① 共有ICS は `visibility` で出し分ける
- ② 分析 は `tags / importance / urgency / category / contextId` で集計
- ③ 見積もり精度 は `estimate` と `actual` を比較

## クリップボード（文脈ページ）
- 文脈ページは**可変**（追加・削除・並べ替え、目安〜10枚）。各ページに〈名前・種別（顧客/プロジェクト/プライベート/その他）・色〉。
- 「全予定」ページは常に全文脈の**合算**を表示。
- UIは「タブ切替」ではなく「盤面固定＋文脈をめくる」。盤面＝上:今月カレンダー / 下:今週＋年間 / 4象限 / 最下部:時間収支バー。
- ホームタブが Phase 1 として実装済み。スワイプ・◀▶・ドットで文脈切り替え。

## UI言語・表記
UIは日本語、ラベル・説明は敬語（丁寧語）で統一。

## Git / GitHub 運用
- `gh` CLI でログイン済み（Korotaneアカウント）。`git push origin main` を直接実行できる。
- GitHub Pages URL: `https://Korotane.github.io/jikanbo/jikanbo.html`
- commit 後に「GitHubにあげて」と言われたら、確認なしで `git push origin main` まで実行してよい。

## 実装の進め方
新しい要件に着手する前に、要件定義書と現行コード（jikanbo.html / clipboard-mock）を突き合わせ、解釈のズレを質問として挙げて認識合わせをすること。いきなり実装しない。推奨ビルド順は `jikanbo-START-HERE.md` および要件定義書 §11 を参照。
