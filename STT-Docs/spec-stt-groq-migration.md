# EchoNote — STT プロバイダ Groq 移行 仕様書

**バージョン**: 1.0
**作成日**: 2026-06-04
**ステータス**: ドラフト（実装着手前）
**関連**: `design.md` v3.0 §3 技術スタック / §16 コスト試算, `CLAUDE.md` Transcription: engine routing

---

## 1. 概要と目的

### 1.1 提案

`/v1/transcribe` の **>=20秒録音パス** で使用している OpenAI Whisper
(`whisper-1`) を、**Groq Whisper Large V3 Turbo** に置き換える。

### 1.2 動機

直近の TestFlight 検証 (2026-06-01〜02) を経て、現在の STT ルーティング
は以下に確定している:

| 録音長 | エンジン | 単価 | 備考 |
|---|---|---|---|
| < 20 秒 | xAI `grok-stt` | 約 \$0.10/h | 30秒以上で truncate するため短尺専用 |
| >= 20 秒 | OpenAI `whisper-1` | \$0.36/h | 真の長尺対応 |

OpenAI Whisper の単価が他より高く、Premium 100人想定で月 ~\$150 を占める
最大コスト要因になっている。**Groq Whisper Large V3 Turbo** は同じ
Whisper 系モデルを **\$0.04/h で提供** しており、API も OpenAI 互換
(`/openai/v1/audio/transcriptions`) なので proxy 改修もごく軽微。

### 1.3 期待効果

| 指標 | 現状 (OpenAI Whisper) | 移行後 (Groq Whisper) | 改善 |
|---|---|---|---|
| 単価 | \$0.36/h | **\$0.04/h** | **9倍安** |
| Premium 500h/月 想定月コスト | \$180 | **\$20** | -\$160/月 |
| レイテンシ (60秒録音) | 5〜15秒 | **約0.3秒** (228x realtime) | 体感大幅改善 |
| 文字起こし精度 (WER) | 約 10%（多言語） | 約 12%（多言語） | 同等〜やや劣化 |
| API 互換性 | OpenAI ネイティブ | OpenAI 互換 | 改修最小 |

### 1.4 スコープ外（将来検討）

- `<20秒` パス（現 Grok STT）の Groq 移行
  - Grok STT のコストは ~\$0.10/h なので、Groq \$0.04/h への置換でも省コスト
    効果あり。ただし短尺の比率が低いため省コスト額は小さく、本フェーズでは
    現状維持
  - ※将来 Grok STT が不安定になったり仕様変更があった場合に、Groq 単独で
    短尺・長尺両方を扱う構成へ統合する選択肢を残す
- Whisper Large V3 (非 Turbo, \$0.111/h) の併用
  - 精度差は WER 10.3% vs 12% の差。テスト後、精度が問題なら Turbo →
    非 Turbo への昇格を検討

---

## 2. 現状アーキテクチャ

### 2.1 transcribe フロー（main `c7d9103` 時点）

```
/v1/transcribe
 └─ estimateAudioSecondsFromBase64(audio)
    ├─ < 20 sec → callGrokSTT (xAI, model=grok-stt)
    └─ >= 20 sec → callWhisper (OpenAI, model=whisper-1)
                     └─ verbose_json + timestamp_granularities=word,segment
                     └─ language=ja
 └─ Grok の word[].text または Whisper の word[].word を正規化
 └─ groupWordsIntoSegments で文単位に再構成
```

### 2.2 関連ファイル

- `proxy/src/api.ts`
  - `callWhisper(audioBase64, filename, env)` — OpenAI への multipart POST
  - `callGrokSTT(audioBase64, filename, env)` — xAI への multipart POST
- `proxy/src/index.ts`
  - `handleTranscribe` の `useWhisper` 判定 + 呼び出し分岐
- `proxy/src/types.ts`
  - `Env` interface（`OPENAI_API_KEY` 等のシークレット型）
- `proxy/wrangler.toml`
  - secrets 一覧コメント

---

## 3. 移行後アーキテクチャ

### 3.1 transcribe フロー（移行後）

```
/v1/transcribe
 └─ estimateAudioSecondsFromBase64(audio)
    ├─ < 20 sec → callGrokSTT (xAI, model=grok-stt)        ← 変更なし
    └─ >= 20 sec → callGroqWhisper (Groq, whisper-large-v3-turbo)  ← NEW
                     └─ verbose_json + timestamp_granularities=word,segment
                     └─ language=ja
 └─ words 正規化（Groq も OpenAI 同じ word[].word 形式のはず → 要確認）
 └─ groupWordsIntoSegments
```

エンジンラベル (`engine` フィールド) は `'whisper-1'` → `'groq-whisper-large-v3-turbo'`
に変更してクライアント / ログから識別可能にする。

### 3.2 callGroqWhisper 実装

