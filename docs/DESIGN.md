# keycat 設計書

- Repository: `github.com/Saber5656/keycat`
- License: MIT（仮）
- Status: v1 設計 / 実装前
- 作成日: 2026-07-05

---

## 1. コンセプト

keycat は、Enter キーを押した瞬間に画面下からネコが「にゅっ」と現れ、Enter のタイミングに合わせて手を叩いて祝ってくれるデスクトップ常駐アプリである。普段は完全に隠れており、CPU もメモリもほぼ消費しない。コマンド実行、コミット、メッセージ送信——開発者の一日は無数の Enter でできており、そのすべてを小さな祝福に変える。グローバルキーフックを使う以上、プライバシーが信頼の生命線であることを最初から設計の中心に置く：**見るのは「Enter が押された」という事実だけ。何を打ったかは、コード上も原理上も知り得ない構造にする。** 配信（OBS）映えを重視し、グリーンバック表示モードを v1 から備える。

---

## 2. v1 スコープ

「Enter → ネコがクラップ」のコア体験と、それを安心して使える権限・プライバシー体験だけに絞る。

| 機能 | v1 | v2 以降 | 判断理由 |
|---|---|---|---|
| Enter 検知 → 出現 → クラップ → 引っ込み | ✅ | — | コア体験そのもの |
| 連打時のクラップ連続再生（引っ込みタイマーのリセット） | ✅ | — | 連打で毎回出入りするとうるさい。状態機械の設計に含めるだけでコストが低い |
| コンボカウント表示（"x12!" などの数字演出） | ❌ | ✅ v2 | 描画・フォント・演出調整のコストが高く、コア体験に必須でない。状態機械には拡張ポイントだけ用意 |
| exit code 連動（成功/失敗でネコの表情が変わる） | ❌ | ✅ v2 | シェル統合が必要（§詳細は本節末尾） |
| グリーンバック表示モード（OBS 用クロマキー背景） | ✅ | — | 実装は「背景色を塗るだけ」でほぼゼロコスト。OBS 対応の現実解（§8） |
| ブラウザソース用ローカル WebSocket サーバ | ❌ | ✅ v2 | 「ネットワーク機能を一切持たない」という v1 のプライバシー主張を単純に保つため（§6, §8） |
| メニューバー常駐 UI（有効/無効、プレビュー、終了） | ✅ | — | 常駐アプリの最低限の操作面 |
| 初回起動オンボーディング（権限誘導） | ✅ | — | 権限が取れないと何も起きないアプリなので、初回体験＝製品体験 |
| 設定：表示ディスプレイ / 出現位置 / サイズ / クールダウン | ✅（最小限） | 拡張 | ディスプレイ選択と位置はマルチモニタ環境で必須。テーマ切替等は v2 |
| 効果音 | ❌ | ✅ v2 | 好みが分かれる。ミュート設計を含め v2 で |
| ネコ以外のキャラ / スキン | ❌ | ✅ v2 | アセット規約の整備が必要 |
| Windows / Linux 対応 | ❌ | ✅ v2（Windows→X11 の順） | §3 |
| 自動アップデート（Sparkle 等） | ❌ | ✅ v2 | ネットワーク不使用の主張を保つ。更新は Homebrew / 手動 |

**exit code 連動を v2 とする判断根拠**：キーフックだけでは exit code は取得できず、シェル統合（zsh の `precmd`、bash の `PROMPT_COMMAND`、fish の `fish_postexec`）から UNIX ドメインソケット等でアプリに通知する仕組みが必要になる。これは (a) ユーザーのシェル設定ファイルを書き換えるというプライバシー/信頼面のハードルが上がる、(b) ローカル IPC の受け口が生まれ「外部入力を受けない」という v1 の攻撃面ゼロ主張が崩れる、(c) ターミナル以外での Enter（エディタ、Slack 等）と意味が競合し UX 設計が複雑になる、の 3 点で v1 の「早く出す」に反する。v1 の状態機械にはイベント種別（`clap` / 将来の `celebrate` / `sad`）を持たせておき、v2 で通知ソースを足すだけで拡張できる形にする。

---

## 3. 対応プラットフォームと優先順位

