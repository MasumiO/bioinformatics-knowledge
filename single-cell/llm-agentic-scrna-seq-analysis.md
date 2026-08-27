---
title: "LLM・AIエージェントによるscRNA-seq解析の手法と実務アーキテクチャ"
date: 2026-08-27
status: active
topic: "single-cell RNA-seq / LLM agents"
tags:
  - single-cell
  - scRNA-seq
  - LLM
  - AI-agent
  - MCP
  - Scanpy
  - Seurat
  - reproducibility
reviewed_from: []
sources:
  - "https://doi.org/10.1038/s41592-026-03029-6"
  - "https://openreview.net/forum?id=BsA2GNkJhz"
  - "https://doi.org/10.18653/v1/2026.acl-demo.40"
  - "https://doi.org/10.1002/aic.70285"
  - "https://arxiv.org/abs/2602.09063"
  - "https://doi.org/10.1038/s41587-025-02857-9"
  - "https://doi.org/10.1038/s41467-025-67084-x"
  - "https://doi.org/10.1038/s41592-024-02235-4"
  - "https://doi.org/10.1038/s41592-024-02201-0"
  - "https://doi.org/10.1038/s41592-024-02305-7"
  - "https://proceedings.mlr.press/v235/levine24a.html"
---

# Summary

2026年8月時点で、LLMにscRNA-seq解析を「実施させる」方法は、少なくとも次の6系統に分けるべきである。

1. **end-to-endのscRNA-seq専用AIエージェント**：CellAgent、CellVoyager、scChat、Gardener、CellPilotなど。LLMが解析計画を立て、Scanpy等のツールやPythonコードを実行し、中間結果を評価しながら次の解析を選択する。
2. **汎用コーディングエージェント＋実行環境**：Codex/Claude Code等の一般的なエージェントに、AnnData/Seurat object、Scanpy/Seurat/scvi-tools等を操作させる。専用エージェントより柔軟だが、scBenchでは依然として完全自律運用に十分な信頼性には達していない。
3. **MCP/function callingによる解析ツール統合**：SCMCPのようにScanpy、細胞間相互作用、pseudotime、enrichment等を構造化ツールとしてLLMに公開する。実務上は、自由なコード生成より監査・安全・再現性を高めやすい。
4. **annotation専用multi-agent**：GPTCelltype、CASSIA、CyteTypeなど。解析全体ではなく細胞型注釈に集中しているが、LLMの生物学知識、RAG、複数エージェントによる検証の効果を評価する重要な実例である。
5. **single-cell foundation model / biological language model**：scGPT、scFoundation、Geneformer、Cell2Sentence/C2S-Scale、scMulanなど。これらは通常、解析ワークフローを自律実行する「エージェント」ではなく、表現学習、cell type prediction、batch integration、perturbation prediction等のモデル部品である。
6. **transcriptome–language multimodal model**：CellWhisperer等。発現プロファイルと自然言語を直接結び付け、自然言語でデータを探索するが、QC→正規化→クラスタリング→統計検定を逐次実行する従来型解析エージェントとは性質が異なる。

現時点での実務上の結論は、**LLMに解析を全面的に任せるのではなく、LLMを「制御面（planner / interpreter）」として使用し、データ処理はローカルの検証済みツール、コンテナ、固定バージョン、明示的な状態管理の下で実行する構成が最も現実的**である。特にQC、サンプル単位の統計設計、batch correction、cluster resolution、annotation、DE、trajectory/CCCの生物学的仮定はhuman-in-the-loopの承認点にすべきである。

## When this matters

このノートは次の判断に使う。

- 「自然言語だけでscRNA-seq解析をどこまで自動化できるか」を評価するとき
- バイオインフォマティクス業務にLLMエージェントを導入するとき
- CellAgent、CellVoyager、scChat、Gardener、CellWhisperer等の違いを整理するとき
- MCP/function callingと自由なコード生成のどちらを採用するかを決めるとき
- LLMによる解析の再現性、統計的妥当性、データプライバシーを設計するとき

# 1. 「LLMがscRNA-seq解析する」の定義

LLM利用を一括りにすると誤解しやすい。実際には次の層がある。

| レベル | LLMの役割 | 例 | 生データからの自律解析 |
|---|---|---|---|
| L0 | 結果の説明だけ | UMAP/DE表の解釈 | なし |
| L1 | marker geneからcell typeを回答 | GPTCelltype | なし |
| L2 | 専用モデルで表現学習・予測 | scGPT, scFoundation, Geneformer | 通常なし |
| L3 | 自然言語から既存ツールを呼ぶ | SCMCP, scChat | 部分的 |
| L4 | 計画→実行→評価→修正 | CellAgent, Gardener, CellPilot | あり |
| L5 | 仮説生成から反復解析まで自律探索 | CellVoyager | あり。ただし研究探索寄り |

