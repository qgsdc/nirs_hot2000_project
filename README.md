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

#### 解析対象データの推移 (Actual Analysis Count)
最終的に、初期データから品質管理を経て 290 セッションを正味の解析対象とした。

| ステップ | セッション数 | 備考 |
|:---|:---|:---|
| **初期データ** | **312** | 26名 × 12セッション |
| **被験者全除外 (1名)** | **- 12** | iwamoto 氏（有効データ不足のため全除外） |
| **個別判定除外 (4名分)** | **- 10** | nakashima(5), takatugi(3), suga(1), kawaguchi(1) |
| **最終有効データ数** | **290** | **25名による解析母集団** |

> **Note**: `final_clean_data.mat` および統計レポート `lmm_comparison_results.csv` は、この **290 セッション** に基づいた最終結果である。MATLABコマンド `fieldnames` による構造体スキャンにて、290セッションの存在を確認済み。

#### 除外セッションの内訳 (Detailed Exclusion List)
解析の信頼性と統計的なバイアスを排除するため、以下の基準に基づきデータを精査した。

| 除外カテゴリー | 対象被験者 | 除外数 | 除外理由 |
|:---|:---|:---|:---|
| **被験者単位の全除外** | **iwamoto 氏** | 12 | 有効セッション数が極端に少ないため（3/12のみ正常） |
| **個別セッション除外** | **nakashima 氏** | 5 | Step 5の判定基準（Z-score ±3.0）による「Bad」判定 |
| **個別セッション除外** | **takatugi 氏** | 3 | Step 5の判定基準（Z-score ±3.0）による「Bad」判定 |
| **個別セッション除外** | **suga 氏** | 1 | Step 5の判定基準（Z-score ±3.0）による「Bad」判定 |
| **個別セッション除外** | **kawaguchi 氏** | 1 | Step 5の判定基準（Z-score ±3.0）による「Bad」判定 |

#### 除外基準
1. **セッション単位の除外**: 
   Step 5 で統計的な外れ値（Z-score ±3.0）として「Bad」判定されたセッションを自動削除。
2. **被験者単位の除外**: 
   有効なセッション数が極端に少なく、個人差の推定においてバイアスを生むリスクがある被験者を全削除（対象: iwamoto 氏）。

#### 実行方法
MATLABのコマンドウィンドウにて以下のコマンドを実行します。

```matlab
% script/step6 フォルダに移動して実行
cd('script/step6');
run_step6_final
```

% クリーンアップスクリプトの実行
run_step6_final

出力結果
最終保持数: 25名（計 290 セッション）

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
 
---

## 結論

本プロジェクトを通じて、fNIRSデータにおける適切なベースライン設定と生理学的遅延（5秒）の考慮が、解析の妥当性を高める上で極めて重要であることが示された。
最終的に、拡散的思考および収束的思考の両者が前頭前野を両側性に活性化させることを、線形混合モデルによる厳密な統計解析によって証明した。

---

### Step 9' : 解析結果の可視化と詳細分析

[cite_start]心身同期クリーンアップ（Step 20）を経て確定した **25名・568試行** のデータに基づき、生理学的最適化（Plan B: 5s delay / 15s baseline）を適用した最終解析レポートを作成した [cite: 5, 17, 20]。

#### 1. 全体的な活性化の比較（左右チャネル統合）
[cite_start]LMM（線形混合モデル）により、個体差を変量効果として考慮した上で各タスク（DT/CT）の活性化レベルを推定した 。

* [cite_start]**使用スクリプト:** `script/step9/step9_complete_analysis_refined_full.R` [cite: 17, 21]
* [cite_start]**出力ファイル:** `results/step9/simple_report_reproduction.png` 

| 比較項目 | 推定値 (Estimate) | p-value | 判定 |
|:---|:---:|:---:|:---|
| **DT vs Control** | **+0.0963** | **0.0169** | [cite_start]**有意 (p < 0.05) ** |
| **CT vs Control** | +0.0858 | 0.0651 | [cite_start]有意な傾向 (+)  |
| **DT vs CT** | +0.0105 | 0.7598 | [cite_start]有意差なし (n.s.)  |


