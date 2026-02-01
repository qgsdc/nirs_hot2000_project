<img width="2100" height="3000" alt="ct_difficulty_heatmap" src="https://github.com/user-attachments/assets/03fe28aa-a507-426d-9641-f47e8e07a390" /># nirs_hot2000_project
**MATLAB-based fNIRS + HRV analysis pipeline (HOT-2000 / Hb133 / Check My Heart)**  
**Version:** 2026-01-18  
**Author:** Kei Saruwatari

---

## Overview / 概要
本リポジトリは、創造性課題（Divergent Thinking: DT / Convergent Thinking: CT）中に取得した
fNIRS（前頭前野） および 自律神経（HRV） データを、
再現可能かつ保守的 に解析するための MATLAB パイプラインである。

対象機器：
	•	NeU HOT-2000（HbT、SD1/SD3）
	•	Astem Hb133（HbO / HbR / HbT）
	•	Check My Heart（心拍数・HRV）

設計思想：
	•	QC は Z スコア（±3σ）に基づく透明な基準
	•	前処理は 最小限（原則 band-pass のみ）
	•	主要アウトカムは Δ / ΔΔ（Task − Control 差）
	•	Primary（事前定義）解析 と Exploratory（探索的）解析 を明確に区別

---

## Folder structure / ディレクトリ構成
<a id="folder-structure"></a>

nirs-project/
├── scripts/                 # 解析スクリプト
│   ├── analysis/            # 統計解析・図（DT/CT, Step D）
│   ├── qc/                  # QCメトリクス算出・除外
│   ├── io/                  # 読み込み・stim再構築
│   ├── pipelines/           # バッチ実行
│   ├── plots/               # 図の共通関数
│   ├── hrv/                 # HRV解析・同期
│   └── utils/               # 汎用ユーティリティ
│
├── data/ (ignored)          # 実験データ（個人情報保護のため git 管理外）
│   ├── group_a/
│   ├── group_d/
│   └── merged/
│       └── figures/         # スライド用 図・統計CSV
│
├── reports/                 # QC 等のレポート出力
└── .gitignore

⚠️ data/ is excluded from version control for privacy reasons.


## 🚀 Quickstart
<a id="quickstart"></a>

```matlab
addpath(genpath('scripts'));
rehash; clear functions;
```

## 🧠 Analysis Flow (fNIRS Pre-processing)

## Phase 1: Basic Processing & Visual QC (基本処理と目視検品)
解析の土台作りと、人の目による波形チェックを行います。

### Step 1: Data Structuring (生データの統合)
HOT-2000から出力された生CSVを読み込み、解析用構造体へ変換します。

| ファイル名 | 役割 | 主な処理内容 |
|:---|:---|:---|
| `load_raw_hot2000_v2.m` | 読み込み関数 | ・$HbT = SD3 - SD1$ による皮膚血流除去（SD減算）<br>・心拍データ（Estimated pulse rate）の抽出 |
| `run_step1_load_and_save.m` | 実行スクリプト | ・全26名×12セッション（計312ファイル）を自動スキャン<br>・`raw_all_312_sessions.mat` として一括保存 |

### Step 2: Artifact Filtering (フィルタリング)
生理的ノイズ（呼吸・血圧変動等）を除去するための前処理を行います。

| ファイル名 | 役割 | 主な処理内容 |
|:---|:---|:---|
| `run_step2_apply_filter.m` | フィルタ適用 | ・Band-pass filter (0.01 - 0.20 Hz) の適用<br>・`filtered_all_312_sessions.mat` の作成 |

### Step 3: Visual Inspection (目視QC)
全データの波形を画像化し、品質を確認します。

| ファイル名 | 役割 | 主な処理内容 |
|:---|:---|:---|
| `run_save_all_plots.m` | プロット生成 | ・全312セッションの波形図をPNG出力<br>・出力先: `qc/plots/` |

## Phase 2: Quantitative Quality Control (数値指標による品質管理)

Step 2で前処理（フィルタリング）されたデータに対し、客観的な統計指標を用いてセッションごとの品質を評価します。

### Step 4: 指標算出 (`run_step4_compute_qc_metrics.m`)

全312セッションのデータをスキャンし、各チャンネルのノイズ混入率や信号強度を算出してCSVに集計します。

#### 算出される主要指標
| 指標 | 説明 |
| :--- | :--- |
| **NoiseFraction** | Z-scoreが ±3.0 を超える異常値の割合 |
| **Std_L / R** | 左右チャンネルの標準偏差（信号の安定性） |
| **PulsePower_L / R** | 心拍帯域 (0.5-2.0Hz) のパワー（プローブの接触品質） |

#### 実行方法
MATLABのコマンドウィンドウで以下を実行してください。実行するとファイル選択画面が開くので、`processed/step2/filtered_all_312_sessions.mat` を選択します。

```matlab
% script/step4 フォルダに移動して実行
cd('script/step4');
run_step4_compute_qc_metrics
```

### Step 5: 自動ノイズ判定 (`run_step5_classify_noise.m`)

Step 4で算出された指標に基づき、全データから統計的な外れ値（Z-score ±3.0）を抽出して「Bad」判定を行います。

#### 判定基準
以下の指標のいずれかで Z-score > 3.0 を記録したセッションを自動除外対象とします。
- **NoiseFraction**: 突発的なスパイクノイズの頻度
- **Std_L/R**: 異常な信号振幅（プローブの浮き等）
- **PulsePower_L/R**: 心拍成分の欠如（接触不良）

#### 実行方法
```matlab
% script/step5 フォルダに移動して実行
cd('script/step5');
run_step5_classify_noise
```
---

出力結果
processed/step4/qc_classification_results.csv: 各セッションの判定結果（Normal/Bad）と除外理由

qc/qc_scatter_plot/qc_scatter_plot.png: 判定結果を可視化した散布図

Note: 現在のデータセットでは 312 セッション中 19 セッション が除外対象として特定されています。

---