実務上重要なのは、L2の「foundation model」とL4/L5の「agent」を混同しないことである。scGPTのようなモデルは強力なsingle-cell表現を与えるが、研究者の代わりに解析計画を決めてScanpyを実行し、結果を見て次の検定に進むわけではない。一方、CellAgentやCellVoyagerはLLMを意思決定ループに置き、外部コードやツールを実行する。

# 2. end-to-end / agentic scRNA-seq解析

## 2.1 CellAgent

**位置付け:** 最も明示的に「自然言語からend-to-end single-cell解析」を目標にしたmulti-agent系の代表例。

CellAgentはPlanner、Executor、Evaluatorからなる階層的multi-agent構成を採用し、解析タスクを分解し、ツールとハイパーパラメータを選択し、実行結果を評価して必要なら再実行する。最新版はscRNA-seqに加えてspatial transcriptomicsも対象とし、expert-curated toolkitである**sc-Omni**をツール層として持つ。bioRxivで2024年に初版、2025年に大幅更新され、ICLR 2026 Posterとして掲載された。

対応範囲として報告されているものには、cell type annotation、batch correction、trajectory inference、spatial domain identification、spatial imputation等が含まれる。論文では人間専門家と比較した効率・品質評価も行っている。

**強み**

- planner/executor/evaluatorを分離し、単発プロンプトより明示的な自己修正ループを持つ。
- ツール選択とパラメータ探索自体をagentic decisionにしている。
- end-to-end natural-language analysisを明確な設計目標にしている。

**限界**

- 評価指標の多くは著者設計のtask-specific benchmarkであり、独立した現場データでの失敗率を直接示すものではない。
- evaluator自体も自動評価であるため、誤った解析仮定をもっともらしく承認する可能性が残る。
- 統計設計やsample-level inferenceの正しさは「ツール実行成功率」とは別問題である。

**成熟度:** 実験利用〜研究デモ。エンドツーエンド自律性は高いが、無監督の本番解析を正当化する独立検証はまだ不足。

Sources:
- OpenReview / ICLR 2026: https://openreview.net/forum?id=BsA2GNkJhz
- bioRxiv: https://doi.org/10.1101/2024.05.13.593861

## 2.2 CellVoyager

**位置付け:** 「定型pipelineの自動化」よりも、LLMが研究背景を読み、データを反復的に解析して新しい仮説を生成する**AI computational biologist**に近い。

Nature Methods 2026のCellVoyagerは、LLMがJupyter notebook内で解析を生成・実行し、結果を観察しながら次の解析を決める。入力は主にAnnData `.h5ad` と研究背景（論文要約等）。GUIでは実行モデル、hypothesis-generation model、Deep Researchの利用、iteration数を設定でき、ユーザーは途中でフィードバック、コード編集、解析継続、終了を行える。

CellBenchでは、公開scRNA-seq研究の背景情報のみから、著者が実際に行った解析を予測する課題でGPT-4oやo3-miniを上回った。Nature Methods版は76 studiesを用い、最大23%の改善を報告している。またCOVID-19、endometrium、brain agingのcase studyで未報告の仮説を生成し、専門家評価を受けている。

**強み**

- notebookを実行可能な研究記録として残す。
- 「次に何を解析すべきか」という探索に強い。
- human feedbackとagent autonomyの中間を取りやすい。

**限界**

- CellBenchは「正しいQC閾値」「統計検定が妥当か」ではなく、主として研究者が実際に選んだ分析の予測能力を測るため、routine pipelineの正確性を直接保証しない。
- 自律的なhypothesis searchはmultiple testing / researcher degrees of freedomを大きく増やす。探索結果をconfirmatory evidenceとして扱うべきではない。
- 生成された「新規発見」は独立データや実験検証を必要とする。

**成熟度:** 研究デモ〜高度な探索支援。仮説生成のco-scientist用途は有望だが、臨床・規制・確証解析にはhuman reviewが必須。

Sources:
- Nature Methods 2026: https://doi.org/10.1038/s41592-026-03029-6
- GitHub: https://github.com/zou-group/CellVoyager

## 2.3 scChat

**位置付け:** multi-agent + RAGを用いたcontext-awareなscRNA-seq co-pilot。