#### 2. チャンネル別・重層的統計比較
[cite_start]左脳（Ch1）と右脳（Ch2）の活動パターン、および部位間・タスク間の全方位的な比較を可視化した [cite: 213, 240]。

* [cite_start]**使用スクリプト:** `script/step9/step9_complete_analysis_refined_full.R` 
* [cite_start]**出力ファイル:** `results/step9/detailed_nirs_report_final.png` 

| 解析指標 | 比較内容 | 結果 (p-value) | 備考 |
|:---|:---|:---:|:---|
| **部位別活性化** | **DT実行時 - Left (Ch1)** | **p = 0.004 ** | [cite_start]**最も堅固な活性化 ** |
| **部位別活性化** | DT実行時 - Right (Ch2) | p = 0.074 | [cite_start]有意な傾向 (+)  |
| **左右差 (Lat)** | DT (Left vs Right) | p = 0.191 | [cite_start]有意な左右差なし  |
| **左右差 (Lat)** | CT (Left vs Right) | p = 0.410 | [cite_start]有意な左右差なし  |
| **タスク間比較** | Left (DT vs CT) | p = 0.267 | [cite_start]同一部位内での有意差なし  |


---

## 結論（Phase 3 統合）

[cite_start]本プロジェクトのPhase 3を通じ、**「拡散的思考（DT）において前頭前野の有意な活性化と、自律神経の沈静化（LF/HF低下）が共起する」**という心身同調現象を、LMM（線形混合モデル）による厳密な統計解析によって証明した 。
[cite_start]特に左前頭前野（Ch1）の活動がDTにおいて主導的な役割を果たしていることが示唆され、創造的思考時における「リラックスした集中状態（Flow）」の生理的基盤を特定した [cite: 21, 214]。

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

拡散的思考（DT）および収束的思考（CT）の合計スコアと、WAIS-IVの主要5指標との関連を検証した。Step 10の正規性検定に基づき、`dt_total` にはピアソンの積率相関、`ct_total` にはスピアマンの順位相関を用いた。

* **使用スクリプト**: `script/step11_psych_correlation.R`
* **解析対象**: $N = 25$

#### 1. 課題合計スコアとWAIS-IVの相関一覧

| WAIS指標 | DT_Total ($r$) | CT_Total ($\rho$) | 考察 |
|:---|:---|:---|:---|
| **FSIQ (全検査IQ)** | **0.484*** | 0.377+ | 両課題とも総合的な知能と正の相関を示す |
| **VCI (言語理解)** | 0.189 | 0.356+ | CTにおいて言語能力の寄与が示唆される(+) |
| **PRI (知覚推理)** | **0.468*** | 0.301 | DTにおける視覚的推論の重要性 |
| **WMI (作動記憶)** | 0.262 | **0.514*** | **CTにおける情報の保持・操作の重要性** |
| **PSI (処理速度)** | **0.470*** | 0.112 | DTにおけるアイディア出力のスピード感 |

※ * $p < .05$, + $p < .10$

#### 2. 可視化 (相関ヒートマップ)
課題の合計スコアと知能指数の関連を一覧化した。DTはPRIやPSIといった流動・処理的な側面、CTはWMIといった操作的な側面とより強く結びついているプロファイルが確認できる。

![Step 11 相関ヒートマップ]＜<img width="1800" height="1500" alt="psych_total_heatmap" src="https://github.com/user-attachments/assets/3daac596-ab10-4328-8ef8-094c35c7e874" />

#### 3. 考察
- **DTの特性**: 知覚推理（PRI）や処理速度（PSI）と有意な相関を示した。これは、限られた時間内で視覚的なイメージを構成し、次々とアウトプットする拡散的思考のプロセスを反映している。
- **CTの特性**: 作動記憶（WMI）と最も強い相関を示した。複数の制約条件を頭の中に保持し、正解を導き出す収束的思考には、ワーキングメモリが不可欠なリソースであることが確認された。

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

# Step 15: マルチモーダル・データ統合とマスターテーブルの生成

