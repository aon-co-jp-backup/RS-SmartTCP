# RS-SmartTCP

**開発開始日: 2026-07-23**(このリポジトリのGitHub作成日)

> 📌 **v0.2.0(2026-08-11)**: 当初のRTT/ジッター適応制御に加え、
> 有線LAN・WiFi・Bluetooth(各最大10チャンネル)+複数WAN回線(最大10本、
> IPv4/IPv6/v6プラス対応)の管理、ルーター/セキュリティルーター機能+
> 既知プラグイン、ダウンロードファイルのウイルススキャン(ClamAV/
> KINGSOFT)+自動実行防止、USBリムーバブルドライブ保護(autorun.inf
> 無害化)を追加。**ファイルの自動実行(AIによる自動判定含む)は明確に
> 実装を見送り**、代わりにスキャンで安全と確認できたファイルのみ
> Windowsのダウンロード警告(Mark of the Web)を解除する
> `verify_and_unblock`(実行操作自体は常にユーザー任せ)と、
> 起動時/定期実行を想定した`maintenance`モジュール(セキュリティソフト
> 登録確認+ClamAV定義更新、本クレート自体はスケジューリングしない)を
> 追加。詳細は[CLAUDE.md](CLAUDE.md)のHANDOFF参照。

IOWN/APN(NTTのオールフォトニクス・ネットワーク、光電融合)のような
超低遅延・ジッター無し回線と、Smart-TCP(AI生成通信プロトコル、
fast/slowモデルによる判断構造)の良いとこ取りハイブリッド適応制御。
実測RTT・ジッターに基づき、TCP(RFC 6298)/QUIC(RFC 9002)と同じ
SRTT/RTTVAR(Jacobson/Karels EWMA)アルゴリズムでネットワーク品質を
分類し、リトライ間隔等を2段階(Fast/Slow)で切り替える。

## これは何か

- **`NetworkQualityMonitor`**: RTTサンプルを記録し、SRTT(平滑化RTT)・
  RTTVAR(RTT変動)をTCP/QUICと全く同じEWMAアルゴリズムで追跡する。
  両方が閾値未満なら「photonic-class」(IOWN/APNのような光ネットワーク
  級)、そうでなければ「standard-class」と判定する。
- **`AdaptivePolicy`**: 判定結果に応じてリトライ待機時間等を切り替える
  (Fast=光ネットワーク級向けの積極的な設定、Slow=通常インターネット
  向けの保守的な設定)。

## 正直な開示・命名の経緯

**本クレートは、arXiv 2512.00491("Agentic AI-based Autonomous and
Adaptive TCP Protocol"、"Smart-TCP")のプロトコルそのものの実装では
ない。** 訓練済み機械学習モデルは使わず、「fast/slowモデル」という
設計思想を、TCP/QUICが実際に使うSRTT/RTTVAR EWMAに基づく決定論的な
2値判定として実装したもの。`RS-SmartTCP`という名前は「Smart-TCPに
着想を得た、このエコシステム独自の実装」であることを示す
(既存の`RS-`接頭辞の命名規則に準拠)。論文の同名プロトコルと混同
しないこと。

同様に、IOWN/APN自体はNTTが構築する物理telecom基盤(光電融合スイッチ・
光ファイバー回線)であり、本クレートが「実装」できる対象ではない
——実際に行っているのは「そのような回線が来た時にソフトウェア層が
足を引っ張らない」設計のみ。

## 使用例

```rust
use rs_smarttcp::{AdaptivePolicy, NetworkQualityMonitor};
use std::time::Duration;

let policy = AdaptivePolicy::new(NetworkQualityMonitor::new());

// 実測RTTを記録していく(呼び出し側が実際に計測した値を渡す)
policy.monitor().record_rtt(Duration::from_millis(17));

// 判定に応じたリトライ間隔を取得
let backoff = policy.retry_backoff();
```

## v0.2.0で追加した機能一覧(2026-08-11)

- **`network_interfaces`**: 有線LAN・WiFi・Bluetoothの接続状況を
  `Get-NetAdapter`(PowerShell、ロケール非依存の技術定数で判定)経由で
  検出。
- **`multi_path`**: 有線LAN最大10本+WiFi最大10チャンネル+Bluetooth
  最大10チャンネルの中から最良RTTの経路を選び、自動フェイルオーバーする
  (`MultiPathManager`)。ルーター・NAS・外付けHDD・PC・タブレット・
  スマホ・TV・ゲーム機のラベル付けにも対応。**正直な開示**: 本物の
  帯域合算リンクアグリゲーションではない(OS/NICチーミングが必要)。
- **`multi_wan`**: 名前付きWAN回線を最大10本管理(`MultiWanManager`)。
- **`wan_config`**: IPv4/IPv6/v6プラス(MAP-E)+WAN自動設定フラグ
  (`WanConfig`)。**正直な開示**: 実際のDHCPv6-PD交渉・トンネル確立は
  OS/ルーター機器側が行う、設定意図フラグのみ。
- **`bandwidth_policy`**: YouTube/U-NEXT/Qobuz等のストリーミング時のみ
  10Mbps固定、それ以外(SFTP・AIチャットツール等)は常に最高速度。
- **`router_features`**: ルーターアプリ/セキュリティルーター機能の
  チェックボックス+既知プラグイン(ポート転送・QoS・DHCP、広告ブロック・
  DNSフィルタ・ペアレンタルコントロール、TLS検査/AI侵入検知は設定
  フラグのみで未実装)。**正直な開示**: 任意の外部コード実行機構ではない。
- **`download_protection`**: ダウンロードファイルをClamAV(オープン
  ソース)またはKINGSOFT Internet Security(無料版)で実際にスキャン。
  脅威は削除ではなく隔離フォルダへ移動(可逆的)。Windows「Mark of the
  Web」による自動実行防止。Windows Security Center未登録時のセキュリティ
  ソフト導入推奨メッセージ(日英)。
- **`usb_protection`**: リムーバブルドライブ検出+`autorun.inf`無害化
  (リネーム、削除ではない)+ドライブスキャン。

`examples/status_gui.rs`(`cargo run --example status_gui`)で全機能を
実際にブラウザから確認できる。

## このエコシステムでの利用箇所

[`open-web-server-wire`](https://github.com/aon-co-jp/open-web-server)
からpath依存として利用される(`Rust-JSON`が`aruaru-db`等から利用される
のと同じ「独立リポジトリとして切り出し、必要な場所からpath依存する」
パターン)。

## ビルド・テスト

```bash
cargo test
```

外部依存クレート無し(標準ライブラリのみ)。

## ライセンス

Apache-2.0

---

作者 / Author: 石 塚 正 浩（MASAHIRO ISHIZUKA） (https://github.com/aon-co-jp)

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

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