scChatはAnnDataを対象とし、Planner、Executor、Evaluator等の役割を組み合わせて、preprocessing、cell type annotation、enrichment、visualizationを自然言語から実行する。特徴は、単なる解析だけでなく、論文・研究文脈をRAGで取り込み、hypothesis validation、mechanistic interpretation、次の実験提案まで扱う点にある。

**強み**

- quantitative analysisと研究文脈の統合を明示的に設計している。
- conversational workflowを通じて結果の説明とfollow-upを行いやすい。

**限界**

- 外部文献RAGの品質と解析結果の品質が混ざるため、citation provenanceを別管理すべき。
- contextual explanationがもっともらしくても、解析手順の妥当性を保証しない。

**成熟度:** 実験利用候補。

Sources:
- AIChE Journal 2026: https://doi.org/10.1002/aic.70285
- GitHub: https://github.com/li-group/scChat

## 2.4 Gardener

**位置付け:** 2026年時点で、実務アーキテクチャとして特に参考になる設計。LLMの賢さより**state、provenance、data residency**を中心に据えている。

ACL 2026 DemoのGardenerは、cloud LLMによるreasoningと、local/on-deviceのscientific engineを分離する。Experiment Management Kernel (EMK) が解析状態をimmutable snapshotとして外部化し、各状態をlineageで接続する。これによりrollback、branching、alternative analysisの比較、過去計算の再利用が可能になる。raw expression matrixやlocal artifactは端末に残し、cloud LLMにはsnapshot IDとsanitized summaryだけを渡す設計である。

**強み**

- conversational historyを唯一のstate storeにしない。
- rollback/branchingがfirst-class conceptで、解析経緯を監査しやすい。
- データプライバシーとクラウドLLM利用を両立しやすい。
- human-in-the-loop GUIを前提にしている。

**限界**

- ACL system demonstrationであり、生物学的benchmarkの厚みはCellAgent/CellVoyagerほどではない。
- tool implementationの品質はagent architectureとは別に検証が必要。

**成熟度:** 実験利用候補。特にコンサルティングや機密データを扱う運用設計の参考として成熟度が高い。

Source:
- ACL 2026 Demo: https://doi.org/10.18653/v1/2026.acl-demo.40

## 2.5 CellPilot

**位置付け:** 大規模クラウドLLMではなく、ローカル実行可能なsmall language modelをstructured workflowで「操縦」し、raw count matrixからannotationまで進める。

2026年7月のbioRxiv preprintでは、8Bモデルでも、prompt-onlyよりagentic orchestrationを加えることでGTExで0.39→0.89までannotation accuracyが向上し、同じframework内のGPT-4o 0.92に近づいたと報告している。raw countからstandard single-cell toolsを呼び、intermediate observationを見て次の判断を修正する。全execution traceを保持する。

**重要な示唆:** performanceはモデルサイズだけでなく、**workflow orchestration、tool grounding、観察→修正ループ**に大きく依存する。

**成熟度:** 研究デモ。主対象はannotationまでのworkflowで、DE、trajectory、CCCまでを包括する汎用agentではない。

Source:
- bioRxiv 2026: https://doi.org/10.64898/2026.07.06.736807

## 2.6 scAgent等の実装系

GitHub上には10x Chromium向けのscAgentなど、QC、normalization、HVG、clustering、annotation、DE、trajectoryを自然言語で操作する実装も出ている。scAgentはSC-Bench canonical Chromium tasksに対する自己評価で、normalization、HVG、clustering、annotation、DEはpassした一方、QCとtrajectoryはfailと報告している。この種のprojectは有用な実験台だが、peer-reviewed evidenceよりもsoftware prototypeとして扱うべきである。

Source:
- https://github.com/deepmind11/scAgent

# 3. MCP / function calling / tool-use

## 3.1 SCMCP

SCMCPはscRNA-seq解析のためのModel Context Protocol serverで、LLM clientから構造化されたtool callとして解析を呼び出す。提供範囲には以下が含まれる。

- I/O
- filtering / QC
- normalization / scaling / HVG
- PCA / neighbors
- clustering
- differential expression
- violin / heatmap / dotplot
- cell-cell communication
- pseudotime
- enrichment

tool modeでは事前定義functionだけを呼べるため、任意コード生成よりpredictableかつ安全にできる。code modeもあるが、本番運用ではtool modeを優先した方が監査しやすい。

Source:
- https://github.com/scmcphub/scmcp

## 3.2 汎用bioinformatics MCP