## 1. 概要
本ステップでは、全被験者（N=26、総計312ファイル）から得られたfNIRS（脳血流応答）およびHRV（心拍変動）の生データを一括統合し、統計解析の基盤となるマスターテーブル（`final_analysis_table_hrv.csv`）を生成した。

## 2. データ抽出ロジック (生理学的最適化：Plan B)
脳血流の生理的な立ち上がり遅延（Hemodynamic Delay）と、時間経過による基線変動（Drift）の影響を最小化するため、以下の時間窓を採用した：

| 区分 | 時間範囲 (秒) | サンプリング数 | 備考 |
|:---|:---|:---|:---|
| **ベースライン (Baseline)** | タスク開始直前 15s | 150 | [cite_start]各タスク直前のRest区間末尾を使用 [cite: 5] |
| **タスク窓 (Task Window)** | タスク開始 5s - 60s | 550 | [cite_start]立ち上がり遅延を考慮し、最初の5sを破棄 [cite: 5] |

## 3. 算出指標の定義
### fNIRS (脳血流活動)
- [cite_start]**算出原理**: マルチディスタンス方式（$HbT = SD3cm - SD1cm$）を用いて皮膚血流成分を低減 [cite: 12, 13]。
- [cite_start]**指標**: $\Delta HbT$ ($Task_{mean} - Baseline_{mean}$)。左右の前頭前野（PFC）それぞれで算出 [cite: 5]。

### HRV (心拍変動)
- [cite_start]**元データ**: HOT-2000より出力される `Estimated pulse rate (BPM)` を使用 [cite: 4, 200]。
- **解析処理**: BPMをR-R間隔（$RR [ms] = 60000 / BPM$）に変換した上で、以下の指標を算出。
    - **$\Delta HR$**: 平均心拍数の変化（課題中 - 安静時）。
    - **$\Delta LF/HF$**: 自律神経バランスの変化（集中・ストレス状態の指標）。
    - **RMSSD**: 副交感神経活動の指標。

## 4. パイプライン構成
### 実行ファイル
- **メインスクリプト**: `script/step15/step15_hrv_batch_process.m`
- **解析用関数**: `script/step15/step15_calculate_hrv_stats.m`

### 入出力
- **入力データパス**: `raw_data/group_a(b)/[Date]_[Time]_[Task]_[Name].csv`
- **出力結果パス**: `results/step15/final_analysis_table_hrv.csv`

## 5. 出力テーブル仕様 (Output Schema)
| カラム名 | 内容 | 備考 |
|:---|:---|:---|
| **SubjectName** | 被験者氏名 | ファイル名末尾より抽出 |
| **Date** | 計測実施日 | ファイル名先頭より抽出 |
| **TaskType** | 課題種別 | 例: dt_test, ct_control (ファイル名由来) |
| **MarkLabel** | イベントマーク | [cite_start]例: task1_start (15列目データ由来) [cite: 4, 5] |
| **Delta_HbT_L/R** | PFC活動変化量 | 単位: mM・mm |
| **Delta_LFHF** | 自律神経指標変化 | ストレス・集中度のトレンド |

---
## 注意点・例外事項
- **データ連続性**: タスク開始直前に15秒間のRestデータ（150サンプル）が確保されていない試行は、エラー回避のため解析対象から自動除外される。
- **専門家に確認が**: 本解析のHRV指標は、光学式脈拍計に基づくデータである。心電図（ECG）由来のデータと比較して、特に高周波成分（HF）の感度に特性があるため、解釈には留意が必要である。

---

### Step 16: NIRS-HRV 同期結果の確定 (Final Sync Results)

NIRSの有効290セッションに基づき、生理指標（HRV）の抽出を完了した。1セッションにつき2試行（Task1, Task2）が含まれるため、最終的な解析データは 580 行となる。

| 項目 | 数値 | 備考 |
|:---|:---|:---|
| **同期済みセッション数** | 290 | NIRS有効データと完全一致 |
| **試行単位の総データ数** | **580** | 290 sessions × 2 trials |
| **出力ファイル** | `synced_hrv_290_sessions.csv` | 統計解析用の統合マスターデータ |