| 優先度 | プラットフォーム | 方式 | 状態 |
|---|---|---|---|
| 1 (v1) | macOS 13+ | CGEventTap（listen-only, keyDown マスク限定） | v1 で対応 |
| 2 (v2) | Windows 10/11 | `SetWindowsHookEx(WH_KEYBOARD_LL)` | 設計だけ v1 で確保 |
| 3 (v2+) | Linux / X11 | XRecord 拡張 | 同上 |
| 対象外 (当面) | Linux / Wayland | 原則不可能。代替のみ提供 | README に明記 |

理由：

- **macOS 最優先**：開発者の環境が macOS。まず自分が毎日使えるものを出すのが個人 OSS の鉄則。
- **Windows が第 2**：`WH_KEYBOARD_LL` は特別な権限プロンプトなしで低レベルキーフックが張れる公式 API であり、実装ハードルが低い。ただしフックプロシージャに **タイムアウト制約**（`LowLevelHooksTimeout`、Windows 10 1709+ では最大 1000ms、超過時は **サイレントにフックが解除され、アプリはそれを検知できない**）があるため、「フックスレッドはキーコード判定と enqueue だけを行い即 return、描画は別スレッド」という構造が必須。これは §5 のアーキテクチャがそのまま満たす。
- **Linux/X11 が第 3**：XRecord でグローバルキー監視が可能だが、ディストリ差分の検証コストが高い。
- **Wayland は対象外と明記**：Wayland はセキュリティ設計として、アプリが他アプリ宛のキー入力を盗聴することを**プロトコルレベルで禁止**している。グローバルフックはコンポジタが仲介する前提で、汎用の「全アプリの Enter を検知する」手段は存在しない。`/dev/input/event*` を直接読む回避策は root 相当（`input` グループ）の権限が必要で、「キーロガーと区別がつかない」ものになりプライバシー方針と真っ向から矛盾するため採用しない。Wayland ユーザー向けには v2 のブラウザソースモード＋シェル統合（Enter ではなくコマンド完了で反応）を代替として案内する。README の対応表で正直に「Wayland: not supported (by design of Wayland)」と書くこと自体が、このプロジェクトのプライバシー姿勢の証明になる。

---

## 4. 技術選定

### 言語・フレームワーク比較

| 候補 | メモリ/CPU | macOS 権限 UX | 透過オーバーレイ | クロスプラットフォーム | 判定 |
|---|---|---|---|---|---|
| **Swift + AppKit（ネイティブ）** | ◎ 常駐 ~20-30MB | ◎ 公式 API 直叩き（preflight/request） | ◎ NSPanel + Core Animation | ✕ macOS 専用 | **v1 採用** |
| Rust + winit/tao + rdev | ○ ~15-30MB | △ rdev の macOS 実装品質・メンテ状況に不安。権限プロンプト制御を自前実装 | △ 透過・クリック透過・全 Space 表示は結局プラットフォーム別コード | ○ | v2 移植時に再評価 |
| Tauri | △ WebView 分 +50-100MB | △ 同上（プラグイン依存） | △ | ○ | 見送り |
| Electron | ✕ 150MB+ | △ | ○ | ◎ | 「CPU/メモリ極小」に反し即却下 |

**結論：v1 は Swift + AppKit のネイティブ macOS アプリ。** 理由：

1. 「CPU/メモリ極小」の制約に最も適合する（WebView・ランタイム同梱なし）。
2. 権限まわり（`CGPreflightListenEventAccess` / `CGRequestListenEventAccess`、設定アプリの該当ペインへのディープリンク）を公式 API で正確に制御でき、初回体験の質を最大化できる。
3. クロスプラットフォーム抽象は v1 では**コード共有ではなくアーキテクチャ共有**で担保する。§5 の「KeySource プロトコル / 演出層の分離」を守れば、Windows/Linux 版は同じ設計の別実装（あるいは将来 Rust コアへの置換）として追加できる。個人 OSS で最初から 3 OS を抱えるのは出荷しない原因の筆頭。

### macOS キーフック方式：CGEventTap listen-only vs NSEvent global monitor

| | CGEventTap (listen-only) | NSEvent.addGlobalMonitorForEvents |
|---|---|---|
| 必要権限 | **Input Monitoring**（入力監視） | **Accessibility**（アクセシビリティ） |
| 権限の事前確認/明示リクエスト API | ◎ `CGPreflightListenEventAccess` / `CGRequestListenEventAccess` あり | △ `AXIsProcessTrustedWithOptions` のみ |
| イベント改変 | 可能（listen-only なら不可） | 不可（コピー受信のみ） |
| 受信マスク指定 | ◎ `CGEventMask` で **keyDown のみに限定可能** | ○ `matching: .keyDown` で同等 |
| ユーザーから見た権限名の正直さ | ◎ 「入力監視」＝やることそのまま | △ 「アクセシビリティ」は過大に見える（UI 操作等も許す強い権限） |