`bioinformatics-mcp`のようにScanpy、Squidpy、Biopython、NCBI、UniProt等をまとめたMCP serverも存在し、`scrna_standard_analysis`のようなguided workflowを提供している。またBioMCPはsingle-cell専用ではないが、PubMed/Reactome等のevidence retrievalをagentに構造化提供できるため、解析agentの「文献・経路解釈面」を補完できる。

Sources:
- https://github.com/raonyguimaraes/bioinformatics-mcp
- https://github.com/genomoncology/biomcp

## MCPの実務的利点

1. **ツールの入力・出力schemaを固定できる**
2. LLMに直接shellを渡すより権限を絞れる
3. 実行ログをtool invocation単位で残せる
4. Scanpy/Seuratの推奨wrapperを中央管理できる
5. 禁止操作やresource limitをサーバ側で強制できる

したがって、業務用途では「LLMが自由にPython/Rを生成するだけ」の構成より、**curated tool registry + restricted code sandbox**の二層構成が望ましい。

# 4. annotation専用LLM / multi-agent

## 4.1 GPTCelltype

Nature Methods 2024。marker gene/top DE geneをGPT-4に渡してcell type名を生成するR package。入力はexpression matrixそのものではなく、通常はクラスタリング後のmarker listである。

この研究は「一般LLMの既存生物知識がcell annotationに使える」ことを示した重要な初期例だが、解析workflow全体の自動化ではない。またmarker selectionの品質、tissue context、モデル更新による再現性が結果に影響する。

Source:
- https://doi.org/10.1038/s41592-024-02235-4

## 4.2 CASSIA

Nature Communications 2025。annotation、validation、formatting、quality scoring、reportingの複数LLMを組み合わせ、必要に応じてRAG、subclustering、uncertainty quantificationを追加する。970 cell populationsを用いた評価を報告しており、単一プロンプトLLMのhallucination/hyperconfidence対策を明示的に設計している。

CASSIAはSeuratのFindAllMarkersやScanpyの`rank_genes_groups`などで得たmarkerを利用するため、**annotation phaseのエージェント**であって、raw matrixからQCを含むfull pipeline agentではない。

Sources:
- https://doi.org/10.1038/s41467-025-67084-x
- https://github.com/ElliotXie/CASSIA

## 4.3 CyteType

2025 bioRxiv。multi-agentでcompeting hypothesesを作り、full expression dataとstudy contextを用いてexternal databaseで検証し、self-evaluationを繰り返す。Scanpy/AnnData版とSeurat版が公開されている。annotationのprovenanceとCell Ontology mappingを重視する。

Source:
- https://doi.org/10.1101/2025.11.06.686964

**実務上の使い方:** full autonomyではなく、primary annotation → marker/evidence review → ontology normalizationの補助として有用。最終ラベルはmarker expression、reference mapping、known biologyと突合する。

# 5. foundation model / large biological model

これらは「LLMに解析をさせる」話と関連するが、agentとは別カテゴリである。

## 5.1 scGPT

Nature Methods 2024。33 million超のcellを用いたgenerative pretrained transformer。cell type annotation、multi-batch integration、multi-omics integration、perturbation response prediction、gene network inference等をdownstream taskとして扱う。

Source: https://doi.org/10.1038/s41592-024-02201-0

## 5.2 scFoundation

Nature Methods 2024。約50 million cells規模のpretrainingを用い、cell/gene representation、cell type prediction、perturbation prediction、drug response等に適用する。

Source: https://doi.org/10.1038/s41592-024-02305-7

## 5.3 Geneformer

rank-value encodingしたgene expressionをTransformerで学習し、cell/gene embeddingやin silico perturbation等に使う。agentic workflowそのものではないが、annotationやfeature extractionのbackendとしてagentに組み込める。

## 5.4 Cell2Sentence / C2S-Scale

Cell2Sentenceは、発現量順にgene symbolを並べた「cell sentence」に変換し、既存language modelをsingle-cell taskへ適応する。ICML 2024ではcell generation、cell type annotation、data-driven text generationを示した。後続のC2S-Scaleはより大きなLLMへ拡張し、perturbation prediction、dataset summarization、cluster captioning、biological QA等を目指す。

Sources:
- ICML 2024: https://proceedings.mlr.press/v235/levine24a.html
- GitHub: https://github.com/vandijklab/cell2sentence

## 5.5 ChatCell / scMulan

ChatCellはCell2Sentence型の表現を利用して自然言語のsingle-cell taskを行うT5系モデル。scMulanはzero-shot annotation、batch integration、conditional generation等を目指すgenerative modelである。いずれも、通常は「解析手順のagent」ではなくモデル層として位置付ける方が正確。

