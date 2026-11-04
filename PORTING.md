# PORTING.md — お引越し可能ファイル

他のプロジェクトへそのまま(または軽微な変更で)移植できる実装パターン
一覧。

## SRTT/RTTVAR(RFC 6298/9002準拠のEWMA)によるネットワーク品質推定
(`src/lib.rs`)

TCP(RFC 6298)・QUIC(RFC 9002)が実際に使うJacobson/Karels EWMA
アルゴリズムをそのまま流用した、RTT/ジッター推定の最小実装。固定
ウィンドウ+標準偏差方式(全サンプルをメモリに保持し毎回平均・分散を
計算し直す)より、サンプル1件ごとにO(1)の更新のみで済む。ネットワーク
品質に応じた適応制御が必要な、あらゆるRust製ネットワーククライアント
/サーバーへそのまま移植可能。

```rust
const ALPHA: f64 = 1.0 / 8.0; // RFC 6298と同じ重み
const BETA: f64 = 1.0 / 4.0;

fn update(prev: Option<(f64, f64)>, r: f64) -> (f64, f64) {
    match prev {
        None => (r, r / 2.0), // RFC 6298 2.2節: 初回サンプルの特別扱い
        Some((srtt, rttvar)) => {
            let rttvar = (1.0 - BETA) * rttvar + BETA * (srtt - r).abs();
            let srtt = (1.0 - ALPHA) * srtt + ALPHA * r;
            (srtt, rttvar)
        }
    }
}
```

## Fast/Slow 2段階適応方針パターン(`src/lib.rs`)

「ネットワーク品質の判定結果を`enum`で表し、呼び出し側のパラメータ
(リトライ間隔・タイムアウト・バッチサイズ等)を2段階で切り替える」
という設計は、ネットワーク品質に限らず、あらゆる「観測に基づいて
挙動を2モードで切り替えたい」場面(負荷に応じたスロットリング、
エラー率に応じたサーキットブレーカー等)に応用できる汎用パターン。

```rust
pub enum AdaptiveMode { Fast, Slow }

impl AdaptivePolicy {
    pub fn mode(&self) -> AdaptiveMode {
        if self.monitor.is_photonic_class() { AdaptiveMode::Fast } else { AdaptiveMode::Slow }
    }
}
```

