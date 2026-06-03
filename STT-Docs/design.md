# EchoNote（エコーノート）— 設計書

**バージョン**: 3.0  
**作成日**: 2026-04-19  
**ステータス**: 戦略整合版ドラフト

---

## 1. 設計方針

### 1.1 設計原則

EchoNote の設計は、次の4原則に従う。

1. **録音は絶対に落とさない**
価値の起点は音声データであるため、録音の堅牢性を最優先とする。

2. **AI出力は必ず根拠に紐付ける**
ノートは生成できるだけでなく、どの発話から生成されたかを追跡できる必要がある。

3. **資料と過去知識を積極活用する**
単発セッションではなく、継続利用で精度が上がる構造を持たせる。

4. **全文レビューを強いない**
ユーザーに全部を確認させず、怪しい箇所だけレビューさせる。

### 1.2 全体アーキテクチャ

```
┌─────────────────────────────┐         ┌──────────────────────────────────────────────────────────────┐
│  Apple Watch App            │         │                  iPhone App（Expo / React Native）            │
│  (SwiftUI / watchOS 10+)    │         │                                                              │
│                             │         │  Recording UI   Session UI   Note UI   Review Queue UI       │
│  RecordingManager           │         │       │            │           │             │                │
│  (AVAudioRecorder)          │         │       └────────────┴───────────┴─────────────┘                │
│  ConnectivityManager        │         │                           │                                  │
│  (WCSession.transferFile)   │         │   ┌───────────────────────▼───────────────────────────────┐   │
│         │                   │         │   │                  Application Layer (TS)               │   │
│         │  m4a + metadata   │         │   │ RecordingCoordinator   AutoPipeline                   │   │
│         │  (bookmarks,      │         │   │ SessionPipeline        WatchRecordings importer       │   │
│         │   duration)       │         │   │ NoteComposer / ReviewQueueBuilder                     │   │
│         │                   │         │   │ ContextMemoryManager                                  │   │
│         └────────────────────────────►│   └───────────────┬───────────────────┬───────────────────┘   │
│                             │ WC      │                   │                   │                       │
│                             │ Session │   ┌───────────────▼─────────┐ ┌───────▼──────────────────┐   │
└─────────────────────────────┘         │   │ Local Persistence (RN)  │ │ Native Bridges (Swift)   │   │
                                        │   │ Zustand + AsyncStorage  │ │ WatchBridge (WCSession)  │   │
                                        │   │ expo-file-system audio  │ │ AVFoundation 録音         │   │
                                        │   │ Session / Segment / Note│ │ VisionKit OCR             │   │
                                        │   │ Evidence / ReviewItem   │ │                           │   │
                                        │   └───────────────┬─────────┘ └───────────┬───────────────┘   │
                                        │                   │                       │                   │
                                        └───────────────────┼───────────────────────┼───────────────────┘
                                                            │                       │
                                                            │ HTTPS                 │ Local preview only
                                                            ▼                       ▼
                                                  ┌──────────────────────┐    ┌────────────────────────┐
                                                  │ Backend Proxy        │    │ Realtime Preview ASR    │
                                                  │ Auth / Rate Limit    │    │ SFSpeechRecognizer      │
                                                  │ Cost Control         │    │ 非保存・参考表示のみ     │
                                                  └──────┬─────────┬─────┘    └────────────────────────┘
                                                         │         │
                                                ┌────────▼───┐ ┌──▼─────────────┐
                                                │ OpenAI API │ │ Anthropic API  │
                                                │ ASR        │ │ Correction /   │
                                                │            │ │ Note / Terms   │
                                                └────────────┘ └────────────────┘
```

> 実装メモ：iPhone 本体は Expo（React Native）で構築している。Watch コンパニオンと、Watch からの受信を担う `WatchBridge` のみが Swift 製で、Expo の bare workflow としてネイティブビルドに組み込まれている。

---

## 2. 価値を生むコアパイプライン

### 2.1 Phase 1 の処理パイプライン

```
録音
  → 音声ファイル保存
  → 高精度ASR
  → セグメント化された原文保存
  → OCR + 用語抽出
  → 用語メモリ統合
  → セグメント単位補正
  → 根拠付きノート生成
  → レビューキュー生成
```

### 2.2 差別化ポイントがどこで生まれるか

| 段階 | 一般的なサービス | EchoNote |
|------|------------------|---------|
| 文字起こし | 全文化して終わり | セグメント + 時間情報を保持 |
| 資料活用 | OCRを添付するだけ | 用語抽出して補正へ反映 |
| 要約 | 本文から要約を生成 | 根拠参照付きノートを生成 |
| 品質確認 | 全文を人が確認 | 低信頼箇所だけレビュー |
| 学習 | セッションごとに完結 | 用語メモリが次回へ継承 |

---

## 3. 技術スタック

| レイヤー | 技術 | 理由 |
|----------|------|------|
| iPhone UI / アプリ層 | Expo (React Native) + expo-router | 単一コードベースで反復速度を確保し、将来の Android / Web 拡張に備える |
| 状態管理 | Zustand + AsyncStorage 永続化 | 軽量・型安全・永続化を1か所にまとめられる |
| 録音 | AVFoundation 経由（expo-av / native） | バックグラウンド録音に対応 |
| リアルタイム参考表示 | SFSpeechRecognizer | プレビュー専用で利用 |
| 録音後ASR | xAI API（Grok） | 長時間音声・マルチモーダル対応 |
| OCR | Vision / VisionKit | 標準フレームワークで十分 |
| AI補正 / 用語抽出 / ノート生成 | Anthropic API | 構造化出力と要約品質を活かす |
| ローカルDB | Zustand 永続化 + expo-file-system 音声ファイル | RN プロジェクトとの相性を優先 |
| Watch コンパニオン | SwiftUI + watchOS 10、AVAudioRecorder | Watch 単体での録音とブックマーク追加 |
| Watch ⇄ iPhone 通信 | WatchConnectivity（`transferFile`）+ Swift 製 `WatchBridge` シングルトン | iPhone がスリープでも OS が後で配信する信頼性のあるファイル転送 |
| バックエンドプロキシ | Cloudflare Workers | キー秘匿・レート制限・低運用コスト |
| 同期（Phase 3） | Firebase | Auth / Firestore / Storage をまとめて扱える |

---

## 4. 画面設計

### S01 ホーム