```ts
// proxy/src/api.ts 新規 export
export async function callGroqWhisper(
  audioBase64: string,
  filename: string,
  env: Env,
): Promise<Response> {
  const audioBytes = Uint8Array.from(atob(audioBase64), (c) => c.charCodeAt(0));
  const formData = new FormData();
  formData.append('file', new Blob([audioBytes]), filename);
  formData.append('model', 'whisper-large-v3-turbo');
  formData.append('response_format', 'verbose_json');
  formData.append('timestamp_granularities[]', 'word');
  formData.append('timestamp_granularities[]', 'segment');
  formData.append('language', 'ja');

  return fetch('https://api.groq.com/openai/v1/audio/transcriptions', {
    method: 'POST',
    headers: { Authorization: `Bearer ${env.GROQ_API_KEY}` },
    body: formData,
  });
}
```

`callWhisper` (OpenAI) は当面残置（フォールバック用、または将来精度比較用）。
`handleTranscribe` の useWhisper 分岐先のみ差し替え。

### 3.3 Env 拡張

```ts
// proxy/src/types.ts
export interface Env {
  // ... 既存
  GROQ_API_KEY: string;  // 新規
  // OPENAI_API_KEY は当面残す（callWhisper 経路を維持）
}
```

```toml
# proxy/wrangler.toml の secrets コメントに追記
# GROQ_API_KEY  (replaces OPENAI Whisper for >=20s recordings)
```

---

## 4. コスト試算（詳細）

### 4.1 単価比較

| プロバイダ・モデル | per hour | per minute | per second |
|---|---|---|---|
| OpenAI `whisper-1` | \$0.36 | \$0.006 | \$0.0001 |
| **Groq `whisper-large-v3-turbo`** | **\$0.04** | **\$0.000667** | **\$0.0000111** |
| Groq `whisper-large-v3` (非 Turbo) | \$0.111 | \$0.00185 | \$0.0000308 |
| xAI `grok-stt` | 約 \$0.10 (推定) | — | — |

### 4.2 想定ボリューム別月コスト

**ケース A: Premium 100人 × 5h/月 = 500h/月 のうち 9割が >=20秒録音**

- 適用範囲: 500h × 0.9 = 450h を Whisper パスで処理
- OpenAI Whisper: 450 × \$0.36 = **\$162/月**
- Groq Whisper Turbo: 450 × \$0.04 = **\$18/月**
- **削減: \$144/月（約 89%）**

**ケース B: ヘビーユーザ拡大 (Premium 500人 × 5h/月 = 2500h/月)**

- OpenAI: 2500 × 0.9 × \$0.36 = **\$810/月**
- Groq: 2500 × 0.9 × \$0.04 = **\$90/月**
- **削減: \$720/月**

### 4.3 最低課金 10秒の影響

Groq は「1リクエストあたり最低 10秒分を課金」する仕様。我々は <20秒 を
Grok ルートに流すため、Groq に来るリクエストは必ず 20秒以上 → **影響なし**。

---

## 5. リスク・検討事項

### 5.1 精度リスク

WER (Word Error Rate) は Whisper Large V3 Turbo が 12%、OpenAI Whisper 1
が 10% 程度。**日本語特化のベンチマークではない** ため、実音声で要検証:

- 検証音声: 既存 TestFlight ユーザの過去録音 5〜10件（30秒〜3分程度の
  多様な内容: 会議・授業・雑談・少人数〜複数話者）
- 比較項目: text 完全一致率 / 固有名詞認識率 / 句読点付与
- 判定基準: 重要な誤認識（固有名詞・数値）が OpenAI と同等以下なら可

精度が劣化していた場合の選択肢:
- (a) Whisper Large V3 (非 Turbo, \$0.111/h) に切替 → OpenAI とほぼ同精度
  が期待でき、それでも 3.2倍安
- (b) OpenAI Whisper にロールバック → 単純戻し

### 5.2 ファイルサイズ上限

| プロバイダ | 上限 |
|---|---|
| OpenAI Whisper | 25 MB |
| Groq Whisper (free tier) | 25 MB |
| Groq Whisper (dev tier) | 100 MB |

EchoNote の M4A は 64kbps なので **25 MB ≒ 53 分**。Premium の月5時間が
1セッションに集約されない限り問題なし。長尺セッション救済のため dev tier
への昇格（無料の場合）を確認しておく。

### 5.3 SLA・可用性

OpenAI と比べて Groq は新興プラットフォーム。障害時のフォールバック方針:

- Phase 1 (本 spec の実装範囲): Groq 失敗時に OpenAI Whisper を即フォール
  バック呼び出し (try/catch で実装)。レイテンシ犠牲だが文字起こしは取れる
- Phase 2 (将来): Groq が安定運用できると確認後、OpenAI フォールバックを
  撤去して `callWhisper` ごと削除（コード簡素化）