# 6. CellWhisperer: transcriptome–language multimodal exploration

CellWhispererはNature Biotechnology 2025。約108万のRNA-seq profileとLLM-curated text annotationをcontrastive learningし、発現プロファイルと自然言語をjoint embeddingへ写像する。その上にchat modelを構築し、transcriptome-awareな自然言語探索を可能にする。

特徴は、marker gene listだけでなくexpression profile自体を自然言語と接続できる点である。ローカル版ではuser-supplied `.h5ad` を解析できる。初期探索やdataset captioning、cell search、annotationの支援には強力だが、QC thresholdの決定、pseudobulk DE、trajectory model selection等の統計ワークフローを自律的に逐次実行するagentとは異なる。

Sources:
- Nature Biotechnology 2025: https://doi.org/10.1038/s41587-025-02857-9
- GitHub: https://github.com/epigen/CellWhisperer

# 7. 汎用LLM＋コード実行環境

scRNA-seq専用agentを使わなくても、一般のcoding agentに次の環境を与えれば実質的に同様のことは可能である。

- Python + Scanpy + scvi-tools + decoupler + CellRank + LIANA+
- R + Seurat + Bioconductor packages
- Jupyter / Quarto / R Markdown
- filesystem access to `.h5ad`, 10x matrix, `.rds`
- literature/database tools

この方式の利点は最新のsingle-cell packageをすぐ利用できること、欠けている機能をコード生成で補えること、既存プロジェクトへ適応しやすいことである。一方、自由なコード生成はAPI hallucination、silent type conversion、random seedの不固定、package version drift、誤った統計モデル選択を起こしやすい。

この問題を直接測る重要なbenchmarkが**scBench**である。

# 8. scBenchと評価研究

scBench 2026は、実データsnapshotと自然言語taskを与え、agentがデータを実際に操作して正しい結果へ到達できるかをdeterministic graderで評価する。taskはQC、normalization、dimensionality reduction、clustering、cell typing、differential expression等を含む。

arXiv paperは**394 verifiable problems、6 sequencing platforms、7 task categories**を記載している。一方、2026-08-27時点の公開GitHub READMEは**195 evaluations**を掲げており、公開セットやrepository versionが変化しているため、数値を引用する際はpaper versionとrepo snapshotを明示すべきである。

公開repoの現行leaderboardではtop agent/model combinationでも約58%程度であり、frontier coding agentであっても**4割程度のtaskを落とす**。arXiv版の報告でもモデル間・platform間の差が大きく、documentationが少ないsequencing platformでは40 percentage points以上の低下が見られるとされる。

これは現時点の最重要な実務メッセージである。

> LLM agentは「コードを書ける」ことと「実データで正しいsingle-cell解析を完遂できる」ことが同義ではない。

Source:
- arXiv: https://arxiv.org/abs/2602.09063
- GitHub: https://github.com/latchbio/scbench

# 9. タスク別の現実的な自律性

| タスク | LLM/agent適性 | 主な失敗要因 | 推奨運用 |
|---|---|---|---|
| データ読込・format変換 | 高い | gene/cell axis、sparse matrix、raw layer取り違え | schema validation必須 |
| QC metric計算 | 高い | metric定義差 | 自動可 |
| QC filtering閾値決定 | 中〜低 | dataset-specific、pre-filtered data | 人間承認 |
| normalization/HVG/PCA | 高い | assay/model依存 | preset + validation |
| batch correction | 中 | biologyとbatchのconfounding | 複数手法比較 + human review |
| clustering | 中 | resolution恣意性、stability | stability/marker確認 |
| cell annotation | 中〜高 | rare state、disease state、hallucination | multi-evidence + uncertainty |
| differential expression | 中 | pseudoreplication、design matrix、paired structure | sample-level designを人間承認 |
| pathway/enrichment | 高いが解釈注意 | background universe、database bias | parameter記録 |
| trajectory/pseudotime | 低〜中 | root choice、topology assumption | domain review必須 |
| RNA velocity | 低〜中 | kinetics assumptions、model choice | diagnostics必須 |
| cell-cell communication | 中 | method disagreement、expression≠interaction | ensemble/validation |
| visualization | 高い | misleading scale/selection | automated + review |
| biological interpretation | 中 | hallucination、citation errors | RAG + primary source verification |
| hypothesis generation | 高い | false discovery、story bias | exploratoryと明示 |

# 10. 統計・科学的妥当性の主要リスク

## 10.1 pseudoreplication