```
┌──────────────────────────────┐
│ EchoNote              [設定⚙] │
├──────────────────────────────┤
│ [🎙 録音を開始]              │
├──────────────────────────────┤
│ 最近のセッション              │
│ ┌────────────────────────┐   │
│ │ 📄 情報工学 第12回      │   │
│ │    52分 │ ✅ 完了       │   │
│ │    🔗 用語23件蓄積済み   │   │
│ └────────────────────────┘   │
│ ┌────────────────────────┐   │
│ │ 📄 マーケ勉強会         │   │
│ │    38分 │ ⏳ 補正中...  │   │
│ └────────────────────────┘   │
│ ┌────────────────────────┐   │
│ │ 🎙 ゼミ                │   │
│ │    ☁️ オフライン保留     │   │
│ └────────────────────────┘   │
└──────────────────────────────┘
```

- セッションカード：タイトル（編集可）・録音時間・処理状態・用語蓄積状況
- 処理状態は `SessionState` に連動
- シリーズ名（seriesKey）をカードに表示し、同シリーズの連続性を示す

### S02 録音画面

```
┌──────────────────────────────┐
│ [←戻る]  録音中...   [📄資料]│
├──────────────────────────────┤
│    ████ ▌▌▌██ ▌▌▌███        │
│         波形インジケーター     │
│         01:23:45             │
├──────────────────────────────┤
│ プレビュー（参考表示）        │
│ ┌────────────────────────┐   │
│ │...本日は機械学習の基礎  │   │
│ │について説明します...    │   │
│ └────────────────────────┘   │
│ ※録音後に高精度で処理します  │
├──────────────────────────────┤
│        [■ 停止]              │
└──────────────────────────────┘
```

- プレビューは SFSpeechRecognizer による参考表示（保存しない）
- 停止後は自動でパイプライン開始、S03へ遷移

### S03 セッション詳細

```
┌──────────────────────────────┐
│ [←] 情報工学 第12回  [テーマ]│
├──────────────────────────────┤
│ ✅ 完了  12ブロック  ⚠ 3件   │
├───────┬──────────┬───────────┤
│[原文] │ [補正後] │[ノート ●] │
├───────┴──────────┴───────────┤
│                               │
│ ┌─────────────────────────┐  │
│ │ 🔑 定義          [⚠高] │  │
│ │ 教師あり学習は入力と正解 │  │
│ │ ラベルの対応から学習する │  │
│ │ ─────────────────────── │  │
│ │ ▶ 1:23  "教師あり学習は │  │
│ │ ラベル付きデータから..." │  │
│ └─────────────────────────┘  │
│                               │
│ ┌─────────────────────────┐  │
│ │ 📐 公式                 │  │
│ │ 損失関数 L(θ) = Σ...    │  │
│ │ ─────────────────────── │  │
│ │ ▶ 8:45  "損失関数とは"  │  │
│ └─────────────────────────┘  │
│                               │
│ ┌─────────────────────────┐  │
│ │ ✅ 課題            🔴   │  │
│ │ 第3章の演習問題を解く   │  │
│ └─────────────────────────┘  │
│                               │
├──────────────────────────────┤
│ [📋 コピー] [↗ 共有] [🎙再生]│
└──────────────────────────────┘
```

- 3タブ：原文 / 補正後 / ノート（アクティブタブは ● で表示）
- 各ブロックカードに `displayIcon + displayLabel` を左上に表示
- 要レビューブロックはカード右上に `[⚠高/中/低]` バッジ表示
- 引用はカード内にインライン表示（▶ タイムスタンプ + 冒頭テキスト）
- ▶ タップで該当区間の音声をインラインミニプレイヤーで再生
- 下部アクションバー：コピー / 共有 / ポッドキャスト再生

### S04 ドキュメントスキャン

```
┌──────────────────────────────┐
│ [←] 資料スキャン             │
├──────────────────────────────┤
│   [VisionKit カメラUI]       │
├──────────────────────────────┤
│ スキャン済み: 3ページ         │
│ 抽出用語: 15件               │
│ [+ ページ追加] [✓ 完了]      │
└──────────────────────────────┘
```

### S05 ノート編集

```
┌──────────────────────────────┐
│ [←] ノート編集     [↗ 共有] │
├──────────────────────────────┤
│ 🔑定義 📐公式 ✅課題 …[絞込] │  ← ラベルフィルタチップ
├──────────────────────────────┤
│ ┌─────────────────────────┐  │
│ │ 🔑 定義  [編集✏]        │  │
│ │ 教師あり学習は...        │  │
│ │ ▶ 1:23                  │  │
│ └─────────────────────────┘  │
│ ┌─────────────────────────┐  │
│ │ 📐 公式  [編集✏]        │  │
│ │ L(θ) = Σ ...            │  │
│ │ ▶ 8:45                  │  │
│ └─────────────────────────┘  │
├──────────────────────────────┤
│ [🔄 再生成] [📋 コピー]      │
└──────────────────────────────┘
```
│ - ...             📎🔊       │
├──────────────────────────────┤
│ [🔄 再生成] [📋 コピー]     │
└──────────────────────────────┘
```

- 各ブロック右端に根拠アイコン（📎）と音声再生アイコン（🔊）
- Markdown編集モードと閲覧モードの切替

### S06 レビューキュー

```
┌──────────────────────────────┐
│ [←] レビュー対象  3/8件 確認済│
├──────────────────────────────┤
│ 🔴 優先度: 高                │
│ ┌────────────────────────┐   │
│ │ セグメント #14          │   │
│ │ "きょうしありがくしゅう"│   │
│ │ → 「教師あり学習」?     │   │
│ │ 理由: 用語と音が近いが  │   │
│ │       補正未適用        │   │
│ │ [✓ 確認済] [✏ 修正]    │   │
│ └────────────────────────┘   │
│ 🟡 優先度: 中                │
│ ┌────────────────────────┐   │
│ │ ノートブロック #3       │   │
│ │ 「回帰分析の精度は...」 │   │
│ │ 理由: 根拠セグメントなし│   │
│ │ [✓ 確認済] [✏ 修正]    │   │
│ └────────────────────────┘   │
└──────────────────────────────┘
```

- レビュー対象を優先度順に一覧表示
- 各アイテムに理由と推奨アクションを表示
- 確認済みマークで進捗管理
- 修正ボタンで該当セグメントまたはノートブロックへジャンプ

---

## 5. ドメインモデル

### 5.1 モデル全体像

EchoNote の中核は `Session` ではなく、**Session + Segment + Evidence + TermMemory + ReviewItem** の組み合わせである。

```swift
@Model
final class Session {
    var id: UUID
    var title: String
    var createdAt: Date
    var duration: TimeInterval
    var audioRelativePath: String?
    var state: SessionState
    var failureReason: String?
    var seriesKey: String?               // 同一シリーズ識別用（ユーザー設定 or AI提案）
    var pipelineAttemptId: UUID?        // 冪等性保証用