## 外部クレート非依存でOS標準ツールを呼び出すパターン(`src/
network_interfaces.rs`・`src/download_protection.rs`・
`src/usb_protection.rs`、2026-08-11新設)

「ネットワークアダプタ一覧・ウイルススキャン・リムーバブルドライブ
検出」等、OS標準機能に依存する処理を、専用のRustクレート(`windows`
クレート等)を追加せずに`std::process::Command`でOS標準コマンド
(PowerShellの`Get-NetAdapter`・`Get-CimInstance`、ウイルス対策ソフトの
CLIスキャナー等)を呼び出しテキスト出力を解析する設計。

実際に踏んだ落とし穴と対処:
1. **ロケール依存の文字化け**: `ipconfig`の出力はシステムのANSI
   コードページ(日本語版なら Shift-JIS)で出力されるため、そのまま
   UTF-8として解釈すると文字化けする。`chcp 65001`での一時切替でも
   完全には直らないことがあった——**可能な限り、ロケールに依存しない
   構造化データ(PowerShellの`Get-NetAdapter`が返す`PhysicalMediaType`
   のような英数字の技術定数)を使う設計に倒す方が根本的に安全**
   (`network_interfaces.rs`の`ipconfig`→`Get-NetAdapter`移行の経緯
   参照)。
2. **特定ベンダー依存を避ける**: Windows Defenderの`MpCmdRun.exe`は、
   別のセキュリティ製品が導入されている環境ではオンデマンドスキャン
   機能自体が無効化されていることがある(実機で確認)。OS標準機能
   よりも、クロスプラットフォームで動作する独立した無料ツール
   (ClamAV等)を使う方が環境依存のリスクを減らせる。
3. **見つからない・無効な場合は正直に`Unavailable`を返す**: 「動く
   ふりをする」フォールバックは絶対に避け、呼び出し側が状態を正しく
   判別できる専用のenumバリアントを用意する(`ScannerUnavailable`等)。

## 未実装拡張点を安全にフォールバックさせるパターン(移植元:
`open-web-server-wire::accel`、本リポジトリと同日に新設)

「将来対応予定のハードウェア/バックエンドを`enum`の選択肢として先に
定義し、未実装のものが選ばれても panic せず既定の実装へ安全に
フォールバックしつつ`tracing::warn!`で可視化する」という設計は、
本クレート自体の`AccelBackend`(GPU/NPU未実装)と同じ考え方——
API形状を将来のハードウェア/機能に合わせて先取りしておき、実装が
追いつくまでは安全側の代替で動かす、という汎用パターン。

## 「スキャン結果に応じて警告解除のみ行い、実行はしない」パターン
(`src/download_protection.rs`の`verify_and_unblock`)

ダウンロードファイルの自動実行(AIによる自動判定込み)を求められた際、
安全性を保ったまま利便性を上げる代替として採用した設計。Windows標準の
「ブロックの解除」チェックボックスと同じ仕組み(Mark of the Web=
`<file>:Zone.Identifier`代替データストリームの削除)を、スキャン結果が
`Clean`の場合のみ行う。**ファイルを開く・実行する処理は一切含めない**
——最終的な実行操作は常にユーザー自身のダブルクリックに残す。「スキャンで
`Clean`」は「絶対安全」を意味しない(ゼロデイ・スキャン回避型マルウェア
の残存リスク)という前提に立ち、リスクの残る最終アクション(実行)だけは
自動化しない、という切り分けは、ダウンロード/添付ファイルを扱う他の
Rust製アプリへそのまま移植可能。

## 「起動時/定期実行の中身だけを提供し、スケジューリングはしない」
パターン(`src/maintenance.rs`)

「PC起動時や定期的に呼ばれるべき処理」を実装する際、呼び出しタイミング
の制御(OSのタスクスケジューラ登録・常駐サービス化)は呼び出し側アプリ
の責務として明確に外部化し、本体クレートは「呼ばれたら何をするか」
(ここではセキュリティソフト登録確認+ClamAV `freshclam`によるウイルス
定義更新)のみを提供する設計。定義更新に失敗した場合は黙って「最新」と
偽らず`DefinitionsUpdateUnavailable`を正直に返す。スケジューリングの
実装を持たずライブラリを疎結合に保ちたい場合に汎用的に移植可能。

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, redaktionell überarbeitet; noch nicht von Muttersprachlern geprüft)
- **REST wird vollständig abgeschafft; neue APIs entstehen nicht mehr als REST.** APIs, die aus dem Browser oder per `curl` aufgerufen werden, verwenden **unäre Connect-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body). Zwischen Diensten kommen **gRPC bzw. gRPC-web** zum Einsatz, im selben Stil wie die Admin-API von RPoem (`open_runo.admin.v1.Admin`). Streaming wird erst ergänzt, wenn es tatsächlich gebraucht wird.
- **Bestehende REST-Endpunkte werden schrittweise umgestellt, und zwar immer dann, wenn man ohnehin an ihnen arbeitet.** Sobald ein Endpunkt umgestellt ist, wird die alte REST-Route entfernt. Bis zum Abschluss der Migration bleiben die übrigen bestehen, es werden aber **keine neuen mehr hinzugefügt**.
- **JSON wird einheitlich über RS-JSON verarbeitet**: Gelesen wird mit `from_str_secure` bzw. `parse_secure` (mit Obergrenzen für Größe, Verschachtelungstiefe und doppelte Schlüssel), geschrieben mit `rust_json::fast`. `prost` wird nicht in das Browser-WASM aufgenommen, da es die Datei vergrößern würde.
- **Ausnahmen (Schnittstellen, deren Form ein fremder Standard vorgibt und die nicht zu unserer eigenen API gehören)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Drittanbietern (z. B. GitHub), Health-Checks, die Auslieferung statischer Dateien, OTLP und Ähnliches. Diese nehmen, wie vom jeweiligen Standard verlangt, gewöhnliches HTTP entgegen.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines einzelnen Repositorys diese Richtlinie bereits erfüllt, hängt vom jeweiligen Repository ab. Dieser Abschnitt dokumentiert lediglich die Richtlinie; er besagt nicht, dass der Code schon angepasst wurde.

### Русский (перевод выполнен с помощью ИИ и отредактирован; носителями языка пока не проверен)
- **От REST мы полностью отказываемся; новые API больше не создаются в стиле REST.** API, к которым обращаются из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело запроса — JSON). Для взаимодействия между сервисами применяются **gRPC и gRPC-web** — в том же стиле, что и административный API RPoem (`open_runo.admin.v1.Admin`). Потоковая передача данных добавляется лишь тогда, когда в ней действительно возникает необходимость.
- **Существующие REST-эндпоинты переводятся поэтапно — по мере того, как с ними приходится работать.** После перевода старый REST-маршрут удаляется. Пока миграция не завершена, оставшиеся эндпоинты сохраняются, но **новые не добавляются**.
- **Работа с JSON унифицирована на основе RS-JSON**: чтение — через `from_str_secure` или `parse_secure` (с ограничениями на размер, глубину вложенности и повторяющиеся ключи), запись — через `rust_json::fast`. `prost` в браузерный WASM не включается, поскольку это увеличило бы размер файла.
- **Исключения (интерфейсы, форму которых определяет чужой стандарт, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и тому подобное. Такие интерфейсы принимают обычный HTTP, как того требует соответствующий стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Данный раздел лишь фиксирует политику и не означает, что код уже изменён.