**採用：CGEventTap を `kCGEventTapOptionListenOnly` + `kCGHIDEventTap` ではなく `kCGSessionEventTap`、マスク `keyDown` のみで使用。** 決め手は 2 点：(1) 権限の preflight / 明示リクエスト API が揃っており、オンボーディング UI（§7）を正確に作れる。(2) 要求する権限が「入力監視」であり、keycat が実際にやることと権限名が一致する。Accessibility はアプリの UI 操作まで許す包括的権限で、「Enter を見るだけのアプリ」が要求するのは過剰であり、プライバシー説明の説得力を損なう。listen-only tap は定義上イベントを改変・遅延させないため、フック起因の入力遅延リスクもない。

- 描画：`NSPanel`（borderless / `ignoresMouseEvents = true` / `level = .statusBar` / `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]`）＋ Core Animation のスプライトフレームアニメーション（`CALayer.contents` 差し替え、`CADisplayLink` は再生中のみ）。
- 設定保存：`UserDefaults`。 
- 依存ライブラリ：**ゼロを目標**（プライバシー検証可能性のため。§6）。

---

## 5. アーキテクチャ

### レイヤ分離

```
┌──────────────────────────────────────────────────────┐
│  App / MenuBar 層（AppKit）                            │
│  ステータスアイテム、設定、オンボーディング              │
└──────────────┬───────────────────────────────────────┘
               │ enable / disable / 設定変更
┌──────────────▼───────────────┐   ┌───────────────────┐
│  KeySource 層（イベント源）     │   │  Presentation 層    │
│  protocol KeySource {         │──▶│  CatOverlay        │
│    var onTrigger: ()->Void    │ ① │  ・状態機械         │
│    func start()/stop()        │   │  ・NSPanel + CA    │
│  }                            │   │  ・スプライト描画    │
│  実装: EnterTapSource          │   └───────────────────┘
│  （CGEventTap, listen-only）   │
└──────────────────────────────┘
   ① 伝達するのは「引数なしのトリガ通知」のみ。
      キーコード・文字・修飾キー・タイムスタンプ以外の情報は境界を越えない。
```

設計上の要点：

- **KeySource → Presentation 間のインターフェースを「引数のないトリガ通知」に固定する。** キーイベントの型（`CGEvent`）が演出層に漏れない。これはプライバシー保証（§6）の構造的な担保であると同時に、v2 で `ExitCodeSource`（シェル統合）や `ManualSource`（プレビュー用）を足すための拡張点になる。トリガに種別が必要になったら `enum TriggerKind { case clap }` を渡す（ペイロードは列挙型のみ、生イベントは絶対に渡さない）。
- **tap コールバックは最小仕事**：keyCode 比較 → 該当ならメインスレッドへ `onTrigger` を dispatch → 即 return。この構造は Windows 版の WH_KEYBOARD_LL タイムアウト制約（§3）にもそのまま適用できる。

### 処理フロー

```
CGEventTap(callback)                       [イベントスレッド]
  └─ event.keyCode == 36(Return) or 76(KeypadEnter)?
       ├─ No  → return event（何もしない。読み取りもここで終わり）
       └─ Yes → DispatchQueue.main.async { keySource.onTrigger() }
                                            [メインスレッド]
CatOverlay.handleTrigger()
  └─ 状態機械へ入力
```

### 演出の状態機械

```
                 trigger
   ┌────────┐ ───────────▶ ┌────────┐  rise 完了   ┌────────┐
   │ Hidden │              │ Rising │ ───────────▶ │  Clap  │◀─┐
   └────────┘ ◀─────────── └────────┘              └───┬────┘  │ trigger
        ▲      (即 retract:                clap 完了    │       │ (再クラップ +
        │       設定 OFF 時)                            ▼       │  linger リセット)
        │                                          ┌────────┐  │
        └───────────────────  linger 満了 ──────── │ Linger │──┘
                 ┌─────────┐ ◀──────────────────── └────────┘
                 │ Retract │      ※ Retract 中に trigger → Rising へ逆再生で復帰
                 └─────────┘
```