    @Relationship(deleteRule: .cascade)
    var segments: [TranscriptSegment]

    @Relationship(deleteRule: .cascade)
    var documents: [ScannedDocument]

    @Relationship(deleteRule: .cascade)
    var notes: [NoteDocument]

    @Relationship(deleteRule: .cascade)
    var reviewItems: [ReviewItem]
}

@Model
final class TranscriptSegment {
    var id: UUID
    var session: Session
    var index: Int
    var rawText: String
    var correctedText: String?
    var startTime: TimeInterval
    var endTime: TimeInterval
    var confidenceScore: Double?
    var needsReview: Bool
}

@Model
final class ScannedDocument {
    var id: UUID
    var session: Session
    var pageIndex: Int
    var imagePath: String
    var ocrText: String
}

@Model
final class TermMemoryEntry {
    var id: UUID
    var seriesKey: String
    var term: String
    var reading: String?
    var category: String
    var source: String                  // document / correction / user
    var isDisabled: Bool
    var lastUsedAt: Date
}

@Model
final class NoteDocument {
    var id: UUID
    var session: Session
    var type: NoteType
    var markdown: String

    @Relationship(deleteRule: .cascade)
    var blocks: [NoteBlock]
}

@Model
final class NoteBlock {
    var id: UUID
    var note: NoteDocument
    var order: Int
    var heading: String?
    var body: String
    var blockType: String               // 処理ロジック用固定値: summary / decision / concept / action / inference
    var displayLabel: String            // 表示用AI生成ラベル（例: "定義", "試験ポイント", "処方内容"）
    var displayIcon: String             // 表示用絵文字（例: "🔑", "📐", "✅"）
    var confidenceScore: Double?

    @Relationship(deleteRule: .cascade)
    var evidences: [EvidenceAnchor]
}

@Model
final class EvidenceAnchor {
    var id: UUID
    var block: NoteBlock
    var segmentIndex: Int
    var quote: String
    var startTime: TimeInterval
    var endTime: TimeInterval
}

@Model
final class ReviewItem {
    var id: UUID
    var session: Session
    var type: ReviewItemType            // low_confidence_segment / weak_evidence_block
    var targetId: UUID
    var reason: String
    var priority: Int
    var resolved: Bool
}

enum SessionState: String, Codable {
    case recording
    case recorded
    case transcribing
    case transcribed
    case correcting
    case corrected
    case noteGenerating
    case reviewBuilding
    case completed
    case failed
}

enum NoteType: String, Codable {
    case studyNote
    case meetingMemo
}

enum ReviewItemType: String, Codable {
    case lowConfidenceSegment
    case weakEvidenceBlock
}
```

### 5.2 なぜこのモデルが必要か

- `TranscriptSegment` を持つことで、発話区間へ戻れる
- `EvidenceAnchor` を持つことで、ノートに根拠を持たせられる
- `TermMemoryEntry` を持つことで、次回セッションに学習を持ち越せる
- `ReviewItem` を持つことで、全文レビューを回避できる

---

## 6. 主要コンポーネント設計

### 6.1 RecordingCoordinator

責務:

- AVAudioSession と AVAudioEngine の制御
- 録音ファイルの生成と終了処理
- バックグラウンド録音継続
- 任意でリアルタイムプレビュー開始

### 6.2 SessionPipeline

責務:

- `recorded -> transcribing -> correcting -> noteGenerating -> reviewBuilding` の遷移管理
- API呼び出しのオーケストレーション
- 冪等な再実行
- 失敗時の状態保持

### 6.3 ContextMemoryManager

責務:

- OCR由来の用語抽出
- seriesKey 単位の既存用語取得
- 補正ジョブへ渡す用語セットの統合
- 不要用語の無効化反映

seriesKey 割り当て戦略:

- ユーザーがセッション作成時にシリーズを手動設定（「情報工学」「マーケゼミ」等）
- 未設定の場合、ノート生成後にAIがタイトルからシリーズ名を提案し、ユーザーが承認する
- 同一 seriesKey のセッション間で TermMemoryEntry を共有する

### 6.4 NoteComposer

責務:

- 補正済みセグメント列からノートを生成
- AI出力を `NoteBlock` 単位へ変換
- 各ブロックに `EvidenceAnchor` を付与

### 6.5 ReviewQueueBuilder

責務:

- セグメント信頼度、未一致用語、根拠不足ブロックを評価
- レビュー対象を優先度付きで生成

### 6.6 WatchBridge（Swift シングルトン）

責務:

- アプリ起動時に `WCSession.default.activate()` を呼び、`WCSessionDelegate` として登録
- Watch から到着した音声ファイル（`didReceive(file:)`）を `Documents/watch-recordings/<UUID>.m4a` に移動
- 同フォルダに manifest JSON（`<UUID>.json`）を書き出す（duration / bookmarks / recordedAt / imported フラグ）
- React Native 側はこのフォルダを監視するだけでよい（ネイティブから JS への push は持たない）

実装メモ:

- `AppDelegate` の `didFinishLaunchingWithOptions` で `WatchBridge.shared.activate()` を呼ぶ
- `delegate` プロパティが weak のため、シングルトンとしてアプリ生存期間中保持する
- 受信ファイルは iOS が一時 Inbox に置くため、デリゲート復帰前に必ず移動する

### 6.7 WatchRecordings importer（`useWatchRecordings`）

責務:

- アプリ前面復帰／マウント時に `Documents/watch-recordings/` を走査
- `imported: false` の manifest をループ処理
  - 音声を `recordings/<sessionId>.m4a` にコピー
  - Zustand session ストアに `Watch録音` セッションを `recorded` 状態で作成
  - ブックマーク・所要時間・recordedAt を反映
  - manifest を `imported: true` で書き戻す
- 失敗した manifest はスキップして次へ進む（ログに警告）

### 6.8 AutoPipeline（`useAutoPipeline`）

責務:

- 起動時／ストア更新時／前面復帰時に「`recorded` 状態のセッション」を走査
- 1件ずつ順次 `runPipeline` を起動（`runningRef` で並列実行を抑止）
- 失敗してもループは継続（pipelineService が `failed` に遷移する）

設計判断:

- 並列実行しない理由：API レート制限・端末発熱・電池消費の抑制
- `failed` 状態は対象外（自動再試行で誤動作を増やさない）
- セッション詳細画面の「開いた瞬間に自動起動する useEffect」と二重に発火する瞬間がありうるが、`pipelineService` が状態遷移を `transitionState` で制御しているため、二人目はそのまま no-op になる

---

## 7. 主要フロー

### 7.1 録音からノート完成まで

```mermaid
sequenceDiagram
    participant U as User
    participant App as iOS App
    participant Pipe as SessionPipeline
    participant Proxy as Backend Proxy
    participant ASR as ASR API
    participant AI as Anthropic API
    participant DB as SwiftData

    U->>App: 録音開始
    App->>DB: Session(state=recording)
    U->>App: 録音停止
    App->>DB: Session(state=recorded)

    App->>Pipe: startPipeline(sessionId)
    Pipe->>DB: state=transcribing
    Pipe->>Proxy: /v1/transcribe(audio)
    Proxy->>ASR: transcription request
    ASR-->>Proxy: segments
    Proxy-->>Pipe: segments
    Pipe->>DB: TranscriptSegment 保存
    Pipe->>DB: state=transcribed

    Pipe->>Proxy: /v1/extract-terms(ocrText + seriesKey)
    Proxy->>AI: terms extraction
    AI-->>Proxy: terms
    Proxy-->>Pipe: terms

    Pipe->>DB: state=correcting
    Pipe->>Proxy: /v1/correct(segments + terms)
    Proxy->>AI: correction
    AI-->>Proxy: corrected segments
    Proxy-->>Pipe: corrected segments
    Pipe->>DB: correctedText 保存
    Pipe->>DB: state=corrected

    Pipe->>DB: state=noteGenerating
    Pipe->>Proxy: /v1/generate-note(corrected segments + terms)
    Proxy->>AI: structured note generation
    AI-->>Proxy: note blocks + evidence
    Proxy-->>Pipe: structured note
    Pipe->>DB: NoteDocument / NoteBlock / EvidenceAnchor 保存

    Pipe->>DB: state=reviewBuilding
    Pipe->>Pipe: buildReviewQueue()
    Pipe->>DB: ReviewItem 保存
    Pipe->>DB: state=completed
