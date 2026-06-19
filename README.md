# headset-eq

**CYBER `CY-P5GMWHS` ゲーミングヘッドセットを「自分の言語で」イコライズする** —
[Almide](https://github.com/almide/almide) で書いたカスケード biquad パラメトリック EQ。

安物ゲーミングヘッドセットの音(低音ブーミー・中音引っ込み・高音ガサつき)を、
ファームウェアに触れず **ホスト側 DSP** で作り直す。その DSP の「脳」を Almide で書き、
OS の音声境界だけ薄いネイティブ/ブラウザの殻が担う、という構成。

> 背景: このヘッドセットの中身は JieLi(杰理)製 SoC で、付属 2.4GHz USB ドングルは
> 48kHz の USB Audio Class デバイスとして macOS に普通に刺さる。接続不良は Bluetooth を
> 捨ててドングル運用にすれば解消する。残るは「音質」で、それを本リポジトリの EQ で補正する。

## 何が入っているか

- `src/eq.almd` — EQ ライブラリ本体。**`Matrix` 型を一切使わず**、`Float` / `List[Float]` /
  record / stdlib `math` だけで実装した純ロジック DSP。
  - `design_peaking(fs, f0, q, gain_db)` — RBJ Audio-EQ-Cookbook ピーキングフィルタの係数設計
  - `process_sample(f, x)` — Direct Form I の 1 サンプル差分方程式(状態を進めて返す)
  - `process_buffer` / `process_chain` — 1 バンド / 多段カスケードをバッファに適用(O(n) in-place)
  - `cyber_curve()` — CY-P5GMWHS 用補正カーブ(7 バンド)
  - 数式的正しさを検証する `test` ブロック群
- `src/main.almd` — `almide run` 用デモ。チェインのインパルス応答と各バンド中心周波数での
  実測ゲインを表示する。

## 補正カーブ (`cyber_curve`)

| 中心周波数 | ゲイン | 狙い |
|---|---|---|
| 125 Hz | -2.5 dB | 低域のボワつき/こもりを除去 |
| 250 Hz | -2.0 dB | 箱鳴り(boxiness)を抑制 |
| 1 kHz | +1.5 dB | 声・実体感を前へ |
| 2 kHz | +1.5 dB | 明瞭度・定位(足音等)を強調 |
| 4 kHz | -1.0 dB | 刺さり/サ行のキツさを抑制 |
| 8 kHz | +1.0 dB | 抜け・ディテール |
| 16 kHz | +1.0 dB | 空気感 |

## 使い方

```bash
almide test                 # DSP の数式的正しさを検証(6 テスト)
almide run                  # デモ: インパルス応答 + 各バンド中心の実測ゲインを表示
almide build src/eq.almd --target wasm --release -o dist/headset_eq.wasm
```

### ブラウザで試す(WebAudio デモ)

```bash
almide build src/eq.almd --target wasm --release -o dist/headset_eq.wasm
python3 -m http.server 8000        # プロジェクトルートで
# → http://127.0.0.1:8000/web/ を開く
```

`web/index.html` は **Almide が設計した biquad 係数を wasm から読み**、ブラウザの
`IIRFilterNode` チェインに流して実行する。トーン/ピンクノイズ/音声ファイルに EQ をかけ、
`EQ: ON/OFF` で A/B 比較でき、チェイン全体の周波数応答も描画する。

> DSP の本体(RBJ 係数設計)は Almide。実際の差分方程式の実行だけ WebAudio の IIR に委譲。
> wasm 境界はスカラのみ(`num_bands() -> Int`, `band_coeff(band, k) -> Float`)。
> 注: Almide の `Int` は wasm `i64` = JS では **BigInt**。引数は `BigInt(b)` で渡す。

### テスト(`almide test`)

整定に依存しない **解析的テスト** と、十分整定させた **統合テスト** の両建て:

- `impulse response h[0] equals b0` — 差分方程式そのものの検証
- `zero-dB peaking is exact passthrough` — フィードバックが恒等になることの検証
- `peaking DC gain is exactly unity (analytic)` — 係数から `H(1)=(b0+b1+b2)/(1+a1+a2)` を厳密検算
- `peaking Nyquist gain is exactly unity (analytic)` — `H(-1)` も厳密に 1
- `full corrective chain settles to unity at DC` — 7 バンド通しても DC 不変
- `peaking +6 dB ~doubles its center-frequency amplitude` — EQ が実際に効く(10^(6/20)=1.995x)

> IIR の DC 整定テストは、過渡が許容誤差まで減衰する十分なサンプル数が要る
> (特に 125Hz 帯は極が単位円に近く整定が遅い)。正しさ検証は整定非依存の解析的テストが確実。

## デプロイ構成

```
[macOS 音声] → ホストの殻 → headset_eq.wasm (Almide製DSP)
                              └ process_chain(cyber_curve(), buffer)
```

- **ブラウザ** — `headset_eq.wasm` を WebAudio (AudioWorklet) から呼ぶ(`web/` 参照、開発中)
- **ネイティブ system EQ** — CoreAudio の殻が wasm を wasmtime でホスト
  (Almide リポジトリの `porta/src/wasm_rt.almd` が同型の実装)

### ネイティブ連携 / 既知の制約

ネイティブ C-ABI で公開するには関数に **`@export(c, "symbol")`** を付ける(`@extern(c)` の逆。
plain な public fn は自動エクスポートされない)。これで `almide build --cdylib --repr-c` が
`#[no_mangle] pub extern "C"` を出力し、CoreAudio の殻から dylib を直リンクできる。
あるいは `--target wasm`(public fn が wasm export される)を wasmtime 等でホストする手もある。

注: `build --target npm`(JS バインディング)は 0.27.10 では非機能スタブ(upstream に issue 済み)。
JS から wasm を直接叩く場合、`Int` は wasm `i64` = JS `BigInt` の境界に注意。

## ステータス

- [x] EQ コア実装(`eq.almd`、Matrix 非依存)
- [x] 数式的正しさを `almide test` で検証(6/6)
- [x] WASM ビルド成功(public fn が export される)
- [x] `almide run` デモ
- [x] WebAudio デモページ(`web/index.html` — Almide係数 → IIRFilterNode、A/B比較 + 応答描画)
- [ ] ネイティブ CoreAudio 殻(wasmtime ホスト)
- [ ] ブロック単位ストリーミング(フィルタ状態をブロック間で持続)