| 状態 | 内容 | 時間（初期値） |
|---|---|---|
| Hidden | ウィンドウ非表示（`orderOut`）。CADisplayLink 停止 | — |
| Rising | 画面下端からせり上がり（ease-out） | 180ms |
| Clap | クラップ 1 回（スプライト 4–6 フレーム） | 250ms |
| Linger | 待機ポーズ。次の Enter を待つ | 800ms |
| Retract | 下へ引っ込む（ease-in）→ Hidden | 220ms |

**連打時の挙動（v1）**：

- Linger / Clap 中に trigger が来たら、Clap を頭から再生し直し、Linger タイマーをリセットする。→ 連打中はネコが出ずっぱりで叩き続ける（これが気持ちよさの核）。
- **レートリミット**：クラップ再生開始は最短 100ms 間隔（10 claps/s 上限）。それ以上の連打はアニメーション再スタートのみ抑制し、trigger 自体は無視してよい（カウントは v2 のコンボ用に内部インクリメントだけ用意）。
- Retract 中の trigger は Rising（逆再生位置から）へ遷移し、出入りのバタつきを防ぐ。
- **v2 拡張点**：Clap→Clap の連続回数カウンタが既にあるため、「N 回以上でコンボ数値表示」「exit code イベントで Clap の代わりに celebrate/sad モーション」を状態追加なしで実装できる。

### リソース使用方針（CPU/メモリ極小）

| 項目 | 方針 |
|---|---|
| アイドル時 CPU | ~0%。tap コールバックは keyDown 時のみ起床し整数比較 1 回。ポーリングなし |
| アニメーション | 再生中のみ CADisplayLink。Hidden で完全停止 |
| メモリ | スプライトは 1 セット（~数百 KB）を起動時ロード。目標 RSS 30MB 以下 |
| GPU | CALayer 差し替えのみ。ブラー・パーティクル等は v1 では使わない |
| 電力 | タイマーは linger 用の 1 本のみ。`tolerance` を設定し coalescing を許可 |

---

## 6. プライバシー設計（最重要）

### 6.1 原則：「何を見て、何を見ないか」

| | 内容 |
|---|---|
| **見るもの** | keyDown イベントの **keyCode フィールドのみ**（Return=36 / KeypadEnter=76 との整数比較） |
| **見ないもの** | 文字・Unicode 文字列、修飾キー、キーの組み合わせ、押下の対象アプリ、keyUp、マウス、クリップボード |
| **残すもの** | 何も残さない。キー由来のデータはログ・ファイル・メモリバッファのいずれにも保存しない |
| **送るもの** | 何も送らない。ネットワークコードが存在しない |

### 6.2 イベントフィルタリングの多層設計

技術的事実として、Input Monitoring を許可された event tap は全キーの keyDown を受信「しうる」。これを認めた上で、受信情報を段階的に絞る：

1. **層 1 — tap マスク**：`CGEventMask = 1 << kCGEventKeyDown` のみで tap を作成。keyUp・flagsChanged・マウス等は OS レベルで届かない。
2. **層 2 — コールバック内即時判定**：受信した `CGEvent` から読むフィールドは `keyboardEventKeycode` の**一つだけ**。一致しなければ何もせず return（listen-only なので return 値はイベントに影響しない）。`CGEvent` はコールバックスコープを出ない＝キー情報の寿命はこの関数内で終わる。
3. **層 3 — 文字化 API の不使用**：`CGEventKeyboardGetUnicodeString` や `NSEvent.characters` 等、キーコードを文字に変換する API を**リポジトリ全体で一切呼ばない**（後述の CI ガードで機械的に検証）。
4. **層 4 — 境界の型**：KeySource → 演出層のインターフェースは引数なしの通知（§5）。仮に演出層にバグがあってもキー情報はそこに存在しない。

### 6.3 ログ方針

- キーイベントに関するログは**成功パスでは一切出力しない**。デバッグビルド限定でも「Enter trigger fired」（内容を含まない発火事実）まで。
- 診断ログは権限状態・tap の生死・状態機械の遷移のみ。`os_log` の privacy 指定はすべて static 文字列。
- クラッシュレポート送信機構（Sentry 等）は組み込まない。

### 6.4 ネットワーク不使用の保証