```

### 7.2 用語メモリ統合フロー

```mermaid
flowchart TD
    A[OCR text] --> B[AIで用語抽出]
    C[seriesKeyに紐づく既存用語] --> D[用語統合]
    B --> D
    D --> E[重複除去]
    E --> F[補正プロンプトへ投入]
    F --> G[補正後に新規有効用語を更新]
```

### 7.3 Apple Watch 録音から取り込みまで

```mermaid
sequenceDiagram
    participant U as User
    participant W as Apple Watch App
    participant WC as WatchConnectivity (OS)
    participant WB as iPhone WatchBridge
    participant Imp as useWatchRecordings
    participant AP as useAutoPipeline
    participant P as SessionPipeline

    U->>W: 録音開始 / ブックマーク追加 / 停止
    W->>W: AVAudioRecorder で m4a を保存
    W->>WC: transferFile(audioURL, metadata={duration, bookmarks, recordedAt})
    Note over WC: iPhone がスリープでも OS が後で配信
    WC-->>WB: didReceive(file)（iPhone アプリ次回起動時）
    WB->>WB: Documents/watch-recordings/<UUID>.m4a + manifest.json を保存
    U->>WB: iPhone EchoNote を起動 / 前面復帰
    Imp->>WB: watch-recordings/ を走査
    Imp->>Imp: recordings/<sessionId>.m4a へコピー、'recorded' セッション作成
    AP->>AP: 'recorded' セッションを順次検出
    AP->>P: runPipeline(sessionId)
    P-->>U: 文字起こし→補正→ノート→レビュー（自動）
```

ポイント:

- Watch から iPhone への配信は `transferFile` で OS が保証する。リアルタイム接続不要
- 取り込み（importer）と自動処理（AutoPipeline）は分離されている。importer 不調でもパイプラインは止まらず、また既存の手動録音も同じ AutoPipeline で救われる
- Watch 側の UI は「転送中…」を表示し、`didFinish` で「転送完了 ✓」に切り替える

### 7.4 レビューキュー生成ルール

レビュー対象は以下の複合条件で作る。

- ASR / 補正信頼度が閾値未満
- 用語メモリに近いが未一致な語が存在
- ノートブロックに evidence が 0 件
- 1ブロックあたりの evidence 数が少なすぎる
- 引用と本文の意味距離が大きい

---

## 8. API設計

### 8.1 プロキシの責務

- APIキー隠蔽
- デバイス認証
- レート制限
- コスト制御
- レスポンスの正規化
- 将来のモデル差し替え吸収

### 8.2 エンドポイント

| Method | Path | AIモデル | 入力 | 出力 |
|--------|------|---------|------|------|
| POST | `/v1/transcribe` | Grok（xAI） | 音声ファイル | `segments[]`（timestamps付き） |
| POST | `/v1/diarize` | AssemblyAI / Deepgram | 音声ファイル | 話者ラベル付き `segments[]` ※話し合いモード時のみ |
| POST | `/v1/extract-terms` | claude-haiku-4-5 | OCRテキスト, seriesKey | `terms[]` |
| POST | `/v1/correct` | claude-haiku-4-5 | raw segments, terms | `correctedSegments[]` |
| POST | `/v1/generate-note` | claude-sonnet-4-6 | corrected segments, noteType, terms | `noteBlocks[]` + `evidences[]` |

**レビューキュー生成はローカル処理**（6.5 ReviewQueueBuilder）で行い、API呼び出しは不要。

### 8.3 APIレスポンス例

#### `/v1/generate-note`

```json
{
  "noteBlocks": [
    {
      "order": 1,
      "heading": "教師あり学習",
      "body": "教師あり学習は入力と正解ラベルの対応から学習する手法。",
      "blockType": "concept",
      "displayLabel": "定義",
      "displayIcon": "🔑",
      "confidenceScore": 0.91,
      "evidences": [
        {
          "segmentIndex": 12,
          "quote": "教師あり学習はラベル付きデータから学ぶ手法です",
          "startTime": 120.4,
          "endTime": 126.8
        }
      ]
    }
  ]
}
```

### 8.4 冪等性

- 各セッションに `pipelineAttemptId` を持たせる
- 同一段階の再試行では重複保存を避ける
- `generate-note` は既存ノートの上書きか版管理かを明示する

---

## 9. プロンプト設計

### 9.1 用語抽出プロンプト

**目的**: OCR全文をそのまま渡さず、補正に効く語彙だけへ圧縮する

```
System:
あなたは資料テキストから専門用語・固有名詞を抽出する専門AIです。
以下のOCRテキストから、音声文字起こしの補正に役立つ用語を抽出してください。