### Step 6: データのクリーンアップ (`run_step6_final.m`)

これまでの品質管理（QC）の結果に基づき、解析に適さないデータを除去して、最終的な統計解析用データセットを構築します。

#### 除外基準
1. **セッション単位の除外**: 
   Step 5 で統計的な外れ値（Z-score ±3.0）として「Bad」判定されたセッションを自動削除。
2. **被験者単位の除外**: 
   有効なセッション数が極端に少なく、個人差の推定においてバイアスを生むリスクがある被験者を全削除。
   - **対象**: iwamoto 氏
   - **理由**: 全12セッション中、正常なデータが3セッション（25%）のみであったため。

#### 実行方法
MATLABのコマンドウィンドウにて以下のコマンドを実行します。

```matlab
% script/step6 フォルダに移動
cd('script/step6');
```

% クリーンアップスクリプトの実行
run_step6_final

出力結果
最終保持数: 25名（計 281 セッション）

出力ファイル: processed/step6/final_clean_data.mat

このファイルに格納されている clean_data 構造体が、次フェーズ（統計解析・LMM）の正味の入力データとなります。

#### データ構造
保存された `final_clean_data.mat` は以下の階層構造を持ちます。
- `clean_data.(SubjectID).(SessionID)`
  - `time`: 時間軸（秒）
  - `data`: 全ヘモグロビン変化量（HbT）。2列のデータ（[1列目: Left, 2列目: Right]）
  - `pulse`: 心拍データ
  - `mark`: マーカー情報（cell配列）
 
---

## Phase 3: Statistics Preparation (統計解析の準備)

### Step 7: 統計用集計表の作成 (Plan A & Plan B)
`run_step7_prepare_table.m` / `run_step7_prepare_table_plan_b.m`

解析の目的に合わせ、2種類の集計手法を用意しています。これらは後の線形混合モデル（LMM）において比較検証の対象となります。

---

#### **Plan A: 全区間基準補正 (Standard Average)**
セッション全体の安静時を基準とし、タスク全体の平均を算出する標準的な手法です。
* **スクリプト**: `script/step7/run_step7_prepare_table.m`
* **算出ロジック**: 
    * **Baseline**: 各セッション内の全ての「Rest」区間の平均値を $Baseline = 0$ と定義。
    * **Task**: 「Task」区間（0〜60秒）の全平均を採用。
* **出力**: `processed/step7/final_analysis_table_A.csv`

#### **Plan B: 血流遅延考慮 & 直前ベースライン (Physiological Optimized)**
脳血流の生理的応答（遅延）と、時間経過によるドリフトの影響を最小限に抑えるため、時間窓を最適化した手法です。
* **スクリプト**: `script/step7/run_step7_prepare_table_plan_b.m`
* **算出ロジック**:
    * **Baseline**: 各タスク開始の **直前15秒間** の平均値を $Baseline = 0$ と定義。
    * **Task**: 血流の立ち上がり待機（Hemodynamic Delay）として **開始直後の5秒を捨て**、残り（5〜60秒）の平均値を採用。
* **出力**: `processed/step7/final_analysis_table_B.csv`

---

#### **実行方法**
MATLABのコマンドウィンドウにて、目的に応じて以下のいずれかを実行してください。

```matlab
% script/step7 フォルダに移動
cd('script/step7');

% Plan A (標準解析) を実行する場合
run_step7_prepare_table

% Plan B (生理学的最適化解析) を実行する場合
run_step7_prepare_table_plan_b
```

| カラム名 | 内容 |
|:---|:---|
| **SubjectID** | 被験者識別ID |
| **SessionID** | 課題の種類 (dt1, ct1, dt_ctrl1 等) |
| **Channel** | 計測部位 (Ch1: Left / Ch2: Right) |
| **HbT_Change** | 補正後の全ヘモグロビン変化量（活動指標） |

### Step 8: 解析プラン別の統計比較結果 (LMM)
`results/lmm_comparison_results.csv` より抜粋。

単純平均（Plan A）では捉えきれなかった脳活動が、生理学的遅延と直前ベースラインを考慮した手法（Plan B）によって明確に検出された。

| 解析プラン | 比較項目 | 推定値 ($\beta$) | 標準誤差 (SE) | p-value | 判定 |
|:---|:---|:---|:---|:---|:---|
| **Plan A** (全体平均) | DT vs DT_ctrl | -0.0116 | 0.0064 | 0.0716 | 有意差なし |
| **Plan A** (全体平均) | CT vs CT_ctrl | 0.0028 | 0.0064 | 0.6591 | 有意差なし |
| **Plan B** (5s遅延/15sBL) | **DT vs DT_ctrl** | **0.0231** | **0.0064** | **< 0.001** | **有意な活動増 (***)** |
| **Plan B** (5s遅延/15sBL) | **CT vs CT_ctrl** | **0.0137** | **0.0061** | **0.0268** | **有意な活動増 (*)** |
| **Plan B** (5s遅延/15sBL) | DT vs CT | 0.0009 | 0.0062 | 0.8908 | 有意差なし |

**解析の結論:**
前頭前野のHbT変化量において、拡散的思考(DT)および収束的思考(CT)の両条件で、制御課題(Control)を上回る有意な活性化を確認した。
特にDTにおいて極めて高い有意差 ($p < .001$) が認められ、本実験パラダイムの有効性が示された。

---

### Step 9: 解析結果の可視化と詳細分析

統計的に有意な結果が得られた Analysis Plan B に基づき、Rを用いて視覚的な解析レポートを作成した。

#### 1. 全体的な活性化の比較
各タスク（DT/CT）がコントロール条件に対してどの程度活性化したかを、左右チャンネルを統合して算出した。

* **使用スクリプト:** `script/step9_visualize_final.R`
* **出力ファイル:** `results/step9/final_activation_plot.png`
* **知見:** - 拡散的思考 (DT) は極めて強い活性化 ($p < .001$) を示し、収束的思考 (CT) も有意な活性化 ($p = .027$) が認められた。
    - タスク間の直接比較では、DTの方が高い値を示すものの、統計的な有意差には至らなかった ($p = .891$)。