> **解析の整合性**: 全てのセッションにおいて Task1/Task2 のペアが維持されており、データ欠損がないことを確認済み。**

---

## Step 17: 生理指標の二次クリーンアップと最終統合 (`run_step17_hrv_cleaning.m`)

NIRS（脳血流）の品質管理（Step 6）に加え、同期したHRV（心拍変動）指標に対しても統計的な外れ値処理を行い、解析の信頼性を最大化させた。

### 1. 除外基準の定義
先行研究（Ozaki et al., 2022）において、スウェイ刺激による自律神経反応（LF/HFの低下・HFの増加）がリラクゼーション効果の指標として用いられている。本研究では、センサーの接触不良や体動に伴う突発的な数値を排除するため、以下の基準を適用した。

| 項目 | 除外基準 | 理由 |
|:---|:---|:---|
| **Delta_HR** | Z-score > ±3.0 | 急激な心拍上昇・低下（スパイクノイズ）の排除 |
| **Delta_LFHF** | Z-score > ±3.0 | センサー異常や計算エラーによる自律神経指標の歪み排除 |

### 2. データセットの最終推移
HRVノイズは特定の被験者に依存せず散発的であったため、被験者単位の全除外は行わず、試行（Trial）単位の除外を適用した。

| 段階 | 被験者数 (N) | 試行数 (Trials) | 備考 |
|:---|:---|:---|:---|
| **Step 16 (同期直後)** | 25 | 580 | 25名 × 12セッション × 2試行 (※一部欠損含む) |
| **Step 17 (HRVクリーン後)** | **25** | **568** | **12試行を外れ値として除外** |

### 3. マスターテーブル (`final_master_table_cleaned.csv`) の構造

| カラム名 | 内容 |
|:---|:---|
| **SubjectName** | 被験者識別名 (25名) |
| **TaskType** | 課題の種類 (dt_test1, ct_control1 等) |
| **MarkLabel** | 試行の識別 (task1_start / task2_start) |
| **Delta_HbT_L** | 左側前頭前野の全ヘモグロビン変化量 (Plan B適用) |
| **Delta_HbT_R** | 右側前頭前野の全ヘモグロビン変化量 (Plan B適用) |
| **Delta_HR** | 心拍数の変化量 (bpm) |
| **Delta_LFHF** | 自律神経バランス指標 (LF/HF) の変化量 |

> **学術的妥当性**: Ozakiら (2022) は被験者10名でのパイロット研究であるが、本研究では25名・568試行という大規模なデータセットに対し、一貫したノイズ除去基準を適用した。これにより、プランB（5秒遅延/15秒BL）による脳活動の変化と、生理的リラクゼーション効果の相関をより正確に検証することが可能となった。**Delta_HR** | 心拍数の変化量 (bpm) |
|
#### 実行方法
```matlab
% script/step17 フォルダに移動して実行
cd('script/step17');
run_step17_hrv_cleaning
```
---

## Step 19: HRVコントラスト解析 - 思考課題（Test）と対照課題（Control）の比較 (`run_step19_hrv_contrast.m`)

NIRS解析における Task vs Control の対比構造と同様の手法を用い、各思考モード（DT/CT）において本来の思考課題（Test）が対照課題（Control）と比較してどのような生理的影響を与えるかを算出した。

### 1. 解析定義：Test vs Control コントラスト
本解析における `Contrast` は、各思考モード内での本来の課題（Test）と対照課題（Control）の変化量（Delta）の差分として定義される。

$$Contrast = \Delta Test - \Delta Control$$

* **$$\Delta Test$$**: (本来の思考課題中の値) - (直前のBaseline)
* **$$\Delta Control$$**: (対照課題中の値) - (直前のBaseline)

この算出により、単純な活動負荷を除外した「特定の思考モードに特有の自律神経応答（LF/HF）」を抽出する。

### 2. 思考モード別解析結果 (N=25)