出力形式（JSON）:
{
  "terms": [
    {"term": "表記", "reading": "カタカナ読み", "category": "専門用語|人名|組織名|略語|数値"}
  ]
}

抽出ルール:
- 一般的な日本語語彙は除外し、誤認識されやすい用語のみ抽出する
- 英語の専門用語はそのまま抽出する
- 略語がある場合は正式名称を term に、略語を reading に記載する
- 読みが自明でない用語のみ reading を付与する
```

```
Human:
以下の資料テキストから用語を抽出してください:

{ocrText}
```

**モデル**: `claude-haiku-4-5-20251001`

### 9.2 補正プロンプト

**目的**: raw segment を意味改変せずに修正する

```
System（キャッシュ対象）:
あなたは音声文字起こしの補正専門AIです。
以下の用語リストを参考に、文字起こしセグメントの誤認識を修正してください。

【参考用語リスト】
{terms を "term (reading) [category]" 形式で列挙}

補正ルール:
- 用語リストに含まれる専門用語・固有名詞・数字を優先して使用する
- 文脈から明らかな誤認識のみ修正し、意味のある言い回しは変えない
- 話し言葉のスタイルを維持する
- セグメントの分割は変更しない
- 各セグメントに対して補正後テキストと信頼度スコア（0.0〜1.0）を返す
- 信頼度が低い箇所（曖昧な修正）は 0.7 未満とする

出力形式（JSON）:
{
  "segments": [
    {"index": 0, "correctedText": "...", "confidenceScore": 0.95},
    {"index": 1, "correctedText": "...", "confidenceScore": 0.62}
  ]
}
```

```
Human（キャッシュ非対象）:
以下の文字起こしセグメントを補正してください:

{segments を index, rawText, startTime, endTime で列挙}
```

**モデル**: `claude-haiku-4-5-20251001`  
**キャッシュ戦略**: Systemプロンプト（用語リスト含む）に `cache_control: {"type": "ephemeral"}` を設定。チャンク分割時にキャッシュヒット。  
**チャンク分割**: 合計約4,000文字を超える場合、セグメント境界で分割して複数回呼び出す。

### 9.3 根拠付きノート生成プロンプト

**目的**: ノート本文だけでなく、ブロックごとの evidence を生成する

```
System（キャッシュ対象）:
あなたは講義・会議の内容を構造化するAIアシスタントです。
補正済みセグメント列から、根拠付きのノートを生成してください。

【テンプレート: {noteType}】
- studyNote: トピック → 主要概念 → 詳細説明 → まとめ → キーワード
- meetingMemo: 議題 → 決定事項 → 未決事項 → 次のアクション

【参考用語リスト】
{terms}

必須ルール:
- 各ブロックに最低1件の evidence（元セグメントからの引用）を付与する
- evidence の quote は元セグメントの原文をそのまま引用する（50文字以内）
- evidence の segmentIndex, startTime, endTime は元セグメントと一致させる
- 元発話で確認できない推論や補足は blockType を "inference" とし、confidenceScore を 0.5 以下にする
- 根拠が十分なブロックは confidenceScore を 0.8 以上にする
- displayLabel はこのブロックの内容を表す具体的な日本語ラベルを10文字以内で生成する
  （例: 医療なら"診断名""処方内容", 講義なら"定義""試験ポイント", 会議なら"決定事項""宿題"）
- displayIcon はブロック内容に合う絵文字を1文字生成する
- displayLabel / displayIcon はセッションのテーマ・文脈に合わせて毎回新しく生成する（固定値にしない）

出力形式（JSON）:
{
  "noteBlocks": [
    {
      "order": 1,
      "heading": "見出し",
      "body": "本文（Markdown）",
      "blockType": "concept|decision|summary|action|inference",
      "displayLabel": "定義",
      "displayIcon": "🔑",
      "confidenceScore": 0.91,
      "evidences": [
        {
          "segmentIndex": 12,
          "quote": "元セグメントからの引用",
          "startTime": 120.4,
          "endTime": 126.8
        }
      ]
    }
  ]
}
```

```
Human（キャッシュ非対象）:
以下の補正済みセグメントから{noteType}形式のノートを生成してください:

{correctedSegments を index, correctedText, startTime, endTime で列挙}
```

**モデル**: `claude-sonnet-4-6`

### 9.4 レビューキュー生成（ローカル処理）

レビューキューはAPI呼び出しではなく、ローカルのルールベースで生成する。

**レビュー対象の判定ルール**:

| ルール | 優先度 | 対象 |
|--------|--------|------|
| セグメント confidenceScore < 0.7 | 高 | lowConfidenceSegment |
| ノートブロックに evidence が 0 件 | 高 | weakEvidenceBlock |
| ノートブロック confidenceScore < 0.6 | 中 | weakEvidenceBlock |
| ノートブロック blockType が "inference" | 中 | weakEvidenceBlock |
| 用語メモリに近いが未一致な語が存在 | 低 | lowConfidenceSegment |

---

## 10. ローカルデータ戦略

### 10.1 保存単位

| データ | 保存先 |
|--------|--------|
| 音声ファイル | Documents 配下 |
| OCR画像 | Documents 配下 |
| 構造化データ | SwiftData |

### 10.2 保存ポリシー

- 音声は相対パスで保持し、移動耐性を持たせる
- ノートは `markdown` と `blocks` の両方を保持する
- evidence はブロック側にネストせず独立モデルで持ち、後から再構築しやすくする

---

## 11. 音声再生設計（Evidence Playback）

### 11.1 概要

ノートブロックの根拠引用（EvidenceAnchor）から元音声の該当区間を再生する機能。EchoNote の「信頼できるノート」体験の中核。

### 11.2 再生仕様

| 項目 | 仕様 |
|------|------|
| 再生区間 | `startTime - 2秒` 〜 `endTime + 2秒`（前後にバッファ） |
| 再生速度 | 1.0x / 1.5x / 2.0x 切替可 |
| UI | インライン再生（ミニプレイヤー） |
| 操作 | タップで再生/停止、スワイプで前後セグメントへ |

### 11.3 実装方針

```swift
class EvidencePlayer {
    private let audioEngine = AVAudioPlayerNode()