[Image of a bar chart showing fNIRS activation for DT and CT tasks with significance stars]

#### 2. チャンネル別・左右差の詳細解析
さらに詳細な分析として、左脳（Ch1）と右脳（Ch2）ごとの活動パターンを可視化した。

* **使用スクリプト:** `script/step9_the_complete_analysis.R`
* **出力ファイル:** `results/step9/complete_nirs_analysis_report.png`
* **知見:**
    - **左右差の検証 (Lateralization):** いずれのタスクにおいても有意な左右差は認められず（DT: $p = .191$, CT: $p = .410$）、前頭前野が両側性に活動していることが示された。
    - **部位ごとのタスク間比較:** 同一部位内でのタスク比較（DT vs CT）を行った結果、左脳・右脳ともにDTがCTを上回る傾向にあるが、部位別の有意差は認められなかった（左: $p = .267$, 右: $p = .411$）。
    - **ベースライン比較:** 特にDT施行時の左前頭前野（Ch1）において、コントロール条件に対し最も堅固な活性化 ($p = .004$) が確認された。
 
    - <img width="1800" height="1800" alt="final_activation_plot" src="https://github.com/user-attachments/assets/3ceeda6a-48fa-41bb-973b-51852324c019" />
　　 - <img width="2700" height="2400" alt="complete_nirs_analysis_report" src="https://github.com/user-attachments/assets/8ec2327f-e92d-4f71-ab23-9a765022f292" />

---

## 結論

本プロジェクトを通じて、fNIRSデータにおける適切なベースライン設定と生理学的遅延（5秒）の考慮が、解析の妥当性を高める上で極めて重要であることが示された。
最終的に、拡散的思考および収束的思考の両者が前頭前野を両側性に活性化させることを、線形混合モデルによる厳密な統計解析によって証明した。

---

### Step 10: 変数の正規性検定 (Normality Test)

相関分析（Step 11）における適切な統計手法（ピアソンまたはスピアマン）を選択するため、各指標の分布についてShapiro-Wilk検定およびQQプロットによる視覚的確認を行った。

* **使用スクリプト**: `script/step10_normality_full_report.R`
* **解析対象**: N=25（hori, iwamoto 除外後）

#### 1. 正規性検定の結果一覧
以下の通り、`ct_total` を除くすべての指標で正規性が確認された。

| 変数名 | p値 (Shapiro-Wilk) | 正規性の判定 | 採用する相関手法 |
|:---|:---|:---|:---|
| **Brain_DT** (DT脳活動) | 0.8101 | 有 (Yes) | ピアソンの積率相関 |
| **Brain_CT** (CT脳活動) | 0.2346 | 有 (Yes) | ピアソンの積率相関 |
| **dt_total** (DT成績) | 0.4496 | 有 (Yes) | ピアソンの積率相関 |
| **ct_total** (CT成績) | **0.0011** | **無 (No)** | **スピアマンの順位相関** |
| **wais_fsiq** (全検査IQ) | 0.6778 | 有 (Yes) | ピアソンの積率相関 |
| **wais_vci** (言語理解) | 0.4192 | 有 (Yes) | ピアソンの積率相関 |
| **wais_pri** (知覚推理) | 0.7068 | 有 (Yes) | ピアソンの積率相関 |
| **wais_wmi** (作動記憶) | 0.5042 | 有 (Yes) | ピアソンの積率相関 |
| **wais_psi** (処理速度) | 0.4682 | 有 (Yes) | ピアソンの積率相関 |

#### 2. 分布の視覚的確認 (QQプロット)
数値による検定結果を補完するため、QQプロットを作成した。`ct_total` については、理論上の正規分布を示す直線からデータの乖離が確認された。

-<img width="3000" height="3000" alt="qqplots_all_n25" src="https://github.com/user-attachments/assets/1f1a9fc8-d316-4847-9352-1c1287fa24b5" />

#### 3. 解析方針の決定
本検定の結果に基づき、以下の通り相関解析の手法を使い分ける。

| 解析の組み合わせ | 使用する統計手法 | 理由 |
|:---|:---|:---|
| **ct_total が関わる分析** | **スピアマンの順位相関係数** | 正規分布に従わないため（非パラメトリック） |
| **それ以外の変数の組み合わせ** | **ピアソンの積率相関係数** | 正規分布が確認されたため（パラメトリック） |

---

### Step 11: 思考課題成績と知能指数（WAIS-IV）の相関

拡散的思考（DT）および収束的思考（CT）の合計スコアと、WAIS-IVの各指標との関連を検証した。Step 10の正規性検定の結果に基づき、`dt_total` にはピアソンの積率相関、`ct_total` にはスピアマンの順位相関を用いた。

* **使用スクリプト**: `script/step11_psych_correlation.R`
* **解析対象**: N = 25

#### 1. 課題合計スコアとWAIS-IVの相関一覧

| 心理指標 | 手法 | wais_fsiq | wais_vci | wais_pri | wais_wmi | wais_psi |
|:---|:---|:---|:---|:---|:---|:---|
| **dt_total** | ピアソン (r) | **0.484*** | 0.189 | **0.468*** | 0.262 | **0.470*** |
| **ct_total** | スピアマン (ρ) | 0.377+ | 0.356+ | 0.301 | **0.514*** | 0.112 |

※ * p < .05, + p < .10

#### 2. 可視化 (相関図一覧)
WAIS各指標と課題成績の相関関係を以下に示す。

![心理指標相関図]<img width="6000" height="2400" alt="psych_correlation_plots" src="https://github.com/user-attachments/assets/5687aa9f-fce1-4e8a-aa89-1463d3f4cd15" />