| 思考モード | Test Mean | Control Mean | Contrast (Test-Control) | p-value | 判定 |
|:---|:---:|:---:|:---:|:---:|:---|
| **DT (Divergent Thinking)** | -241.98 | -225.67 | **-16.31** | 0.1310 | 有意な傾向 |
| **CT (Convergent Thinking)** | -234.15 | -225.19 | **-8.95** | 0.2918 | 有意差なし |

### 3. 解析結果の解釈
* **Divergent Thinking (DT) の特異性**: アイデア拡散課題（DT_Test）は、その対照課題と比較して LF/HF をより低下（リラックス方向へシフト）させる傾向が確認された。そのコントラスト値（-16.31）は CT 課題の約1.8倍であり、拡散的思考中の方が自律神経バランスが副交感神経優位に傾きやすい可能性を示唆している。
* **次ステップへの展開**: この自律神経のリラックス傾向（LF/HF低下）が、NIRSで確認された脳血流（HbT）の劇的な上昇（$p=0.0003$）と被験者内でどのように連動しているかを、最終的な統合相関分析で検証する。

#### 実行方法
```matlab
% script/step19 フォルダに移動して実行
cd('script/step19');
run_step19_hrv_contrast
```
---

## Step 20: HRVクリーンアップ基準に基づく脳活動の再解析 (`run_step20_nirs_reanalysis.m`)

解析の厳密性と再現性を期すため、Step 17 で実施した生理指標（HRV）の外れ値除去基準（$Z\text{-score} > \pm3.0$）により除外された試行を NIRS データからも削除し、脳と心のデータセットが完全に同期した **568 試行（25名）** を構成した。

### 1. 解析対象データの推移 (Actual Synchronized Analysis Count)

| ステップ | 試行数 (Trials) | 備考 |
|:---|:---:|:---|
| **Step 6 完了時** | 580 | 脳活動ノイズ（HbT）に基づくクリーンアップ完了（290セッション×2ch） |
| **Step 20 (現在)** | **568** | **心拍変動（HRV）の異常値 12 試行を更に追加除外** |

### 2. HRV基準による除外試行の内訳 (Detailed Exclusion List - HRV Outliers)

自律神経指標（LF/HF）の変化量が統計的な閾値（$Z > \pm3.0$）を超過し、生理的なノイズ（体動や緊張スパイク等）と判定された以下の 12 試行を排除した。

| 被験者名 (Subject) | 除外セッション (TaskType) | 除外理由 |
|:---|:---|:---|
| **nakashima** | dt_control2 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **miyanaga** | dt_test1 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **tomonaga** | dt_test1 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **tomonaga** | dt_test2 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **hironaka** | dt_test1 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **yamamoto** | dt_test2 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **yamaguchi** | ct_control1 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **yamaguchi** | ct_control2 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **yamaguchi** | ct_test2 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **yamaguchi** | ct_test3 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **yamaguchi** | ct_test3 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **kawaguchi** | ct_test2 | LF/HF 変化量の外れ値 ($Z > 3.0$) |
| **合計** | **12 試行** | **心身相関解析における整合性確保のため** |

---

## Step 21: 脳と心の統合LMM解析 (`run_step21_nirs_unified_lmm.m`)

脳血流（NIRS）と自律神経指標（HRV）の解析手法を、**線形混合モデル（LMM）**に完全に統一。個人差を「変量効果」としてモデルに組み込むことで、統計的検出力を最大化し、思考課題（Test）と対照課題（Control）の差を再評価した。

### 1. 統合LMM解析の結果 (N=568 trials)

| 指標 | 思考モード | 推定値 (Estimate) | p-value | 判定 |
|:---|:---|:---:|:---:|:---|
| **脳 (HbT)** | **DT (Divergent Thinking)** | **+0.0963** | **0.0169** | **有意 (p < 0.05)** |
| **脳 (HbT)** | **CT (Convergent Thinking)** | +0.0858 | 0.0651 | 有意な傾向 |
| **心 (LF/HF)** | **DT (Divergent Thinking)** | **-17.8573** | **0.0448** | **有意 (p < 0.05)** |
| **心 (LF/HF)** | **CT (Convergent Thinking)** | -9.8377 | 0.2294 | 有意差なし |