    func play(evidence: EvidenceAnchor, audioPath: String) {
        let start = max(0, evidence.startTime - 2.0)
        let end = evidence.endTime + 2.0
        // AVAudioFile のフレーム位置を計算し、区間再生
    }
}
```

- 音声ファイルは端末内に保持されているため、ネットワーク不要
- 複数の evidence を連続再生するモード（セッション通し聴き）も将来対応可

---

## 12. 処理完了通知

### 12.1 概要

録音停止後のパイプライン処理（文字起こし → 補正 → ノート生成）はバックグラウンドで数分かかる。ユーザーが他の作業をしている間に処理が完了したことをプッシュ通知で伝える。

### 12.2 通知タイミング

| イベント | 通知内容 |
|----------|---------|
| ノート生成完了 | 「📝 [セッション名] のノートが完成しました」 |
| レビュー対象あり | 「⚠ [N]件の確認事項があります」（ノート完了通知に併記） |
| 処理失敗 | 「❌ [セッション名] の処理でエラーが発生しました」 |

### 12.3 実装方針

- `UNUserNotificationCenter` によるローカル通知
- パイプライン完了時に `SessionPipeline` から通知をスケジュール
- アプリがフォアグラウンドの場合はバナー表示のみ

---

## 13. バックグラウンドと障害対応

### 13.1 録音

- `UIBackgroundModes: audio` を利用
- 端末ロック中の録音継続を前提設計
- 割り込み時はセグメント境界を記録

### 13.2 オフラインキュー

```swift
final class OfflinePipelineQueue {
    func enqueueTranscription(for sessionId: UUID) {}
    func resumePendingJobs() async {}
    func markFailed(sessionId: UUID, reason: String) {}
}
```

### 13.3 リトライポリシー

| 失敗種別 | リトライ |
|----------|---------|
| ネットワークエラー | 3回 |
| 429 | Retry-After 優先 |
| 5xx | 2回 |
| 4xx | リトライしない |

---

## 14. セキュリティ設計

### 14.1 基本方針

- クライアントに秘密情報を置かない
- 録音データは必要最小限の送信に留める
- Phase 1 では個人利用でも安心して使えることを優先する

### 14.2 具体策

- APIキーはバックエンドプロキシのみ保持
- 通信は HTTPS（TLS 1.3）前提
- Phase 1 はデバイスUUID + バンドルIDのHMAC署名で簡易認証
- Phase 3 で Firebase Auth トークンベースへ移行
- 音声ファイルはASR API以外には送信しない（AI補正にはテキストのみ）

---

## 15. Firestore 設計（Phase 3）

### 15.1 コレクション

```
/users/{userId}
/projects/{projectId}
/projects/{projectId}/members/{userId}
/projects/{projectId}/sessions/{sessionId}
/projects/{projectId}/terms/{termId}
```

### 15.2 方針

- `memberIds` はルール判定用に保持
- 詳細ロールは `members` サブコレクションで管理
- セッション共有時は note, review, term memory を含める

---

## 16. コスト試算

### 16.1 前提条件

- 1セッション = 平均60分
- 文字起こし平均文字数: 約12,000文字（200文字/分）
- 日本語トークン換算: 1文字 ≈ 1.5トークン
- OCRテキスト: 約3,000文字（A4×5枚）
- 補正チャンク分割: 3チャンク

### 16.2 セッションあたりコスト

| 処理 | モデル | 入力トークン | 出力トークン | コスト |
|------|--------|-------------|-------------|--------|
| 音声文字起こし | Grok（xAI） | — | — | 要確認（Whisper比較中） |
| 用語抽出 | Haiku 4.5 | ~5,000 | ~1,000 | ~$0.01 |
| 補正（3チャンク） | Haiku 4.5 | ~20,500 | ~18,000 | ~$0.10 |
| ノート生成 | Sonnet 4.6 | ~22,000 | ~3,000 | ~$0.11 |
| **合計** | | | | **~$0.58** |

### 16.3 月額試算（1ユーザー・週2回）

| パターン | 月額 |
|----------|------|
| フル利用 | ~$4.6 |

### 16.4 コストの支配要因と抑制策

- **ASR コストが支配的**（従来 Whisper は全体の62%）。Grok 移行後に再計測予定
- コスト抑制策:
  - OCR全文ではなく用語抽出結果を補正へ渡す（トークン削減）
  - セグメントを適切にチャンク化しキャッシュヒット率を上げる
  - ノート生成は構造化JSONで冗長文を減らす
  - 話し合いモード時のみ話者分離API（AssemblyAI / Deepgram）を追加呼び出し

---

## 17. 検証戦略

### 17.1 技術検証

| # | 項目 | 方法 | 合格基準 |
|---|------|------|----------|
| 1 | 60分録音 | 実機で5回 | ファイル破損なし |
| 2 | 90分録音 | 実機で3回 | アプリクラッシュなし |
| 3 | バックグラウンド録音 | 画面オフ→30分後確認 | 録音継続・ファイル完全 |
| 4 | 文字起こし速度 | 60分音声で計測 | 10分以内 |
| 5 | OCR精度 | 資料10枚 | 認識率 > 95% |
| 6 | オフライン→復帰 | 機内モードで録音→復帰 | 自動で処理完走 |

### 17.2 価値検証

| # | 項目 | 方法 | 合格基準 |
|---|------|------|----------|
| 1 | 根拠付きノート | ノートブロック検査 | 90%以上に evidence 存在 |
| 2 | 資料活用効果 | 同一音声で資料あり/なし比較 | 資料ありのほうがWER低い |
| 3 | 補正精度（資料あり） | WER計測（20サンプル） | 補正前比 30%以上改善 |
| 4 | レビューキュー有効性 | 実修正箇所との照合 | 70%以上を先回り検出 |
| 5 | 手修正時間 | 被験者5名で計測 | 平均5分以内 |
| 6 | 用語メモリ効果 | 同シリーズ2回目セッションの精度 | 1回目より改善 |

### 17.3 UX検証

| # | 項目 | 方法 | 合格基準 |
|---|------|------|----------|
| 1 | 初回録音開始 | 初見ユーザー5名 | 30秒以内に開始 |
| 2 | ノート信頼感 | 5段階評価アンケート | 平均4.0以上 |
| 3 | 根拠再生体験 | 被験者に引用タップを促す | 「元音声が聞けて安心」の回答70%以上 |
| 4 | レビューキューUX | 被験者にレビュー操作を依頼 | 「全文見直しより楽」の回答80%以上 |
| 5 | 処理待ち体験 | パイプライン処理中の行動観察 | 通知を受けて再開できる |

---

## 18. 実装優先順位

### 18.1 先に作るもの

1. 録音の堅牢化（バックグラウンド・割り込み対応）
2. 録音後ASR（Whisper API連携）
3. セグメント保存（タイムスタンプ付き）
4. OCR + 用語抽出
5. 用語メモリ（ローカル）
6. AI補正（信頼度スコア付き）
7. 根拠付きノート生成
8. Evidence Playback（音声区間再生）
9. レビューキュー
10. 処理完了通知

### 18.2 後回しにするもの

1. 高度なリアルタイム表示
2. 派手な差分可視化
3. クラウド同期
4. チーム共同編集

---

## 19. ノートUI設計

### 19.1 ブロックカードのビジュアル設計

各ノートブロックをカード形式で表示し、`blockType` に応じた色とアイコンで視覚的に区別する。

#### カード構造

```
┌─────────────────────────────────┐
│ {displayIcon} {displayLabel}  [⚠️]│  ← ラベル行（要レビュー時はバッジ表示）
│─────────────────────────────────│
│ {body テキスト}                  │  ← 本文（2〜3行で折りたたみ、タップで展開）
│─────────────────────────────────│
│ ▶ {timestamp}  "{quote冒頭}..."  │  ← 根拠引用（タップで音声再生）
└─────────────────────────────────┘
```

#### blockType 別カラーリング

| blockType | 左ボーダー色 | ラベル背景 |
|---|---|---|
| concept | 青 `#4A90D9` | 薄青 |
| decision | 緑 `#27AE60` | 薄緑 |
| action | オレンジ `#E67E22` | 薄オレンジ |
| summary | 紫 `#8E44AD` | 薄紫 |
| inference | グレー `#95A5A6` | 薄グレー（点線ボーダー） |