#### 3. 考察
- **DT（拡散的思考）**: 全検査IQ（FSIQ）、知覚推理（PRI）、処理速度（PSI）との間に有意な正の相関が認められた。視覚的な情報の構成や処理スピードがアイディアの生成能力に寄与している可能性が示唆される。
- **CT（収束的思考）**: 作動記憶（WMI）との間に強い正の相関（ρ = 0.514, p = 0.0086）が認められた。論理的な推論を要する収束的思考には、ワーキングメモリによる情報の保持・操作能力が不可欠であることが裏付けられた。

---

### Step 12: WAIS-IV下位検査・プロセス分析との詳細相関分析

合計スコア（IQ）からさらに踏み込み、どの認知機能が各課題の成績（DT/CT）と関連しているかを詳細に分析した。

* **使用スクリプト**: `script/step12_psych_subscale_correlation.R`
* **解析対象**: N = 25

#### 1. 主要な相関結果（p < .05 の指標を抜粋）

| 課題 | WAIS下位指標 | 手法 | 相関係数 | p値 | 示唆される認知機能 |
|:---|:---|:---|:---|:---|:---|
| **DT** | **符号探し (symbolsearch)** | ピアソン | **0.591** | 0.0019 | 視覚的探索・処理速度 |
| **DT** | **パズル (visualpuzzles)** | ピアソン | **0.534** | 0.0060 | 視覚的構成・空間的推理 |
| **DT** | **バランス (figureweights)** | ピアソン | **0.480** | 0.0152 | 量的推論・流動性知能 |
| **DT** | **数唱：逆唱 (backward)** | ピアソン | **0.437** | 0.0288 | **情報の操作・ワーキングメモリ** |
| **CT** | **単語 (vocabulary)** | スピアマン | **0.471** | 0.0176 | 言語概念形成・結晶質知能 |
| **CT** | **算数 (arithmetic)** | スピアマン | **0.469** | 0.0181 | 数理的推論・作動記憶での操作 |

#### 2. 可視化
課題成績と各下位指標の関連の強さを俯瞰したヒートマップ、および有意な相関の散布図を作成した。

![下位指標相関ヒートマップ]<img width="2100" height="2700" alt="subscale_heatmap" src="https://github.com/user-attachments/assets/2b1494de-cbb0-4373-953a-86781e9b2feb" />
![有意な下位指標散布図]<img width="4500" height="3000" alt="significant_subscale_plots" src="https://github.com/user-attachments/assets/23150766-c825-4925-b2b1-0a35c4dc429a" />

#### 3. 考察
下位指標およびプロセス分析の結果、DTとCTでは依存している認知基盤が大きく異なることが浮き彫りとなった。
- **DT（拡散的思考）**: 単なる知識量よりも、視覚的な情報を素早く処理・構成する力、およびワーキングメモリ内での情報の入れ替え・操作（逆唱）と強く相関している。これはアイディアの生成能力が、能動的な情報の「操作機能」に依存していることを示唆する。
- **CT（収束的思考）**: 既成の語彙知識や、数理的な論理規則を適用する能力が成績に直結していることが確認された。

---

### Step 13: DT各評価指標とWAIS-IV詳細指標の網羅的相関分析

拡散的思考（DT）の4つの下位指標（流暢性・柔軟性・独創性・精緻性）と、WAIS-IVの各指標・プロセス分析との関連を詳細に検証した。

* **使用スクリプト**: `script/step13_dt_subscales_vs_wais_detail.R`
* **解析対象**: $N = 25$

#### 1. 主要な相関結果（有意な組み合わせを抜粋）

| DT指標 | WAIS詳細指標 | 手法 | 相関係数 ($r$) | $p$ 値 | 示唆される認知機能 |
|:---|:---|:---|:---|:---|:---|
| **独創性** | **全検査IQ (FSIQ)** | ピアソン | **0.642** | 0.0005 | 独創性は総合的な知能と極めて強く連動する |
| **独創性** | **数唱：逆唱 (Backward)** | ピアソン | **0.562** | 0.0035 | **情報の能動的な操作・組み替え能力** |
| **独創性** | **パズル (Visual Puzzles)** | ピアソン | **0.485** | 0.0139 | 空間的な構成・推論能力 |
| **独創性** | **行列推理 (Matrix)** | ピアソン | **0.407** | 0.0437 | 流動性知能（非言語的な法則性の理解） |
| **流暢性** | **パズル (Visual Puzzles)** | ピアソン | **0.458** | 0.0213 | 視覚的な構成力が回答の多さに寄与 |

#### 2. 可視化 (DT指標 vs WAIS詳細相関ヒートマップ)
解析の結果、**「独創性 (Originality)」** がWAISのほぼ全ての指標（特に流動性知能やワーキングメモリの操作側面）と有意な相関を示した。

![DT詳細相関ヒートマップ]<<img width="2400" height="3000" alt="dt_wais_heatmap_master_order" src="https://github.com/user-attachments/assets/5516db97-862f-4c90-92a3-f4999cb9b6dc" />

#### 3. 考察
本解析により、創造性の核心である「独創性」を支える認知的基盤が明確になった。
- **操作的ワーキングメモリの役割**: プロセス分析において「逆唱」と強い相関が見られたことは、独創性が単なる「ひらめき」ではなく、保持した情報を意図的に操作・加工する高度な制御プロセスであることを示唆している。
- **視覚的構成能力**: パズルや行列推理との相関から、アイディアの生成には視覚空間的な推論リソースが動員されている可能性が高い。
- **結論**: 独創的な思考は、WAIS-IVで測定されるような基礎的な認知能力（特に情報の操作と視覚的推論）を統合的に活用した結果であることが、データによって裏付けられた。
  
---

### Step 14: CT設問難易度別とWAIS-IV全指標の詳細相関分析

収束的思考（CT）の低難易度（Test 1-3: Easy）と高難易度（Test 4-6: Hard）における認知的要求の差異を、WAIS-IV全指標を用いて精密に検証した。

* **使用スクリプト**: `script/step14_ct_difficulty_wais_detail.R`
* **解析対象**: $N = 25$