- v1 のコードベースに **URLSession / Network.framework / ソケット API を含めない**。依存ライブラリゼロ方針（§4）はこの検証を単純にするためでもある。
- App Sandbox を有効化し、entitlements で **ネットワーク関連 entitlement（`com.apple.security.network.client` / `.server`）を付与しない**。sandbox 下では entitlement のない outbound 接続は OS が拒否する。→「作者を信じる」ではなく「OS が禁じている」を主張できる。これが keycat の最強のプライバシー主張。
  - 検証項目（§11）：sandbox + Input Monitoring（CGEventTap listen-only）の両立は macOS 10.15+ で可能とされるが、実機での動作確認を実装前検証の最優先に置く。万一 sandbox と両立しない場合も、ネットワーク API 不使用＋下記 CI ガードは維持する。
- 自動アップデートを入れない（§2）のも同じ理由。更新配布は GitHub Releases / Homebrew に委ねる。

### 6.5 検証可能性（コードと README の両方で）

- **コード側**：
  - キーイベントを扱うコードを `Sources/KeySource/EnterTapSource.swift` の**単一ファイルに隔離**し、100 行以下に保つ。第三者は このファイルだけ読めば監査が完了する。
  - コールバック該当行に `// PRIVACY:` コメントアンカーを置き、README から行パーマリンクで参照。
  - **CI ガード**（GitHub Actions）：`git grep` で禁止シンボル（`KeyboardGetUnicodeString`, `NSEvent.characters`, `URLSession`, `Network.framework`, `NWConnection`, `socket(` 等）を検出したら fail。「守っていること」を CI バッジで常時証明する。
- **README 側**（§10）：Privacy セクションに (1) 見る/見ない表、(2) `EnterTapSource.swift` への直接リンク、(3) entitlements ファイルへのリンク（ネットワーク entitlement がないことの確認方法）、(4) CI ガードの説明、(5) ユーザー自身で確認できる手順（`codesign -d --entitlements - /Applications/keycat.app` の実行例）を載せる。
- **権限の説明責任**：オンボーディング UI（§7）でも同じ内容を要約表示し、「OS の権限名（入力監視）は広いが、実際に読むのはこれだけ」というギャップを初回起動時に説明する。

---

## 7. UI/UX

### 初回起動フロー（権限誘導）

```
起動 → CGPreflightListenEventAccess() で権限確認
 ├─ 許可済み → メニューバー常駐開始（ウィンドウは出さない）＋ ネコが一度だけ挨拶クラップ
 └─ 未許可 → オンボーディングウィンドウ表示
      [1] keycat の 10 秒説明（デモ GIF 同梱再生）
      [2] プライバシー説明（§6.5 の要約 + GitHub リンク）
      [3] [権限をリクエスト] → CGRequestListenEventAccess()
          → システム設定「プライバシーとセキュリティ > 入力監視」へのディープリンクボタン併設
      [4] 許可検知（1s ポーリング、この間のみ）→ お祝いクラップ → ウィンドウ自動クローズ
```

**権限が拒否された場合のフォールバック**：

- アプリは終了せず**デモモード**で常駐する：メニューバーから「Clap now（手動クラップ）」で演出だけ楽しめる。→ 権限なしでも価値ゼロにしない。演出の動作確認・OBS のセッティングも権限なしで可能。
- メニューバーアイコンに ⚠ バッジ。メニュー先頭に「入力監視が未許可です → 設定を開く」を常設。
- 起動のたびにモーダルで迫らない（初回のみ。以降はメニューバーの導線に留める）。

### メニューバー UI（常駐の主インターフェース）

```
🐾 keycat
──────────────
✓ Enabled                （トグル。⌥クリックで一時停止 30min）
  Clap now               （手動トリガ＝プレビュー兼デモモード）
──────────────
  Position      ▸ Display / 画面下オフセット / 左右位置
  Size          ▸ S / M / L
  Green screen mode      （OBS 用。§8）
──────────────
  Privacy & About…       （§6.5 の要約と GitHub リンク）
  Launch at login
  Quit
```

- Dock アイコンなし（`LSUIElement = true`）。設定は独立ウィンドウを作らずメニュー内サブメニューで完結（v1 の実装量削減）。
- 「Clap now」がそのまま**演出プレビュー**になる（設定変更 → Clap now → 確認、のループ）。

### 設定項目（v1）

| 項目 | 値 | 既定 |
|---|---|---|
| Enabled | on/off | on |
| 表示ディスプレイ | ディスプレイ選択 | メイン |
| 左右位置 | left / center / right / カスタム(%) | right |
| サイズ | S/M/L | M |
| Green screen mode | on/off | off |
| Launch at login | on/off | off |