### 5.4 レート制限

Groq の正確な rate limit は要確認（公開情報では明示なし）。月コスト
\$18〜\$90 規模なら問題にならない想定だが、本実装前に Groq dashboard で
plan 上限を確認しておく。

### 5.5 Cloudflare Workers との互換性

- `fetch` で multipart POST → Workers 標準で対応済（既存 `callWhisper`
  と同じパターン）
- ストリーミングレスポンス不要（verbose_json は一括 JSON）
- 追加依存パッケージなし

---

## 6. 実装フェーズ

### Phase 1 — 移行本体（推定 半日〜1日）

1. `GROQ_API_KEY` を発行（[console.groq.com](https://console.groq.com/keys)）
2. proxy 側変更:
   - `proxy/src/types.ts`: `Env.GROQ_API_KEY: string` 追加
   - `proxy/src/api.ts`: `callGroqWhisper` 実装
   - `proxy/src/index.ts`: `handleTranscribe` の useWhisper 分岐先を差し替え
     - 成功時は `engine: 'groq-whisper-large-v3-turbo'` を返す
   - try/catch でフォールバック: Groq 失敗時に旧 `callWhisper` (OpenAI) を
     呼び出してログ出力（`[transcribe] Groq failed, falling back to OpenAI`）
3. `wrangler.toml` の secrets コメント更新
4. Cloudflare に `wrangler secret put GROQ_API_KEY`
5. `wrangler deploy` で反映
6. `CLAUDE.md` の "Transcription: engine routing" セクション更新

### Phase 2 — 精度検証（推定 1日）

- 過去録音 5〜10件で OpenAI vs Groq の transcript 比較
- 「明らかな誤認識」「固有名詞」「数値」を中心に評価
- 結果次第で:
  - 大半 OK → そのまま運用継続
  - 一部 NG → Whisper Large V3 (非 Turbo) に切替検討
  - 大半 NG → OpenAI Whisper にロールバック

### Phase 3 — フォールバック撤去（オプション、Phase 1 から 1〜2ヶ月後）

- Groq の運用が安定したことを確認
- `callWhisper` (OpenAI) と `OPENAI_API_KEY` 依存を削除
- ※ただし OCR で `OPENAI_API_KEY` を使っている場合は維持。要確認

---

## 7. 検証項目（実装完了時）

- [ ] `wrangler tail` で `engine=groq-whisper-large-v3-turbo` が出ること
- [ ] 1分音声で `range=0.0s..NNNs` がほぼ全長カバーすること（truncation
      なし）
- [ ] words[] と segments[] が正しく `groupWordsIntoSegments` に渡って
      文単位セグメントになること
- [ ] 既存テスト (`proxy/test/*.test.ts`) 全件 pass
- [ ] 日本語固有名詞を含む録音で OpenAI と Groq の transcript を並べて
      手動レビュー
- [ ] Groq エラー時に OpenAI フォールバックが機能すること（GROQ_API_KEY
      を一時的に invalid にして試行）
- [ ] 25MB 超のファイルが来た場合の挙動確認（事前推定で reject すべきか）

---

## 8. ロールバック方針

- Phase 1 完了直後にロールバックする必要が出た場合:
  - `proxy/src/index.ts` の useWhisper 分岐先を `callWhisper` (OpenAI) に
    戻す 1 line edit + `wrangler deploy`
  - 所要時間: 5分
- `GROQ_API_KEY` 自体は KV に残しても害なし（将来再試行用）

---

## 9. 未決定事項

| 項目 | 備考 |
|---|---|
| Phase 2 検証の合格判定の定量基準 | 「重要な誤認識ゼロ」を主観評価で運用するか、WER を数値計測するか |
| Whisper Large V3 (非 Turbo) への自動 fallback | 単発の失敗で Turbo → 非 Turbo に上げるか、Turbo のみ運用するか |
| <20秒 パスも Groq に統合するか | Grok STT を残すメリット（日本語精度・xAI 関係性）次第。本 spec のスコープ外 |
| Phase 3 撤去の判断時期 | 1ヶ月運用後、エラーログ件数で判断 |

---

## 10. 参考: コード差分の規模感

| ファイル | 変更量 |
|---|---|
| `proxy/src/api.ts` | +30 行 (`callGroqWhisper` 追加) |
| `proxy/src/types.ts` | +1 行 (`GROQ_API_KEY` 追加) |
| `proxy/src/index.ts` | ±10 行 (分岐差し替え + フォールバック追加) |
| `proxy/wrangler.toml` | +1 行 (secrets コメント) |
| `CLAUDE.md` | ±20 行 (engine routing セクション) |
| 合計 | **約 60 行** |

実装規模は非常に小さい。spec 検討と Phase 2 精度検証のほうに時間がかかる
タイプの作業。