#### 信頼度インジケーター

- confidenceScore ≥ 0.8：表示なし
- 0.6 〜 0.8：カード右上に `⚠️ 中` バッジ（黄）
- < 0.6：カード右上に `⚠️ 高` バッジ（赤）

### 19.2 フィルタ・ナビゲーション

#### ラベルフィルタチップ（ノート画面上部）

```
[全て] [🔑定義] [📐公式] [✅課題] [💡考察] …
```

- セッション内に存在する `displayLabel` を自動でチップ生成
- タップで該当ブロックのみ表示（複数選択可）

#### ブロック数サマリー（タブ直下）

```
12ブロック  •  要レビュー 3件  •  用語 23件
```

### 19.3 根拠引用のインタラクション

#### 通常状態

```
▶ 1:23  "教師あり学習はラベル付きデータ..."
```

#### タップ後（インラインプレイヤー展開）

```
┌─────────────────────────────┐
│ ◀◀  ▶▌  ▶▶   [×]  1.0x    │
│ ████████░░░░░░░░  1:23/1:30 │
│ "教師あり学習はラベル付きデータ │
│  から学ぶ手法です"           │
└─────────────────────────────┘
```

- 前後2秒バッファ込みで再生
- 速度切り替え：1.0x / 1.5x / 2.0x
- 別のブロックをタップすると自動で閉じる

### 19.4 スワイプアクション

| 操作 | アクション |
|---|---|
| カードを右スワイプ | ✅ レビュー済みにする |
| カードを左スワイプ | ✏️ 編集モードへ |
| カードを長押し | 順序変更モード（ドラッグ） |

### 19.5 ラベルの表示例（セッション文脈別）

| セッション | displayLabel 例 |
|---|---|
| 大学講義（情報工学） | 定義 / 公式 / 例題 / 試験ポイント / まとめ |
| 内科診察 | 主訴 / 診断名 / 処方内容 / 次回予約 / 注意事項 |
| 商談 | 現状課題 / 提案内容 / 決定事項 / 持ち越し / 担当アクション |
| 保護者会 | 学校からの連絡 / 子どもの様子 / お願い事項 / 次回の予定 |

---

## 20. まとめ資料 アーキテクチャ（HTML テンプレート方式）

### 20.1 概要

外部ツール依存を避け、アプリ完結で美しいまとめ資料を生成する。
Anthropic がコンテンツを生成し、カスタム HTML/CSS テンプレートで描画、WKWebView 経由で PDF 化する。

### 20.2 生成パイプライン

```
NoteBlocks（SwiftData）
  → /v1/generate-summary（Anthropic claude-sonnet）
      → 構造化 JSON（テンプレート型に応じたスキーマ）
  → TemplateRenderer（ローカル）
      → HTML + インライン CSS
  → WKWebView レンダリング
      → プレビュー確認
  → WKWebView.createPDF()
      → PDF ファイル出力 / 共有シート
```

### 20.3 テンプレート別 HTML 設計方針

| テンプレート | レイアウト特徴 | フォント | カラー |
|---|---|---|---|
| 学習ノート | 章立て・余白多め | Noto Serif JP | ネイビー × ホワイト |
| 議事メモ | 表・バッジ・二色 | Noto Sans JP | グリーン × グレー |
| ZINE | 大見出し・コラム組 | 明朝 + ゴシック混合 | 自由（生成時に選択） |
| プレゼン | 1ブロック = 1スライド領域 | Bold Sans | テーマカラー自動選択 |
| 論文 | 二段組・参照番号 | Noto Serif JP | ブラック × ホワイト |

### 20.4 HTMLテンプレートの管理

- テンプレートファイルはアプリバンドル内に同梱（`/templates/*.html`）
- CSS 変数でテーマカラーを切り替え
- フォントは Google Fonts CDN ではなくバンドル内に含める（オフライン対応）

### 20.5 コスト

- /v1/generate-summary：claude-sonnet 使用、入力 ~20,000 token → 約 $0.09/回
- PDF 変換はローカル処理のためコスト追加なし

---

## 21. 話し合いモード設計（話者分離）

### 19.1 概要

2名の話者の録音を分析し、双方の主張を中立的に整理して落としどころを提示する機能。
カップル・夫婦の話し合い、家族会議、ビジネス交渉など、法人サービスが参入しにくい個人ユースケースを対象とする。