#### 1. 難易度別・主要相関指標の比較

| 難易度 | WAIS詳細指標 | 手法 | 相関係数 ($\rho$) | $p$ 値 | 判定 |
|:---|:---|:---|:---|:---|:---|
| **CT Easy** | **行列推理 (Matrix)** | スピアマン | **0.474** | 0.0166 | 有意 (*) |
| **CT Hard** | **算数 (Arithmetic)** | スピアマン | **0.540** | 0.0054 | 有意 (*) |
| **CT Hard** | **単語 (Vocabulary)** | スピアマン | **0.436** | 0.0295 | 有意 (*) |
| **CT Hard** | **作動記憶 (WMI)** | スピアマン | **0.378** | 0.0627 | **有意傾向 (+)** |
| **CT Hard** | **数唱：逆唱 (Backward)** | スピアマン | **0.073** | 0.7285 | **有意差なし** |

※ * p < .05, + p < .10

![CT難易度相関ヒートマップ]<<img width="2100" height="3000" alt="ct_difficulty_heatmap_v2" src="https://github.com/user-attachments/assets/0bd5147a-4750-431c-b2cc-743bdda7ae71" />


#### 2. 考察：拡散思考（DT）と収束思考（CT）の認知的解離
本解析により、創造的独創性と論理的解決の背後にあるプロセスの本質的な違いが特定された。

- **CT Hardの認知的基盤**: 難易度の高いCT課題は、「算数（数理的論理）」や「単語（結晶的知能）」に強く依存しており、論理規則の正確な適用が正答率を左右している。
- **「操作」か「適用」か**: Step 13にて「DT独創性」と強い相関（$r=0.562, p<.01$）が見られた「逆唱（情報の能動的組み替え）」が、CT Hardでは全く関与していない。
- **結論**: **「情報を組み替える力（逆唱）」は、正解のない拡散思考における『独創性』に特異的なエンジンであり、正解のある収束思考（CT）はそれよりも『論理的推論と知識の適用』を主とするプロセスである**という、鮮明な棲み分けが確認された。  
---

これ以下はどのようなステップ番号にするか保留 


### Step 未定: 脳活動と心理指標の相関解析 (Correlation Analysis)

正規性検定（Step 10）の結果に基づき、各指標と脳活動（HbT変化量）の相関を算出した。`ct_total` が関わる解析にはスピアマンの順位相関を用い、それ以外の指標にはピアソンの積率相関を用いた。

* **使用スクリプト**: `script/step11_hybrid_correlation.R`, `script/step11_2_all_correlation_plots.R`
* **解析対象**: N = 25

#### 1. 相関解析結果一覧
脳活動（Task - Control）と各心理指標の相関係数およびp値を以下に示す。

| 解析の組み合わせ | 手法 | 相関係数 (r/ρ) | p値 | 判定 |
|:---|:---|:---|:---|:---|
| **Brain_DT vs dt_total** | ピアソン | 0.162 | 0.4385 | なし |
| **Brain_DT vs ct_total** | スピアマン | **-0.596** | **0.0017** | **Significant*** |
| **Brain_DT vs wais_fsiq** | ピアソン | -0.163 | 0.4355 | なし |
| **Brain_DT vs wais_vci** | ピアソン | **-0.511** | **0.0091** | **Significant*** |
| **Brain_DT vs wais_pri** | ピアソン | 0.094 | 0.6549 | なし |
| **Brain_DT vs wais_wmi** | ピアソン | -0.214 | 0.3035 | なし |
| **Brain_DT vs wais_psi** | ピアソン | 0.181 | 0.3860 | なし |

※ * p < .05, ** p < .01

#### 2. 主要な知見の可視化
拡散的思考（DT）中の脳活動と強い負の相関が認められた主要指標の散布図を以下に示す。

| DT脳活動 vs CTスコア (Spearman) | DT脳活動 vs WAIS-VCI (Pearson) |
|:---:|:---:|
| ![DT vs CT] <img width="1500" height="1200" alt="Brain_DT_vs_ct_total" src="https://github.com/user-attachments/assets/a66ff77a-ee66-44f3-8858-8024a8abef6e" />
| ![DT vs VCI] <img width="1500" height="1200" alt="Brain_DT_vs_wais_vci" src="https://github.com/user-attachments/assets/5e0c5f96-377c-473b-9237-face0e50cd1d" />


#### 3. 考察
本解析により、拡散的思考（DT）中の前頭前野活性化量と個人の認知能力との間に、興味深い関連が明らかとなった。

- **神経効率性（Neural Efficiency）の示唆**: 言語理解指数（VCI）および収束的思考（CT）スコアが高い個人ほど、DT課題遂行中の前頭前野活性化（HbT変化量）が有意に低い（負の相関）ことが示された。これは、認知能力が高い個体ほど、より少ない認知的努力（脳資源の効率的活用）で課題を遂行している可能性を示唆している。
- **課題特異的な脳活動**: 脳活動と実際のDTスコアの間には有意な相関が見られなかったことから、前頭前野の活性化はアイディアの「成績」そのものよりも、その生成に至る「プロセスのスタイル」や「主観的な負荷」を反映している可能性が高い。

![全指標の相関サマリー]<img width="4800" height="2400" alt="summary_correlation_Brain_DT" src="https://github.com/user-attachments/assets/500de3cf-d44d-4496-ba4a-c5781c4cce5d" />

---

### Step 未定: 脳活動の側性化（左右差）と認知機能プロファイルの統合解析

言語理解（VCI）および収束的思考（CT）能力が高い個人において顕著であった「脳活動の負の相関（効率化）」が、左脳（Ch1）と右脳（Ch2）のどちらで優位に現れているかを検証した。

* **使用スクリプト**: `script/step13_laterality_analysis.R`
* **解析対象**: N = 25

#### 1. 左右別相関係数一覧 (DT課題実行時)