---

## 8. OBS / 配信対応の設計判断

**前提（検証結果）**：OBS の macOS ウィンドウキャプチャは**透過ウィンドウのアルファチャンネルを保持できない**。透過オーバーレイを直接「抜き」で取り込む正攻法は存在しない。

これを踏まえた段階的な設計判断：

| 手段 | 内容 | 配置 | 判断理由 |
|---|---|---|---|
| **画面キャプチャに映り込む**（デフォルト） | 配信者が画面全体をキャプチャしていれば、ネコは実画面の一部としてそのまま映る | v1（何もしなくて良い） | ゲーム配信以外のデスクトップ配信ではこれで十分「映える」 |
| **グリーンバックモード** | ネコ用ウィンドウの背景を `#00FF00` 不透明で塗り、角丸なしの矩形ウィンドウにする。配信者は OBS で「ウィンドウキャプチャ + クロマキー（緑）」を設定 | **v1** | 実装コストが背景色 1 枚分と極小。OBS 側の手順も配信者には常識的。アルファ非保持問題への現実解 |
| **ブラウザソース用ローカルサーバ** | `localhost` の WebSocket でトリガを配り、同梱 HTML（Web 版ネコアニメ）を OBS ブラウザソースに読ませる。アルファ完全対応・配置自由 | **v2** | 品質的には最良だが、(1) ネットワーク API を持ち込み「ネットワークコード不存在」の v1 プライバシー主張（§6.4）が崩れる、(2) Web 版アニメの二重実装、のコストが大きい。v2 で entitlement を loopback server に限定し、README のプライバシー説明を改訂した上で導入 |

- README に「Streaming with OBS」セクションを設け、グリーンバックモードの設定手順（スクリーンショット付き）を載せる（§10）。
- グリーンバックモード時はウィンドウを通常レベル・キャプチャしやすい独立ウィンドウ（タイトルバーなし・ドラッグ移動可）に切り替え、配信レイアウトに合わせて動かせるようにする。
- v2 のブラウザソース対応は Wayland ユーザーへの代替手段（§3）も兼ねる。

---

## 9. 配布方法

| 項目 | 方針 |
|---|---|
| 配布チャネル | GitHub Releases（`.dmg` または `.zip`）＋ Homebrew Cask（`brew install --cask keycat`） |
| 署名 | **Developer ID Application 証明書で署名（必須扱い）** |
| notarization | `notarytool` で必須（macOS 14+ では未 notarize アプリの起動ハードルが非常に高い） |
| App Store | 出さない（Input Monitoring 前提のユーティリティは審査・体験両面で不利。sandbox 化自体は §6.4 のため行う） |
| CI | GitHub Actions：build → codesign → notarize → staple → Release 添付。禁止シンボル grep ガード（§6.5）も同一ワークフローで実行 |

**署名と Input Monitoring 権限の関係（重要）**：TCC（権限 DB）はアプリを**コード署名の同一性**で識別する。ad-hoc 署名はビルドごとに別 ID になるため、**更新のたびに権限が剥がれる／権限リストに登録すらされない**事象が起きる。Developer ID 署名なら更新後も同一アプリとして権限が維持される。つまり **Apple Developer Program（$99/年）は keycat では UX 必須コスト**であり、「署名なしで配って `xattr -d com.apple.quarantine` してもらう」運用は権限が毎回リセットされるため成立しない。README のトラブルシューティングに `tccutil reset ListenEvent <bundle-id>` を記載する。

---

## 10. README 構成案（英語）

```markdown
<banner: ネコがクラップする GIF をループ再生（幅いっぱい）>

# keycat 🐱👏
A tiny menu-bar app: press Enter, and a cat pops up from the
bottom of your screen to clap along. That's it. That's the app.

<demo GIF: ターミナルで Enter 連打 → ネコが叩き続ける実録>

## Install
    brew install --cask keycat
Or grab the notarized .dmg from Releases. macOS 13+.

## 🔒 Privacy — read this first        ← 目立たせる（最上部近く、絵文字+区切り線）
keycat needs macOS "Input Monitoring" permission. Here is exactly
what it does and does not do:
  | keycat sees            | keycat never sees / does           |
  |------------------------|------------------------------------|
  | key-down events' keycode, compared to Enter (36/76) | which characters you type |
  | —                      | logging, buffering, or storing keys |
  | —                      | any network access — the app has **no networking code and no network entitlement (OS-enforced)** |
- All key handling lives in one file: [`EnterTapSource.swift`](permalink) (< 100 lines — audit it yourself)
- Entitlements: [`keycat.entitlements`](link) — no network entitlements
- CI fails if any char-decoding or networking symbol appears: [privacy-guard workflow](link)
- Verify the shipped binary yourself:
      codesign -d --entitlements - /Applications/keycat.app

## Streaming with OBS
Green-screen mode + chroma key. <スクショ手順 3 枚>

## Settings / Combos & exit-code reactions (roadmap) / Platform support
(Wayland: not supported — Wayland by design forbids global key hooks. See #issue)

## Troubleshooting
(permission not sticking → tccutil reset ..., etc.)

## License
MIT
```