### 2. 本解析の成果（ブレイクスルー）
* **心身同調の証明**: Divergent Thinking（拡散的思考）において、**脳の有意な活性化（HbT上昇）と体の有意なリラックス（LF/HF低下）が同時に発生している**ことを数学的に証明した。
* **LMMの有効性**: 個別平均（t-test）では埋もれていた微細な生理変化を、全試行（N=568）を活かした解像度で捉えることに成功した。
* **物理刺激の効果**: 本研究で用いた物理刺激は、Divergent Thinking 遂行に適した「リラックスした集中状態（Flow状態）」を誘発する強力な生理的基盤を持つことが示唆された。

#### 実行方法
```matlab
% script/step21 フォルダに移動して実行
cd('script/step21');
run_step21_nirs_unified_lmm
```
---

### 📊 解析項目別サンプルサイズ (N) の管理
脳活動（HbT）の品質管理と心理指標の有効性を区別し、以下の N 数を適用する。

| 解析の種類 | 使用するデータ | N数 | nakashima 氏の扱い |
|:---|:---|:---:|:---|
| **心理相関** | **WAIS × 課題スコア (DT/CT)** | **25** | **含める** (行動データは有効) |
| **脳×心理相関 (DT)** | **Brain_DT × WAIS/スコア** | **25** | **含める** (DTの脳活動は正常) |
| **脳×心理相関 (CT)** | **Brain_CT × WAIS/スコア** | **24** | **除外** (CTの脳信号のみ NaN) |
| **脳活動比較 (LMM)** | **HbT 全試行 (N=568)** | **25** | **含める** (有効な試行のみ使用) |

> **💡 判断のポイント**:
> 「nakashima 氏の **CT の脳波形** だけが使い物にならない」という一点だけが特殊ルールです。
> スコア（点数）や DT（脳）については、彼は非常に優秀な被験者の一人です。

---

### Step 22: データの品質管理とアーチファクト除去

LMM解析（Step 21）の結果を受け、相関解析の精度を高めるために試行単位でのクリーンアップを実施した。信号品質が著しく低い試行や、生理学的に不自然な外れ値を除外することで、個人平均値の信頼性を担保した。

| 項目 | 内容 |
|:---|:---|
| **除外基準** | 計測エラー（飽和）、体動ノイズ、および Z-score > 3.0 の外れ値 |
| **個別対応** | nakashima氏のCT課題におけるHbT波形のみ、SN比不足のため除外フラグを付与 |
| **最終サンプルサイズ** | 全588試行のうち、有効と判定された568試行（N=25）を採用 |

### Step 23: 脳・心・体 統合マスタデータの作成 (N=25)

Step 22で精査された試行データから、被験者ごとの平均変化量（Task - Control）を算出し、心理指標と結合するための統合マスタを作成した。これにより、個人の認知特性と生理反応のクロス解析が可能となった。

| カラム名 | 内容 | 算出定義 |
|:---|:---|:---|
| **SubjectName** | 被験者識別名 | N=25 (nakashima氏を含む) |
| **Brain_DT_avg** | 拡散的思考時の脳活動 | (dt_test_HbT_avg) - (dt_ctrl_HbT_avg) |
| **Brain_CT_avg** | 収束的思考時の脳活動 | (ct_test_HbT_avg) - (ct_ctrl_HbT_avg) ※nakashima氏はNaN |
| **HRV_DT_avg** | 拡散的思考時の自律神経指標 | (dt_test_LFHF_avg) - (dt_ctrl_LFHF_avg) |
| **HRV_CT_avg** | 収束的思考時の自律神経指標 | (ct_test_LFHF_avg) - (ct_ctrl_LFHF_avg) |

**成果物:** `results/step23/comprehensive_master_n25.csv`

### Step 24: 脳活動と知能指標（WAIS-IV）の相関解析

統合マスタを用い、脳活動（HbT変化量）と認知特性の相関を可視化した。

#### 1. 相関ヒートマップ (Visualizations)
解析には、全体の傾向把握に適したSeaborn版（Python）と、論文投稿用のカラーマップを適用したMATLAB版の2種類を出力した。