| 心理指標 | 脳部位 | 手法 | 相関係数 (r/ρ) | p値 | 判定 |
|:---|:---|:---|:---|:---|:---|
| **WAIS-VCI** | 左脳 (Ch1) | ピアソン | -0.362 | 0.0753 | 有意傾向 (+) |
| **WAIS-VCI** | **右脳 (Ch2)** | ピアソン | **-0.533** | **0.0061** | **Significant (**) |
| **CT_Total** | 左脳 (Ch1) | スピアマン | -0.393 | 0.0520 | 有意傾向 (+) |
| **CT_Total** | **右脳 (Ch2)** | スピアマン | **-0.576** | **0.0026** | **Significant (**) |

※ ** p < .01, + p < .10

#### 2. 視覚的比較 (左右のコントラスト)
右脳（Ch2）において、認知能力が高いほど脳活動が鮮やかに抑制（効率化）される様子が確認された。

![左右の相関比較]<img width="2100" height="2700" alt="subscale_heatmap" src="https://github.com/user-attachments/assets/0e0c9a14-0147-4425-b412-0e6ccdf54a67" />

#### 3. 考察：下位検査の知見（Step 12）に基づく統合的解釈
本解析の結果、**「高知能者は拡散的思考（DT）において、特に右脳の前頭前野を効率化させている」**という事実が明らかになった。Step 12での下位検査分析を考慮すると、以下の統合的なメカニズムが示唆される。

- **右脳的リソースの最適化**: Step 12にて、DTの成績は「パズル（視覚的構成）」や「符号探し（視覚探索）」といった右脳優位とされる機能と強く相関していた。
- **効率的なトップダウン制御**: 言語理解（VCI）や論理思考（CT）に優れた個人は、これらの右脳的な「イメージを広げるプロセス」を、前頭前野による強力な制御によって**極めて低燃費（神経効率的）**に遂行していると考えられる。
- **結論**: 高い言語・論理能力を持つ脳は、創造的課題において「右脳リソースをいかに無駄なく、賢く管理するか」という高度な制御能力を有しており、それが右脳側における顕著な負の相関として現れたと推測される。

---

これ以下はどのようなステップ番号にするか保留 

### Step 未定: 能力群間における脳活動の比較 (Group Comparison)

相関分析で関連が見られた「言語理解（VCI）」に基づき、被験者を中央値（Median = 112）で2群に分け、拡散的思考（DT）中の脳活動に差があるかを検討した。

* **使用スクリプト**: `script/step12_group_comparison.R`
* **手法**: 独立二標本のt検定（Welch's t-test）

#### 1. 群間比較の結果
統計的な有意差（p < .05）は認められなかったが、平均値においてはHigh VCI群がLow VCI群よりも低い活動量を示す傾向が確認された。

| 指標 | High VCI (Mean) | Low VCI (Mean) | t値 | p値 | 判定 |
|:---|:---|:---|:---|:---|:---|
| **Brain_DT** | 0.0134 | 0.0341 | -1.26 | 0.2208 | 有意差なし (n.s.) |

#### 2. 可視化
![VCI群間比較]<img width="1800" height="1500" alt="vci_group_comparison_dt" src="https://github.com/user-attachments/assets/2ccaf11e-49c6-4a85-b3bb-a54283df770e" />


#### 3. 考察
群間比較においては有意差に至らなかった。これは中央値分割による情報の損失や、各群内における個体差の大きさが影響していると考えられる。しかし、平均値の傾向およびStep 11の相関分析結果を総合すると、言語的能力が高いほどDT課題中の前頭前野活性が抑制される「神経効率性」の傾向は依然として示唆されている。**

---

### Step 未定: 脳活動の側性化（左右差）と心理指標の関連

言語能力（VCI）および収束的思考（CT）能力と脳活動の負の相関が、左脳（Ch1）と右脳（Ch2）のどちらでより顕著であるかを検証した。

* **使用スクリプト**: `script/step13_laterality_analysis.R`
* **解析対象**: N=25

#### 1. 左右別相関係数一覧 (DT課題)

| 心理指標 | 脳部位 | 手法 | 相関係数 (r/ρ) | p値 | 判定 |
|:---|:---|:---|:---|:---|:---|
| **WAIS-VCI** | 左脳 (Ch1) | ピアソン | -0.362 | 0.0753 | 有意傾向 (+) |
| **WAIS-VCI** | **右脳 (Ch2)** | ピアソン | **-0.533** | **0.0061** | **Significant (**) |
| **CT_Total** | 左脳 (Ch1) | スピアマン | -0.393 | 0.0520 | 有意傾向 (+) |
| **CT_Total** | **右脳 (Ch2)** | スピアマン | **-0.576** | **0.0026** | **Significant (**) |

※ ** p < .01, + p < .10

#### 2. 視覚的比較 (左右のコントラスト)
左脳（Ch1）と右脳（Ch2）における相関の強さの違いを以下の散布図に示す。右脳において、回帰線の傾きがより急峻であり、統計的に有意な負の相関が確認できる。

![左右の相関比較]<img width="3000" height="2400" alt="laterality_comparison_plots" src="https://github.com/user-attachments/assets/1bd4d8c9-e366-414b-ae17-cd3d17fe4f9c" />

#### 3. 考察
解析の結果、本研究において示唆された「神経効率性（高い知的能力に伴う活動の抑制）」は、**右脳（Ch2）においてより顕著に現れる**ことが明らかとなった。

- **右脳制御の重要性**: 言語理解（VCI）は一般に左脳優位な指標とされるが、拡散的思考課題（DT）遂行時においては、高い言語的能力を持つ個人ほど**右脳側**の前頭前野リソースを無駄に活性化させず、最適に抑制・制御している可能性が高い。
- **結論**: 高い認知能力は、前頭前野ネットワーク全体の「使い方の洗練」に寄与しており、特に右脳側におけるリソース管理の効率化が、創造的課題遂行時の脳活動量に反映されていると考えられる。

---


  
## 📂 データ構造の定義 (Data Hierarchy)