- 1 行インストール（Homebrew）を Install 節の先頭に。
- Privacy セクションはバッジ（`privacy-guard: passing`）を README 上部のバッジ列にも置く。

---

## 11. リスクと実装前検証項目

| 優先度 | 項目 | 内容 / 検証方法 |
|---|---|---|
| **P0** | App Sandbox + CGEventTap(listen-only) の両立 | sandbox 有効 & network entitlement なしで Input Monitoring が機能するか実機検証。不可なら sandbox を諦め「ネットワーク API 不使用 + CI ガード」に主張を切り替える（§6.4 の根幹） |
| **P0** | Developer ID 証明書の入手 | Apple Developer Program 加入。未加入なら配布計画（§9）全体が変わるため最初に確定 |
| **P0** | セキュアインプット（パスワード欄）時の挙動 | Secure Event Input 中はイベントが届かない想定。ネコが出ない旨を README に明記できるか確認（プライバシー上はむしろ加点） |
| **P1** | tap の自動無効化からの回復 | イベントタップは timeout / システム状態で `kCGEventTapDisabled*` になる。検知 → 再有効化の実装と、権限剥奪との区別を検証 |
| **P1** | フルスクリーンアプリ / Space 切替時のオーバーレイ表示 | `.canJoinAllSpaces + .fullScreenAuxiliary` でゲーム・動画全画面時に出るか。出ない環境の一覧化 |
| **P1** | OBS グリーンバックモードの実写確認 | OBS ウィンドウキャプチャ + クロマキーで縁の緑残りが許容範囲か。スプライトの縁処理（マット処理）の要否判断 |
| **P2** | キーリピート（Enter 長押し）の扱い | autorepeat フラグ（`kCGKeyboardEventAutorepeat`）を無視するか反応するか。連打演出と干渉しない既定を決める |
| **P2** | 多言語キーボード / テンキー Enter | JIS/US、keypad Enter(76) の実機確認 |
| **P2** | 低電力モード・バッテリー影響測定 | 1 時間常駐で Energy Impact 計測、README に数値を書けると理想 |

---

## 12. v1 Issue 分割案（8 個）

1. **Set up menu bar app skeleton with sandbox and CI**
   - 本文: LSUIElement なメニューバー常駐アプリの骨格を作る。App Sandbox 有効・ネットワーク entitlement なし。GitHub Actions で build + 禁止シンボル grep（privacy-guard）を回す。
   - 受け入れ条件: メニューバーに常駐し Quit できる / sandbox 有効 / privacy-guard ワークフローが green。
   - ラベル: `core`, `infra`, `privacy`

2. **Implement Enter-only key source with CGEventTap (listen-only)**
   - 本文: `EnterTapSource` を単一ファイル 100 行以内で実装。keyDown マスク限定、keyCode 36/76 判定のみ、引数なしトリガ通知を発火。tap 無効化検知と再有効化を含む。
   - 受け入れ条件: Enter でのみトリガ発火 / 他キーで副作用ゼロ / `// PRIVACY:` アンカーあり / tap 自動無効化から復帰する。
   - ラベル: `core`, `privacy`

3. **Build cat overlay window and animation state machine**
   - 本文: 透過 NSPanel（クリック透過・全 Space 表示）に Hidden→Rising→Clap→Linger→Retract の状態機械とスプライト描画を実装。手動トリガ API を持たせる。
   - 受け入れ条件: 状態遷移が §5 の図どおり / Hidden 時 CPU ~0% / フルスクリーン Space でも表示。
   - ラベル: `core`, `animation`