### 19.2 話者分離アーキテクチャ

```
音声ファイル
  ├─ Grok（xAI）         → 全文文字起こし + タイムスタンプ付きセグメント
  └─ AssemblyAI / Deepgram → 話者ラベル付きセグメント（話者A / 話者B）

  → タイムスタンプで突合して「誰が何を言ったか」を統合
  → 登録済み声紋 embedding と照合して「自分 / 相手」にラベル付け
  → Anthropic（claude-sonnet）→ 中立分析・落としどころ生成
```

### 19.3 声紋登録（Voice Enrollment）

- オンボーディング時に30〜60秒のサンプル音声を録音
- 声の特徴量（speaker embedding）を端末ローカルに保存
- 話者分離後のクラスタと照合し、近い方を「自分」として自動ラベリング
- 実装候補：resemblyzer / ECAPA-TDNN ベースの CoreML モデル（将来）、または AssemblyAI の話者照合機能

### 19.4 精度改善のための制約

| 課題 | 対策 |
|---|---|
| 机上スマホ録音での音量差 | 録音開始時に「スマホを中央に置く」ガイダンス表示 |
| 短い発話の連続 | 話者分離 API は 2-speaker 制約を明示して呼び出す |
| 室内反響 | ノイズ除去（RNNoise等）を前処理として挟む |

### 19.5 出力フォーマット

```markdown
## 自分の主張
- ...（根拠引用付き）

## 相手の主張
- ...（根拠引用付き）

## AI による中立分析
- 共通認識: ...
- 認識の相違点: ...
- 落としどころの提案: ...
```

### 19.6 倫理・安全設計

- 録音開始時に「相手の同意を得てください」という同意確認モーダルを表示
- DV等の被害が疑われるキーワード検出時は相談窓口リンクを表示
- 音声データは通常セッションと同様にローカル保存・外部送信はテキストのみ

---

## 20. LINE AI仲裁 設計

### 20.1 概要

話し合いモードのまとめ資料をインプットとして、LLMが双方とLINEで個別に会話し解決案を提示する機能。
相手はアプリ不要、LINE公式アカウントへのメッセージのみで参加できる。

### 20.2 アーキテクチャ

```
[iOS App]
  まとめ資料生成
  → 「AIに仲裁を依頼する」タップ
  → POST /v1/mediation/create
       { summaryId, userAId }
  ← { sessionId, secret }

[Cloudflare Workers + KV]
  sessions/{sessionId} = {
    summary: "...",       // まとめ資料テキスト
    secret: "...",        // HMAC署名で検証
    threadA: [],          // User A との会話履歴
    threadB: [],          // User B との会話履歴
    status: "waiting_b" | "active" | "drafting" | "done",
    expiresAt: timestamp
  }

[LINE Messaging API → Webhook → Cloudflare Workers]
  受信メッセージ形式: "{sessionId} {secret}"（初回認証）
  → sessionId に紐づくスレッドを特定
  → LLM（Anthropic claude-sonnet）と会話継続

[解決フロー]
  双方の会話が収束
  → /v1/mediation/draft でAIが解決案生成
  → 両者に確認メッセージ送信
  → 両者承認 → /v1/mediation/close → 同一サマリーを双方に送信
```

### 20.3 セッション状態遷移

```
waiting_b   : User A が作成済み、User B 未参加
active      : 双方参加、個別会話中
drafting    : AIが解決案ドラフト中、両者の確認待ち
done        : 合意完了
expired     : 有効期限（72時間）切れ
```

### 20.4 LLMへのコンテキスト設計

```
System（キャッシュ対象）:
あなたは中立的な対話仲裁者です。
以下の話し合いの記録を踏まえて、{role}（自分 / 相手）と個別に会話し、
言い分を丁寧に聞いてください。相手の発言は開示しないでください。

【話し合いの記録】
{summary}

ルール:
- 一方の肩を持たず、双方の言い分を等しく尊重する
- 感情を否定せず、まず受け止める
- 20ターンを超えたら解決案のドラフトを提案する

Human（キャッシュ非対象）:
{user_message}
```

### 20.5 コスト・インフラ

| 項目 | 内容 |
|---|---|
| LINE公式アカウント | ライトプラン（¥5,000/月）+ ¥3/通（500通超） |
| Cloudflare Workers KV | セッションデータ保存（TTL 72時間で自動削除） |
| Anthropic API | 1セッション最大40ターン × 約$0.02 = 最大約$0.80 |
| 1仲裁セッションの目安コスト | ~$1.00〜1.50（LINE費用含む） |

### 20.6 セキュリティ設計

- secret は `HMAC-SHA256(sessionId + userId, serverSecret)` で生成
- LINE Webhook の署名検証（`X-Line-Signature`）を必ず実施
- セッション終了後72時間でKVデータを自動削除
- まとめ資料テキストはKV保存時にAES-256で暗号化

---

## 21. 新機能アイデアと評価（2026-05）

会話の中で検討・評価した機能アイデアの記録。

| 機能 | 評価 | 優先度 | 判断理由 |
|---|---|---|---|
| 話し合いモード（痴話喧嘩モード） | ★★★★★ | 高 | 法人不参入領域・バズりやすい・ユーザー声紋登録でNotta等との明確な差別化 |
| 自己分析（ジャーナリング統合） | ★★★★ | 中 | 用語メモリと同じ「継続利用で精度向上」設計。リテンション効果が高い |
| ジャーナリング | ★★★ | 中 | 競合多いが「引用タップで自分の声が再生される」体験で差別化可能 |
| ポッドキャスト生成 | ★★★ | 低 | Phase 5 として既定路線（要件定義書 機能13） |
| ディープリサーチ付与 | ★★ | 対象外 | Perplexity・Gemini・ChatGPT と真正面衝突。開発コスト対比で優位性なし |

### 20.1 GPS文脈先読み（アイデア）

録音開始時にGPSで現在地の施設カテゴリを取得し、用語セットを自動ロードする構想。

| 場所カテゴリ | 自動で起動する用語セット |
|---|---|
| 大学キャンパス | 学術用語、seriesKey 候補提示 |
| 病院 | 医療用語セット |
| 弁護士事務所 | 法律用語セット |
| オフィスビル | 会議定型フレーズ、同行者名 |

カレンダー連携（会議タイトル・参加者名を補正用語に自動追加）の方が実装コストが低く即効性が高いため、GPS連携より先に検討する。