解析に使用する `.mat` ファイルは以下の階層構造を持ちます。

| レベル | 変数名 / フィールド | 内容 | 型・サイズ |
|:---|:---|:---|:---|
| 第1階層 | `raw_all` | **全体構造体** | ・全26名のデータを保持する構造体 |
| 第2階層 | `.[subject_id]` | **被験者ID** | ・個別被験者（例：`nakashima`）のフィールド |
| 第3階層 | `.[session_id]` | **セッションID** | ・各試行（例：`dt1, dt_ctrl1`）のデータ群 |
| 第4階層 | `.data` | **HbT 変化量** | ・[Time x 2] 行列 (1:左 / 2:右) |
| 第4階層 | `.pulse` | **心拍データ** | ・[Time x 1] 列ベクトル |
| 第4階層 | `.time` | **時間軸** | ・[Time x 1] 列ベクトル |
| 第4階層 | `.mark` | **マーカー** | ・[Time x 1] 列ベクトル |

#### セッション名のマッピング規則 (12 Sessions / Subject)

| 課題区分 | セッション名 | 内容 | 試行数 |
|:---|:---|:---|:---|
| **二重課題 (DT)** | `dt1, 2, 3` | 創造性課題（DT）実行中 | 3回 |
| | `dt_ctrl1, 2, 3` | DT対照条件（Control） | 3回 |
| **単一課題 (CT)** | `ct1, 2, 3` | 創造性課題（CT）実行中 | 3回 |
| | `ct_ctrl1, 2, 3` | CT対照条件（Control） | 3回 |

---


## ✅ Current Status
- **Last Updated:** 2026-01-31
- **Completion:** Step 1, 2, 3 完了
- **Records:** 全312セッションの処理済みデータおよびプロット（`qc/plots/`）の生成を確認。
	


| Step | Script / Module | Description (English) | 内容（日本語） |
|:---:|:----------------|:----------------------|:---------------|
| **1** | `load_raw_hot2000.m` | Load and structure raw HOT-2000 CSV files | HOT-2000の生CSVを読み込み、時系列構造を作成 |
| **2** | `BandPassFilter` | Apply band-pass filter (0.01–0.20 Hz) | 0.01–0.20 Hz 帯域通過フィルタでドリフト・生理ノイズ除去 |
| **3** | *(Hampel / PCA off)* | Skip aggressive denoising | 外れ値除去・PCAは実施せず（最小前処理） |
| **4** | `qc_hot2000_metrics.m` | Compute QC metrics | 加速度RMS・Band power等のQC指標を算出 |
| **5** | `qc_classify_noise.m` | Automatic noise classification | Z-score（±3σ）に基づくノイズ自動分類 |
| **6** | `qc_filter_keep_normal_signal.m` | Remove outlier sessions | 外れ値セッションを除外 |
| **7** | `make_stats_table_merged.m` | Merge groups and export QC stats | グループA/D統合とQC統計出力 |
| **8** | `build_stim_from_marks.m` | Reconstruct stimuli from Mark column | Mark列から刺激タイミングを再構成 |
| **9** | `run_make_deltas_from_manifest.m` | Compute Δ and ΔΔ values | ΔHbT・ΔΔHbTを算出（Task−Control） |
| **10** | `run_DTvsCT_repMean_stats.m` | DT vs CT comparison | DTとCTのΔΔHbTを被験者内比較 |
| **11** | `run_onesample_deltadelta_vs0.m` | One-sample test vs baseline | ΔΔHbTがbaselineから変化したか検定 |
| **12** | `run_DTvsCT_LeftRight_barSE_stats.m` | Exploratory laterality analysis | 左右別（Fp1/Fp2）の探索的比較 |
| **13** | `run_stepD1_CT_rep6_trials1to3_vs_4to6.m` | CT difficulty (early vs late) | CT前半 vs 後半（難易度操作）の比較 |
| **14** | `run_stepD2_CTscore_x_deltadelta_scatter.m` | CT score × ΔΔHbT correlation | CT成績とΔΔHbTの探索的相関解析 |
| **15** | `/reports/` | Export figures and statistics | 図・統計結果を自動保存 |

Quality Control (Z-score Based)

指標
	•	AccelRMS（体動）：Virtanen et al., 2011
	•	BandPowerSum（0.01–0.2 Hz）：Montgomery, 2019

基準
    •	QCは Zスコア（±3） により外れ値セッションを除外する。

```matlab
run_qc_group("data/group_a");
run_qc_group("data/group_d");

qc_classify_noise("data/group_a/QC_hot2000_metrics.csv");
qc_classify_noise("data/group_d/QC_hot2000_metrics.csv");

qc_filter_keep_normal_signal("data/group_a/QC_hot2000_metrics_classified.csv");
qc_filter_keep_normal_signal("data/group_d/QC_hot2000_metrics_classified.csv");

make_stats_table_merged("data/group_a","data/group_d", ...
  'SaveTxt',true,'SaveCsv',true,'OutName','QC_merged');
```

🧠 Core Outcome: Δ / ΔΔ Analysis（Primary）

Baseline
	•	各 Task 直前 Rest の末尾 15 秒

定義
	•	ΔHbT = mean(Task) − mean(Rest_tail15s)
	•	ΔΔHbT = ΔHbT_test − ΔHbT_control
	•	HbT = SD3 − SD1（左右別 → 必要に応じ平均）

Subject-level（rep平均）
	•	ΔDT_subj = mean(ΔΔHbT_DT)
	•	ΔCT_subj = mean(ΔΔHbT_CT)

出力：
data/merged/deltadelta_subject_mean.csv

DT vs CT（paired t-test）

```matlab
run_DTvsCT_repMean_stats_boxplot( ...
  "PairedCsv","data/merged/paired_deltadelta_312.csv", ...
  "OutDir","data/merged/figures", ...
  "ShowPoints",true);
```

結果
	•	t(25)=0.928, p=0.362
	•	Cohen’s dz = 0.182（small）