4. **Handle rapid Enter presses with re-clap and rate limiting**
   - 本文: 連打時に Clap 再生し直し + Linger リセット + 100ms レートリミット + Retract 中の復帰遷移を実装。内部連打カウンタ（v2 コンボ用）も用意。
   - 受け入れ条件: 連打中ネコが出ずっぱりで叩き続ける / 10+ 押/秒でもアニメーション破綻・CPU スパイクなし。
   - ラベル: `core`, `animation`

5. **Add onboarding flow and permission fallback (demo mode)**
   - 本文: 初回起動時の権限説明ウィンドウ（preflight → request → 設定ディープリンク → 許可検知）と、未許可時のデモモード（Clap now・⚠ バッジ）を実装。
   - 受け入れ条件: 未許可でもクラッシュ・モーダル連発なし / 許可後に再起動不要で動作開始 / プライバシー要約が表示される。
   - ラベル: `ux`, `privacy`

6. **Add settings menu and green screen mode for OBS**
   - 本文: メニューバーからディスプレイ / 位置 / サイズ / Launch at login / Green screen mode を設定可能にする。グリーンバック時は移動可能な不透明緑背景ウィンドウに切替。UserDefaults 永続化。
   - 受け入れ条件: 全設定が再起動後も保持 / OBS ウィンドウキャプチャ + クロマキーで抜き成功のスクショを Issue に添付。
   - ラベル: `ux`, `obs`

7. **Create release pipeline with codesign, notarization, and Homebrew cask**
   - 本文: GitHub Actions で Developer ID 署名 → notarytool → staple → Release 添付を自動化し、Homebrew Cask を作成。tag push で一気通貫。
   - 受け入れ条件: 素の macOS で dmg から起動して Gatekeeper 警告なし / アップデート後も Input Monitoring 権限が維持される / `brew install --cask keycat` 成功。
   - ラベル: `infra`, `release`

8. **Write README with prominent privacy section and demo GIF**
   - 本文: §10 の構成で英語 README を作成。バナー GIF・デモ GIF・1 行インストール・Privacy 表・ソースパーマリンク・codesign 検証手順・OBS 手順・Wayland 非対応の明記を含む。
   - 受け入れ条件: Privacy セクションが折りたたみなしで first view 付近にある / 全リンク有効 / privacy-guard バッジ表示。
   - ラベル: `docs`, `privacy`

---

## 参考資料（WebSearch 検証済み）

- CGEventTap は Input Monitoring、NSEvent global monitor は Accessibility を要求。CGEventTap には `CGPreflightListenEventAccess` / `CGRequestListenEventAccess` があり、10.15+ で sandbox 互換: [KeePassXC Issue #3393](https://github.com/keepassxreboot/keepassxc/issues/3393), [pqrs-org/osx-event-observer-examples](https://github.com/pqrs-org/osx-event-observer-examples), [Apple Developer Forums](https://developer.apple.com/forums/thread/707680)
- Windows WH_KEYBOARD_LL のタイムアウト（1709+ で最大 1000ms、超過時サイレント解除・検知不能）: [Microsoft Learn: LowLevelKeyboardProc](https://learn.microsoft.com/en-us/windows/win32/winmsg/lowlevelkeyboardproc), [SetWindowsHookExA](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowshookexa)
- Wayland はグローバルキー監視をコンポジタ仲介前提で原則禁止、X11 とはモデルが根本的に異なる: [wayland.app: keyboard-shortcuts-inhibit](https://wayland.app/protocols/keyboard-shortcuts-inhibit-unstable-v1), [Handy Issue #949](https://github.com/cjpais/Handy/issues/949), [rustdesk Discussion #5117](https://github.com/rustdesk/rustdesk/discussions/5117)
- OBS のウィンドウキャプチャはアルファチャンネルを保持できない（クロマキー等が現実解）: [OBS Forums: Window Capture Transparency](https://obsproject.com/forum/threads/window-capture-transparency-no-color-key.11781/)
- ad-hoc 署名はビルドごとに TCC 上の別アプリ扱いとなり権限が維持されない。Developer ID 署名で同一性が保たれる: [jano.dev: Accessibility Permission](https://jano.dev/apple/macos/swift/2025/01/08/Accessibility-Permission.html), [Apple Developer Forums: TCC](https://developer.apple.com/forums/thread/703188)
- rdev（Rust クロスプラットフォームフック、macOS は CGEventTap ベース）: [Narsil/rdev](https://github.com/Narsil/rdev)