最も危険な失敗の一つ。cellを独立replicateとして扱ったWilcoxon testだけでcondition差を主張すると、サンプル間変動を無視して過度に小さいP値を得る。LLM agentには「DE taskを見たらpseudobulkまたはsample-aware modelを優先する」というpolicyを固定するべきである。

## 10.2 QCの固定閾値

`n_genes < 5000`、`pct_mt < 5%`のような教科書的cutoffを無条件に適用すると、tissue、chemistry、nuclei/cell、disease contextにより有用細胞を失う。scAgentのSC-Bench失敗例でも、すでにcleaned datasetへ一般閾値を適用したことがQC failureとして示されている。

## 10.3 batch correctionによるbiology removal

LLMは「batchがある→Harmony/scVIを実行」と単純化しやすい。conditionとbatchが共線の場合、補正はbiological signalを消す可能性がある。design metadataを先に検査し、`sample × condition × batch`の交絡を明示する必要がある。

## 10.4 cluster resolutionとannotationの循環

「markerがきれいになるまでresolutionを上げる」ような最適化は、同じデータでcluster choiceとannotation qualityを自己評価する循環を生む。stability、biological replicability、reference mapping、marker consistencyを別々に評価する。

## 10.5 trajectory / CCCのstory bias

trajectoryやcell-cell communicationは特に「もっともらしい物語」を生成しやすい。root cell、graph topology、ligand-receptor resource、expression threshold等の仮定を明記し、alternative methodsでrobustnessを確認する。

# 11. Hallucination、再現性、プライバシー

## Hallucination

- 存在しないScanpy/Seurat function名
- 廃止されたparameter
- gene-marker関係の誤記
- 文献DOIの捏造
- エラーが出ないが意味的に誤ったlayer/slotの利用

対策は「LLMに詳しく考えさせる」より、**tool schema、unit/integration test、result assertions、official docs retrieval**を実装する方が強い。

## Reproducibility

最低限記録すべきもの:

- input file checksum / object snapshot ID
- genome build / gene ID namespace
- software versions
- container image digest
- Python/R lock file
- random seed
- 全tool callsとparameters
- generated code
- stdout/stderr
- intermediate AnnData/Seurat snapshots
- figures and tables
- user approvals / overrides
- LLM provider/model/version
- prompt/template version

## Data privacy

single-cell expression matrixは研究対象によっては機微なhuman molecular dataである。クラウドLLMへraw matrixを直接送る必要は通常ない。Gardenerのように、**data planeはローカル、LLMへはsanitized summaryのみ**という構成が望ましい。

# 12. 実務利用するなら推奨するアーキテクチャ

## 推奨: “LLM control plane + validated local data plane”

```text
User / bioinformatician
        |
        v
LLM planner / conversational interface
        |
        +--> literature/RAG tools (primary sources)
        |
        v
Policy + tool registry (MCP/function calling)
        |
        +--> QC tools
        +--> Scanpy / scvi-tools
        +--> Seurat / Bioconductor
        +--> DE / pseudobulk
        +--> CellRank / velocity
        +--> LIANA+ / enrichment
        |
        v
Containerized local execution
        |
        v
Immutable analysis state / artifact store
        |
        +--> AnnData/Seurat snapshots
        +--> code + params + logs
        +--> figures/tables
        +--> provenance manifest
        |
        v
Human approval gates
        |
        v
Report / notebook / reproducible pipeline
```

## 12.1 LLMはcontrol planeに置く

LLMに向いているのは、質問理解、分析計画、tool selection、エラー解釈、結果要約、次の候補生成である。matrix演算や統計計算そのものは、既存libraryに実行させる。

## 12.2 tool registryは「狭いAPI」を優先

本番で直接`python`/`bash`を無制限に与えず、まず以下のようなvalidated toolを用意する。

- `inspect_anndata()`
- `compute_qc_metrics()`
- `propose_qc_thresholds()`
- `apply_qc_filter()`
- `normalize_log1p()`
- `run_scvi()`
- `cluster_leiden()`
- `rank_markers()`
- `run_pseudobulk_de()`
- `run_enrichment()`
- `run_cellrank()`
- `run_liana()`

自由コードは「tool registryで表現できない例外」に限定する。

## 12.3 human approval gate

少なくとも以下を承認点にする。

1. **QC filtering before/after summary**
2. **batch correction strategy**
3. **cluster resolution / merge-split decision**
4. **cell annotation**
5. **sample-level statistical design**
6. **trajectory root / direction**
7. **final biological claims**

## 12.4 immutable state