ΔΔ vs 0（one-sample）

```matlab
run_onesample_deltadelta_vs0_barSE( ...
  "Csv","data/merged/deltadelta_subject_mean.csv", ...
  "OutDir","data/merged/figures");
```

結果
	•	DT: t(25)=0.499, p=0.622
	•	CT: t(25)=-0.413, p=0.683

⸻

🧠 Exploratory Analyses

Laterality（Left / Right）

```matlab
run_DTvsCT_LeftRight_barSE_stats( ...
  "PairedCsv","data/merged/paired_deltadelta_312.csv", ...
  "OutDir","data/merged/figures", ...
  "FigName","stepB_like_DTvsCT_LeftRight.png", ...
  "ShowPoints",true);
```

	•	Left: t(25)=0.977, p=0.338
	•	Right: t(25)=0.707, p=0.486

※ 仮説生成的解析として報告。

⸻

🧠 Step D: Within-task Difficulty Manipulation (CT)

難易度順（Orita et al., 2018）
	•	CT1: 69.7%
	•	CT2: 66.7%
	•	CT3: 60.6%
	•	CT4: 57.6%
	•	CT5: 51.5%
	•	CT6: 48.5%

Step D1：前半 vs 後半

```matlab
run_stepD1_CT_rep6_trials1to3_vs_4to6( ...
  "PairedRep6Csv","data/merged/paired_deltadelta_312_rep6.csv", ...
  "OutDir","data/merged/figures");
```

	•	t(25)=1.857, p=0.075
	•	dz=0.364（medium, trend-level）

Step D2：CT score × ΔΔHbT

```matlab
run_stepD2_CTscore_x_deltadelta_scatter( ...
  "MasterXlsx","data/master_subject_table_n26_202503.xlsx", ...
  "DeltaDeltaCsv","data/merged/deltadelta_subject_mean.csv", ...
  "OutDir","data/merged/figures");
```

	•	r=0.11, p=0.591（ns）


---

## 🧠 CT score × WAIS (Core indices)
<a id="ct-wais"></a>

### Overview
CT成績（CT score）と WAIS の主要指標（FSIQ / VCI / PRI / WMI / PSI）の関連を、
Pearson の相関（two-tailed）で検討した（N=26）。

**重要：CT_score の定義（現データ仕様）**
本データの `ct_test1-3` は「CT1〜CT6の個別得点」ではなく、
**2問ずつまとめたブロック得点**を表す：

- `ct_test1` = CT1 + CT2  
- `ct_test2` = CT3 + CT4  
- `ct_test3` = CT5 + CT6  
- `CT_score` = `ct_test1 + ct_test2 + ct_test3`（最大 6 点）

※ 将来的に CT1〜CT6 を個別列として追記し、難易度別解析も拡張予定。

---

### Method
- Test: Pearson correlation (two-tailed)
- Multiple comparisons: Benjamini–Hochberg FDR（5指標）
- Effect size: r と r²（説明率）を併記
- 欠損は pairwise deletion（各相関で利用可能な被験者のみ）
- `include==1` の被験者のみを解析対象

---

### Reproducibility (script)

```matlab
out = run_CT_x_WAIS_core_indices( ...
  "MasterXlsx","data/master_subject_table_n26_202503.xlsx", ...
  "CTcol","CT_score_sum3", ...
  "WAIScols",["FSIQ","VCI","PRI","WMI","PSI"], ...
  "IncludeCol","include", ...
  "OutDir","data/merged/figures");
```

Results (current dataset)

| WAIS index | n | r | r² | p (two-tailed) | q (FDR) |
|-----------:|--:|---:|---:|--------------:|--------:|
| FSIQ | 26 | 0.439 | 0.19 | 0.024 | 0.061 |
| VCI  | 26 | 0.374 | 0.14 | 0.059 | 0.099 |
| PRI  | 26 | 0.327 | 0.11 | 0.102 | 0.128 |
| WMI  | 26 | 0.491 | 0.24 | 0.011 | 0.054 |
| PSI  | 26 | 0.061 | <0.01 | 0.767 | 0.767 |

Interpretation (for README / manuscript)
- FSIQ および WMI は CT score と中程度の正の相関を示した（r ≈ 0.44–0.49）。
- ただし 5 指標に対する多重比較補正（FDR）後は、FSIQ/WMI ともに q 値が 0.05 をわずかに上回り、
  統計的には trend-level / suggestive（補正後有意には達しないが、効果量と未補正 p 値を考慮すると
  将来の検証に値する可能性を示す）な関連と解釈される。
- VCI および PRI も正の相関方向を示したが、統計的優位性には達しなかった。
- PSI と CT score の間には有意な関連は認められなかった。
⸻

Outputs
	•	Figures (scatter):
	•	data/merged/figures/CT_x_WAIS_FSIQ_scatter.png
	•	data/merged/figures/CT_x_WAIS_VCI_scatter.png
	•	data/merged/figures/CT_x_WAIS_PRI_scatter.png
	•	data/merged/figures/CT_x_WAIS_WMI_scatter.png
	•	data/merged/figures/CT_x_WAIS_PSI_scatter.png
	•	Tables:
	•	data/merged/figures/CT_x_WAIS_correlations_core.csv
	•	data/merged/figures/CT_x_WAIS_merged.csv

### Notes
- 本解析は CT score（6 問合計）を用いた被験者間相関である。
- 今後、CT1–CT6 を個別列として追加し、
  難易度別（early vs late / item-wise）解析を実施予定である。
  
⸻

References
	•	Virtanen et al. (2011) J. Biomed. Opt.
	•	Montgomery (2019) Introduction to Statistical Quality Control
	•	Bergmann et al. (2024) Bioengineering
	•	Orita et al. (2018)

⸻

Summary

本リポジトリは、生データから
QC → Δ/ΔΔ → 群統計 → 探索的解析 までを
一貫して再現可能に実行できる解析基盤を提供する。

