# AI UPSCALE — SAKASU ORIGINAL

ブラウザ内で動作する画像高画質化ツール。サーバー不要・ログイン不要・回数制限なし。

**AIモデル（`models/realesrgan_x4.onnx`）がなくても動作します。** モデルがない場合は自動的に「高品質補間モード（Bicubic + Sharpen）」で動作するため、GitHubにそのままデプロイしてすぐ使えます。AIモデルを追加すると、より高品質なAI超解像に切り替わります。

---

## 重要な仕様について

### 画像データの取り扱い
- **画像データは外部サーバーに送信されません。** すべての処理はブラウザ内で完結します。
- ただし、AIモデル・ライブラリ・フォントの**初回読み込みには外部通信が発生します**（CDN経由）。

### AI超解像について
- AIモデル（`models/realesrgan_x4.onnx`）が配置されており、かつブラウザが対応している場合のみ **AI超解像** で処理します。
- AIモデルが利用できない環境では、自動的に **高品質補間モード（Bicubic + Sharpen）** にフォールバックします。
- 処理方式は画面上に必ず表示されます。**補間処理時に「AI」とは表示しません。**

### 処理方式の表示（正確）
| 状態 | 表示 |
|------|------|
| AIモデル + WebGPU（全タイルAI成功） | `AI超解像 / Real-ESRGAN / WebGPU` |
| AIモデル + WASM（全タイルAI成功） | `AI超解像 / Real-ESRGAN / WASM` |
| AIモデルあり・一部タイル補間 | `AI超解像（一部補間フォールバックあり）` |
| 全タイル補間 / モデルなし | `高品質補間 / Bicubic + Sharpen` |

### ×2 / ×3 選択時の処理について
- AIモデルは×4固定出力のため、**×2・×3を選択した場合でも内部では×4推論を行い、最後に縮小**します。
- **処理負荷・処理時間は×4とほぼ変わりません。** WASMモードでは特に時間がかかります。
- ×2が必要な場合は処理後に画像編集ソフトでリサイズするのが高速です。

---

## AIモデルのセットアップ（AI超解像を使う場合のみ）

AI超解像を使うには、ONNXモデルファイルを手動で配置してください。モデルがなくても高品質補間モードとして動作します。

### 必要なモデル仕様（重要）

配置するモデルは以下の仕様に一致している必要があります。起動時にテスト推論（128→64→32の順で自動検出）で入出力形式・倍率を確認し、仕様外の場合は補間モードに切り替わります。

| 項目 | 要件 |
|------|------|
| 入力テンソル形状 | `[1, 3, H, W]`（float32）。固定サイズ・動的サイズどちらも対応 |
| 出力テンソル形状 | `[1, 3, H×N, W×N]`（float32、N=2〜8の整数倍） |
| 値の範囲 | 0.0〜1.0 の float32 |
| ランタイム | ONNX Runtime Web（v1.17.3）で動作するもの |
| 対応バックエンド | WebGPU または WASM |

**固定入力サイズモデルの例（`[1,3,128,128]` → `[1,3,512,512]`）**
- 起動時のテスト推論で128×128が通れば、モデル入力サイズを自動的に128×128と認識します
- 画像の端でサイズが不足する場合は、edge paddingで補完して128×128を確保します

### 推奨モデル: Real-ESRGAN x4plus

**取得先（複数の方法）:**

**方法1: HuggingFace（要確認）**
```
https://huggingface.co/sberbank-ai/Real-ESRGAN
```
※ HuggingFaceのリポジトリ構成は変わる場合があります。実際にアクセスして `*.onnx` ファイルを探してください。

**方法2: GitHub公式リポジトリからPyTorchモデルをONNX変換**
```
https://github.com/xinntao/Real-ESRGAN
```
変換コマンド例:
```bash
python convert_to_onnx.py -n RealESRGAN_x4plus --output_path models/realesrgan_x4.onnx
```

**方法3: ONNX Model Zoo / Community**
- `realesrgan` で検索し、入力 `[1,3,H,W]`・出力 `[1,3,H*4,W*4]` の float32 モデルを探す

### ライセンスについて
- Real-ESRGAN: [BSD 3-Clause License](https://github.com/xinntao/Real-ESRGAN/blob/master/LICENSE)（非商用・商用ともに使用可能、ただし条件を確認してください）
- モデルを使用する場合は必ずライセンスを確認してください

### ファイルサイズの目安
- Real-ESRGAN x4plus: 約67MB
- GitHubの1ファイル上限は100MBのため、ギリギリ収まりますが **Git LFS の使用を推奨します**

### ファイル配置
```
sakasu-upscaler/
├── index.html
├── README.md
├── models/
│   └── realesrgan_x4.onnx   ← ここに置く
└── .gitignore
```

---

## 対応フォーマット

### 入力
| フォーマット | 備考 |
|-------------|------|
| PNG | 透明背景（アルファチャンネル）保持対応。ただし半透明フチのある画像ではモデルや元画像によってフチが出る場合があります |
| JPEG / JPG | EXIF保持オプションあり |
| WebP | |
| TIFF | **一部形式のみ対応**（非圧縮・PackBits）。LZW等は非対応 |
| BMP | |
| PDF | **一度画像化してから処理**。文字・ベクター情報は保持されません |

### 出力
PNG（無劣化）/ JPEG（品質調整可）/ WebP（品質調整可）

### 非対応
- GIFアニメーション（アニメーション情報を保持できないため非対応）
- TIFFの圧縮形式（LZW、ZIP等）

---

## 処理制限（ブラウザ保護）

| 制限 | 値 |
|------|----|
| 出力最大長辺 | 12,000px（超える場合は処理停止） |
| 出力最大画素数 | 80MP（超える場合は処理停止） |
| 大サイズ警告 | 出力長辺 8,000px 以上 |
| PDF警告 | 20ページ超え |
| サイズチェック | キュー内の**全ファイル**に対して実施 |

---

## EXIFデータについて
- EXIF保持は**初期OFF**です
- EXIFには撮影日時・カメラ機種・GPS位置情報が含まれる場合があります
- 必要な場合のみ「EXIFデータを保持する」をONにしてください
- **JPEG出力時のみ有効**（PNG・WebPでは保持されません）

---

## デプロイ方法

### GitHub Pages
```bash
git remote add origin https://github.com/<ユーザー名>/sakasu-upscaler.git
git branch -m master main
git push -u origin main
```
Settings → Pages → Source: main / root → Save

大きいモデルファイルを含む場合は Git LFS を使用:
```bash
git lfs install
git lfs track "*.onnx"
git add .gitattributes
git add models/realesrgan_x4.onnx
git commit -m "add ONNX model"
```

### Netlify
`index.html` と `models/` フォルダをドラッグ＆ドロップ

---

## 技術仕様

- AI推論: ONNX Runtime Web 1.17.3（WebGPU優先 / WASM fallback）
- モデル検証: 起動時にテスト推論（128×128→64×64→32×32の順で自動検出）で入出力形式・倍率・固定サイズを確認
- タイル処理: WebGPU時128px / WASM時64px、overlap付きで境界破綻を軽減
- Alpha保持: アルファチャンネルをRGBとは別にバイキュービック拡大して合成
- フォールバック: バイキュービック補間 + アンシャープマスク + ローカルコントラスト
- 処理表示: AI成功タイル数 / 補間タイル数を正確にカウントして表示

---

## ライセンス

MIT License — SAKASU ORIGINAL

Real-ESRGANモデルを使用する場合は [Real-ESRGAN のライセンス](https://github.com/xinntao/Real-ESRGAN/blob/master/LICENSE) を確認してください。