GardenerのEMKの考え方を採用し、会話履歴を解析状態のSSOTにしない。各stepでAnnData/Seurat objectのsnapshotを保存し、parent-child lineageを持たせる。

例:

```text
S0 raw
 └─S1 QC metrics
    ├─S2a strict filter
    │  └─S3a scVI integration
    └─S2b lenient filter
       └─S3b Harmony integration
```

これにより、LLMが途中で方針を変えても再現可能になる。

## 12.5 container + lockfile

- Docker/Apptainer image digest固定
- `uv.lock` / `conda-lock` / `renv.lock`
- Scanpy/scvi-tools/Seuratのversion固定
- notebookだけでなく、最終runはscript/workflowへ昇格

必要ならSnakemake/Nextflowでproduction pipelineを生成し、LLMはpipeline definitionを編集する役割にする。

## 12.6 automated tests

agentの出力を自然言語評価だけにしない。例:

- cell countが0になっていない
- `.raw` / `.layers['counts']`が期待通り残る
- PCA/neighbor graphが存在する
- conditionごとに複数sampleが存在するか
- pseudobulkでsample IDが使われているか
- DE contrastがdesign matrixと一致するか
- cluster labelが一意か
- marker listにgene ID mismatchがないか

## 12.7 benchmark gate

導入前に、自組織で使うchemistry/tissueに近いknown-answer datasetを用意し、scBench型のdeterministic gradingを行う。frontier modelを更新するたびにregression testを回す。

# 13. 成熟度評価（2026-08-27）

| 手法/ツール | 主用途 | 自律性 | 実行環境 | エビデンス | 成熟度 |
|---|---|---:|---|---|---|
| CellAgent | end-to-end scRNA/ST | 高 | tool execution | ICLR 2026 + benchmark | 実験利用 |
| CellVoyager | 自律探索・仮説生成 | 高 | Jupyter | Nature Methods 2026 | 実験利用 |
| scChat | context-aware co-pilot | 中〜高 | AnnData + tools | AIChE J 2026 | 実験利用 |
| Gardener | privacy/provenance重視agent | 中〜高 | local engine + cloud LLM | ACL 2026 demo | 実務設計候補 |
| CellPilot | raw→annotation | 中〜高 | local SLM + tools | bioRxiv 2026 | 研究デモ |
| SCMCP | MCP tool layer | LLM依存 | local MCP server | OSS | 実務部品候補 |
| GPTCelltype | annotation | 低 | API | Nature Methods 2024 | 限定的実務候補 |
| CASSIA | multi-agent annotation | 中 | R/Python/API | Nature Communications 2025 | 実務候補（annotation） |
| CyteType | evidence-based annotation | 中 | R/Python | bioRxiv 2025 | 実験利用 |
| CellWhisperer | natural-language exploration | 低〜中 | local/web | Nature Biotechnology 2025 | 実務候補（探索） |
| scGPT/scFoundation | representation/downstream ML | なし | model inference | Nature Methods 2024 | model部品 |
| Cell2Sentence | cell-language model | なし | LLM inference/training | ICML 2024 | 研究/モデル部品 |
| 汎用coding agent | 任意解析 | 高 | shell/Jupyter | scBenchで直接評価 | 実験利用。要guardrail |

# 14. バイオインフォマティクスコンサルティングでの推奨構成

現時点で単一の「自律scRNAエージェント製品」に全面依存するのは推奨しない。代わりに以下を組み合わせる。

### ベース解析

- Scanpy/scverseまたはSeuratをcanonical engineにする。
- standard operating procedureをtool化する。
- raw counts、normalized data、latent representationを明示的に分離する。

### agent layer

- frontier LLMをplanner/reviewerとして利用。
- MCP/function callingでvalidated toolsを呼ぶ。
- 例外時だけsandboxed code executionを許可。

### annotation

- marker-based reasoning + reference mapping + CASSIA/CyteType等のLLM annotationをensemble的に使う。
- Cell Ontologyへ正規化。
- uncertaintyを保持し、無理に全clusterへ単一labelを付けない。

### biological interpretation

- CellWhispererやLLM-RAGは探索に利用可能。
- 最終claimにはprimary literatureを付ける。
- agent-generated hypothesisはexploratoryと明示。

### provenance

- Gardener型のimmutable snapshot lineageを導入。
- 最終成果物はNotebookだけでなくscript/workflow/containerで再実行可能にする。

### security

- raw molecular dataはlocal/VPC内。
- cloud LLMにはsummary、marker、aggregate statistics等のみ送信。
- tool permissionをleast privilegeにする。