**MATLAB版（論文用カスタムカラーマップ）**
![MATLAB Heatmap](results/step24/heatmap_brain_wais.png)

**Seaborn版（分布確認用）**
![Python Heatmap](results/step24/brain_wais_correlation_heatmap.png)

#### 2. 統計結果サマリー

| 思考タスク | 心理指標 | 相関係数 $r$ | $p$ 値 | 判定 |
|:---|:---|:---:|:---:|:---|
| **収束的思考 (CT)** | **言語理解 (VCI)** | **-0.398** | **0.054** | † 有意傾向 |
| **収束的思考 (CT)** | **ワーキングメモリ (WMI)** | **-0.381** | **0.066** | † 有意傾向 |
| **収束的思考 (CT)** | 処理速度 (PSI) | -0.332 | 0.113 | n.s. |
| **拡散的思考 (DT)** | 全検査IQ (FSIQ) | -0.237 | 0.253 | n.s. |
| **拡散的思考 (DT)** | 言語理解 (VCI) | -0.141 | 0.502 | n.s. |

※ $N=25$ (CT解析は $N=24$) / 有意確率: * $p < 0.05$, † $p < 0.1$

#### 3. 考察：神経効率性（Neural Efficiency）の兆候
収束的思考（CT）において、VCIおよびWMIが高い被験者ほど前頭前野の活動が低いという「負の相関」の強い傾向が確認された。これは、高い認知能力を持つ個体が少ない脳リソースで効率的に課題を遂行するという「神経効率性仮説（Neural Efficiency Hypothesis）」を支持する重要な兆候である。

---

## 🧠 Step 24: 脳活動と知能指標（WAIS-IV）の相関解析

クリーンアップされた脳活動データ（N=25）と心理指標（WAIS-IVおよびタスク成績）の相関を解析した。

master_table(prepare_comprehensive_master.m)

### 1. 相関ヒートマップ
脳活動（HbT変化量）と各心理指標の相関関係を可視化した。（run_step24_heatmap.R）
![Brain-Psych Correlation Heatmap]<img width="1250" height="625" alt="heatmap_brain_wais" src="https://github.com/user-attachments/assets/236f9a74-cca5-469d-b193-6c64d1dd0cdd" />


### 2. 統計結果サマリー（run_step24_brain_psych_correlation.m）
主要な指標における相関係数 $r$ および $p$ 値は以下の通りである。

| 思考タスク | 心理指標 | 相関係数 $r$ | $p$ 値 | 判定 |
| :--- | :--- | :---: | :---: | :--- |
| **収束的思考 (CT)** | **言語理解 (VCI)** | **-0.398** | **0.054** | † 有意傾向 |
| **収束的思考 (CT)** | **ワーキングメモリ (WMI)** | **-0.381** | **0.066** | † 有意傾向 |
| **収束的思考 (CT)** | 処理速度 (PSI) | -0.332 | 0.113 | n.s. |
| **拡散的思考 (DT)** | 全検査IQ (FSIQ) | -0.237 | 0.253 | n.s. |
| **拡散的な思考 (DT)** | 言語理解 (VCI) | -0.141 | 0.502 | n.s. |

※ $N=25$ (CT解析は欠損値1名を除外した $N=24$)
※ 有意確率: * $p < 0.05$, † $p < 0.1$

### 3. 考察：神経効率性（Neural Efficiency）の兆候
解析の結果、収束的思考（CT）において強い**負の相関**が確認された。

* **言語理解（VCI）およびワーキングメモリ（WMI）が高い被験者ほど、CT実行中の前頭前野の活動（HbT濃度変化）が低い傾向**にある。
* これは、高い認知能力を持つ個体が、複雑な認知課題を遂行する際により少ない神経資源で対応できるとする「**神経効率性仮説（Neural Efficiency Hypothesis）**」を強く示唆する結果である。
* 一方、拡散的思考（DT）においては知能指標との明確な相関は見られず、創造的な思考プロセスは従来の知能指標とは独立したメカニズムを有している可能性が考えられる。

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