# 15. 結論

2024年には「GPT-4でmarker geneからcell typeを答える」段階が中心だったが、2026年には、LLMが**計画し、コード/ツールを実行し、結果を見て修正し、仮説まで生成する**agentic single-cell analysisが現実の研究分野になっている。CellAgent、CellVoyager、scChat、Gardener、CellPilotはそれぞれ、end-to-end automation、scientific exploration、context-aware reasoning、provenance/privacy、local small-model orchestrationという異なる設計軸を示している。

しかし、scBenchが示すように、frontier agentでも実データtaskの成功率はまだ完全自律運用を支持する水準ではない。特にsingle-cell解析では、コードの正しさ以上に、**実験単位、統計設計、交絡、QC、データ生成過程、生物学的仮定**の理解が必要である。

したがって2026年時点の最適解は、**autonomous scientistではなく audited computational co-pilot** として設計することである。LLMには柔軟な計画・説明・探索を担当させ、実行は検証済みsingle-cell toolsに限定し、解析状態・provenanceを外部化し、重要な科学判断にhuman approvalを入れる。この構成なら、LLMの速度と柔軟性を活かしつつ、scRNA-seq解析に必要な再現性・統計的妥当性・機密性を維持しやすい。

# Sources

## Agentic / workflow systems

- Alber S, et al. *CellVoyager: AI CompBio agent generates new insights by autonomously analyzing biological data.* Nature Methods. 2026. https://doi.org/10.1038/s41592-026-03029-6
- Xiao Y, et al. *CellAgent: LLM-Driven Multi-Agent Framework for Natural Language-Based Single-Cell Analysis.* ICLR 2026. https://openreview.net/forum?id=BsA2GNkJhz
- Liu J, et al. *Gardener: An Agentic AI System for Single-Cell RNA Sequence Analysis.* ACL 2026 System Demonstrations. https://doi.org/10.18653/v1/2026.acl-demo.40
- Chiu H-H, et al. *scChat: A large language model-powered co-pilot for contextualized single-cell RNA sequencing analysis.* AIChE Journal. 2026. https://doi.org/10.1002/aic.70285
- Jiang S, et al. *CellPilot: an agentic framework that pilots small language models through autonomous single-cell annotation.* bioRxiv. 2026. https://doi.org/10.64898/2026.07.06.736807
- SCMCP: https://github.com/scmcphub/scmcp
- CellVoyager code: https://github.com/zou-group/CellVoyager
- scBench: https://github.com/latchbio/scbench

## Benchmarks

- Workman K, et al. *scBench: Evaluating AI Agents on Single-Cell RNA-seq Analysis.* arXiv:2602.09063. https://arxiv.org/abs/2602.09063

## Annotation-focused LLM systems

- Hou W, Ji Z. *Assessing GPT-4 for cell type annotation in single-cell RNA-seq analysis.* Nature Methods. 2024. https://doi.org/10.1038/s41592-024-02235-4
- Xie E, et al. *CASSIA: a multi-agent large language model for automated and interpretable cell annotation.* Nature Communications. 2025. https://doi.org/10.1038/s41467-025-67084-x
- Ahuja G, et al. *Multi-agent AI enables evidence-based cell annotation in single-cell transcriptomics.* bioRxiv. 2025. https://doi.org/10.1101/2025.11.06.686964

## Foundation / multimodal models

- Cui H, et al. *scGPT: toward building a foundation model for single-cell multi-omics using generative AI.* Nature Methods. 2024. https://doi.org/10.1038/s41592-024-02201-0
- Hao M, et al. *Large-scale foundation model on single-cell transcriptomics.* Nature Methods. 2024. https://doi.org/10.1038/s41592-024-02305-7
- Levine D, et al. *Cell2Sentence: Teaching Large Language Models the Language of Biology.* ICML 2024. https://proceedings.mlr.press/v235/levine24a.html
- Schaefer M, et al. *Multimodal learning enables chat-based exploration of single-cell data.* Nature Biotechnology. 2025. https://doi.org/10.1038/s41587-025-02857-9

# Review notes

- Reviewed: 2026-08-27.
- この分野は更新が速く、agent/model leaderboardやsoftware capabilityは数か月単位で変化する可能性が高い。
- scBenchはpaper記載の394 tasksと公開GitHub README記載の195 evaluationsに差があるため、将来の引用時には利用したversion/snapshotを確認する。
- 新しいagentを比較するときは、著者自身のbenchmarkだけでなく、独立benchmark、実データでのfailure analysis、statistical designの妥当性を優先して評価する。
