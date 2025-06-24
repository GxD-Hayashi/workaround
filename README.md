# イレギュラー発生時の対応
解析フォルダのファイル操作を伴うため、**全ての工程は gxd_pipeline ユーザーで実行する。**
## case1. PureCN エラー終了時の手順
eWES Pipeline CNV解析工程において PureCN の実行時に purity/ploidy の算出ができずに途中終了することがある。\
2025/6/6 時点では、bin size 400,800,1600のうちいずれか1つだけエラー終了するケースが確認されています。
<details>
  <summary> 
    More Details
  </summary>

### 1\. 変数の設定
```
WORKDIR=/data1/data/result/eWES
SNAKEFILE=/data1/GxD_eWES/Pipeline/workflow/Snakefile
batch=
sample=
```
batch : 当該検体のbatchフォルダ名 \
sample :当該検体のSample ID
### 2\. bin sizeの設定
PureCNを完了した bin size について、\
&nbsp;&nbsp;&nbsp;&nbsp; **[WORKDIR]/[batch]/[sample]/CNV/PureCN/[bin_size]/${sample}.tumour.exome.purecn.csv** \
の数値を比較し、後続の解析で採用するbin sizeを決定する。\
PureCNが実行できたものが2つで、Purityの値が近い場合はbin sizeが小さい方を採用。\
PureCNが実行できたものが2つで、Purityの値が大きく異なる場合はGSの判断を仰ぐ。\
PureCNが実行できたものが1つだけの場合、GSの判断を仰ぐ。
```
bin_size=
```
bin_size : 400,800,1600 のうち1つを採用
### 3\. データの編集
残りの工程の実行に必要なファイルを作成する。
```
cd ${WORKDIR}/${batch}/${sample}/CNV/PureCN
echo -n ${bin_size} > bin_size.txt
ln -s `pwd`/${bin_size}/${sample}.tumour.deduped_coverage_loess.png ./
ln -s `pwd`/${bin_size}/${sample}.tumour.deduped_coverage_loess_qc.txt ./
ln -s `pwd`/${bin_size}/${sample}.tumour.deduped_coverage_loess.txt.gz ./
ln -s `pwd`/${bin_size}/${sample}.tumour.deduped_coverage.txt.gz ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_amplification_pvalues.csv ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_chromosomes.pdf ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn.csv ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_dnacopy.seg ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_genes.csv ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_local_optima.pdf ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn.log ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_loh.csv ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn.pdf ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn.rds ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_segmentation.pdf ./
ln -s `pwd`/${bin_size}/${sample}.tumour.exome.purecn_variants.csv ./
```
### 4\. 後工程の実行
snakemake実行用の環境に入る
```
source /data1/iGeniPipe/miniconda3/bin/activate cs
```
snakemake dry run で実行されるコマンドを確認する。purecn_purecn,purecn_mergeを実行しないこと、cnv_tier以降が実行されることを確認する。
```
snakemake --dry-run --snakefile $SNAKEFILE --directory /data1/GxD --profile /data1/GxD_eWES/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch}
```
解析の続きを実行する
```
snakemake --snakefile $SNAKEFILE --directory /data1/GxD --profile /data1/GxD_eWES/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch} &
```
仮想環境から出る
```
conda deactivate
```
</details>

## case2. STAR-SEQR 超過時の手順
WTS Pipeline Fusion解析工程において、STAR-SEQRが長時間かかる場合がある。\
200時間を超えるとタイムアウトする可能性があるとのこと。
[STAR-SEQR issue](https://github.com/ExpressionAnalysis/STAR-SEQR/issues/23)
<details>
  <summary> 
    More Details
  </summary>

### 1\. 変数の設定
```
WORKDIR=/data1/data/result/WTS
SIF=/data1/GxD_WTS/Pipeline/containers/metafusion.sif
SCRIPT=/MetaFusion/scripts/convert_fusion_results_to_cff.py
batch=
sample=
```
batch : 当該検体のbatchフォルダ名 \
sample : 当該検体のSample ID
### 2\. STAR-SEQR 進捗状況の確認
ログの最終行に以下の文字列が含まれていることを確認する。（融合候補の相同性を計算する工程。STAR-SEQRが終了しない場合はここでスタックしている可能性が高い）
>	INFO - Getting fusions homology mapping scores
```
ls -t ${WORKDIR}/${batch}/${sample}/Logs/*.${sample}.starseqr_[0-9]*.err | head -1 | tail -4
```
または、ログファイルに以下の文字列が出現することを確認する。(chimeric transcriptsの書き出し終了フラグ)
>	INFO - Writing chimeric transcripts
```
grep "INFO - Writing chimeric transcript" `ls -t  ${WORKDIR}/${batch}/${sample}/Logs/*.${sample}.starseqr_[0-9]*.err | head -1`
```
### 3\. 実行ジョブの削除 
qstat -r で実行中のジョブを確認し、job name が [sample].starseqr_[0-9] があれば qdel で強制終了する。\
※ STAR-SEQR実行中の場合のみ実施。**すでにタイムアウトしている場合はスキップする。**
### 4\. convert_cff 工程の実行
STAR-SEQR の結果ファイルが作成されず、後続の convert_cff工程でエラー終了するため、この工程を手作業で実行する。\
arriba/STAR-Fusionの結果ファイルをcff形式に整形する。
```
mkdir ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion 
singularity exec --bind /data1 $SIF $SCRIPT ${sample} - Tumor arriba ${WORKDIR}/${batch}/${sample}/Fusion/Arriba/${sample}.fusions.tsv ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion
singularity exec --bind /data1 $SIF $SCRIPT ${sample} - Tumor star_fusion ${WORKDIR}/${batch}/${sample}/Fusion/STAR-Fusion/star-fusion.fusion_predictions.abridged.coding_effect.tsv ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion
```
STAR-SEQR のcffファイルはダミーを作成する。
```
yes NA | head -n 17 | paste -sd '\t' > ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion/${sample}.star_seqr.cff
```
### 5\. 後工程の実行
STAR-SEQR工程を明示的にスキップしてPipelineを実行するsnakefileを利用する。\
&nbsp;&nbsp;&nbsp;&nbsp; /data1/GxD_WTS/Pipeline/workflow/Snakefile_prevent
snakemake実行用の環境に入る。
```
source /data1/iGeniPipe/miniconda3/bin/activate cs
```
必要に応じて --unlock オプションで作業ディレクトリのロックを解除する。
```
snakemake --unlock --snakefile /data1/GxD_WTS/Pipeline/workflow/Snakefile_prevent --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch}
```
snakemake dry run で実行されるコマンドを確認する。\
starseqrが実行されないこと、merge_cff以降が実行されることを確認する。
```
snakemake --dry-run --snakefile /data1/GxD_WTS/Pipeline/workflow/Snakefile_prevent --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch}
```
snakemake実行
```
snakemake --snakefile /data1/GxD_WTS/Pipeline/workflow/Snakefile_prevent --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch} &
```
仮想環境からでる。
```
conda deactivate
```
</details>

## case3. Fusion不検出による解析中断
arriba, STAR-Fusion, STAR-SEQR の出力結果のうち、いずれか1つ以上のツールでFusionが検出されず rule: convert_cff で出力されるcffが空ファイルとなった場合にエラー終了する。
<details>
  <summary> 
    More Details
  </summary>
  
### 1\. 変数の設定
```
WORKDIR=/data1/data/result/WTS
SNAKEFILE=/data1/GxD_WTS/Pipeline/workflow/Snakefile
batch=
sample=
```
batch : 当該検体のbatchフォルダ名 \
sample : 当該検体のSample ID

### 2\. 解析の進捗確認
qstatで当該検体の解析が実行中でないことを確認したのち、Logsフォルダに出力されている最新の *.${sample}.merge_cff_*.err の中身を確認し、merge_cff 工程がエラー終了していることを確認する。
```
cd $WORKDIR/$batch/$sample/Logs
ll -t *.${sample}.merge_cff_*.err
```

### 3\. Fusionが検出されていないツールを特定する
*/Fusion/Metafusion/ の直下に各Fusion検出ツールの結果をcff形式に変換したものが作成されている。
```
cd $WORKDIR/$batch/$sample/Fusion/Metafusion
ll ${sample}.*.cff
```
Fusionが検出されなかった場合はデータサイズが0になる。ツールに対応する出力ファイル名は以下の通り。\
&nbsp;&nbsp;&nbsp;&nbsp;STAR-Fusionの出力結果: ${sample}.star_fusion.cff \
&nbsp;&nbsp;&nbsp;&nbsp;STAR-SEQRの出力結果: ${sample}.star_seqr.cff \
&nbsp;&nbsp;&nbsp;&nbsp;arribaの出力結果: ${sample}.arriba.cff 

### 4\. 中間ファイルの作成
データサイズが0のもののみcffファイルを作成する。**ファイルサイズが0以上のものを上書きしないように注意する**\
STAR-Fusionの場合
```
yes NA | head -n 17 | paste -sd '\t' > ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion/${sample}.star_fusion.cff
```
STAR-SEQRの場合
```
yes NA | head -n 17 | paste -sd '\t' > ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion/${sample}.star_seqr.cff
```
arribaの場合
```
yes NA | head -n 17 | paste -sd '\t' > ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion/${sample}.arriba.cff
```
### 5\. 後工程の実行
snakemake実行用の環境に入る
```
source /data1/iGeniPipe/miniconda3/bin/activate cs
```
snakemake dry run で実行されるコマンドを確認する。convert_cffを実行しないこと、merge_cff以降が実行されることを確認する。
```
snakemake --dry-run --snakefile $SNAKEFILE --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch}
```
解析の続きを実行する
```
snakemake --snakefile $SNAKEFILE --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch} &
```
仮想環境から出る
```
conda deactivate
```

</details>

## case4. 解析結果の修正とレポートの再作成（手作業）
検出された変異等を<ins>**削除**</ins>する場合は worksheet ツールの remove コマンドを利用して解析結果を修正できるが、
検出された変異の<ins>**報告内容を変更**</ins>する場合(Oncogenicityの変更など)はsummaryファイルを手作業で修正し、
データベースの書き換えとレポートの再作成を実施する必要がある。\
OncoStation上でComfirm済みの場合は、GSにComfirmを取り下げてもらってから作業すること。
<details>
  <summary> 
    More Details
  </summary>

### 1\. 変数の設定
```
WORKDIR=/data1/data/result
test_type=
batch=
sample=
SIF=/data1/GxD_${test_type}/Pipeline/containers/inhouse.sif
```
test_type : 解析種別。eWESまたはWTS \
batch : バッチフォルダ名 \
sample : 当該検体のSample ID
### 2\. データの編集
解析結果を格納しているフォルダに移動してsummaryファイルを編集する \
 - type1 eWES SNV & InDelの編集
```
cd ${WORKDIR}/${batch}/${sample}/Summary
cp ${sample}.summarized.snv.target.tsv ${sample}.summarized.snv.target.original.tsv
vi ${sample}.summarized.snv.target.tsv
```
 - type2 eWES SNV/InDel with Insufficient Depthの編集
```
cd ${WORKDIR}/${batch}/${sample}/Summary
cp ${sample}.summarized.snv.exome.tsv ${sample}.summarized.snv.exome.original.tsv
vi ${sample}.summarized.snv.exome.tsv
```
 - type3 eWES CNVの編集
```
cd ${WORKDIR}/${batch}/${sample}/Summary
cp ${sample}.summarized.cnv.exome.tsv ${sample}.summarized.cnv.exome.original.tsv
vi ${sample}.summarized.cnv.exome.tsv
```
 - type4 WTS Fusionの編集
```
cd ${WORKDIR}/${batch}/${sample}/Summary
cp ${sample}.summarized.fusion.tsv ${sample}.summarized.fusion.original.tsv
vi ${sample}.summarized.fusion.tsv
```
※ type1-4 は不要な変異の行を削除して上書き保存（DRUG が複数該当する場合は、該当するものすべて削除する）

 - type5 WTS Alternative Splicingの編集
```
cd ${WORKDIR}/${batch}/${sample}/Summary
cp ${sample}.summarized.splice.tsv ${sample}.summarized.splice.original.tsv
vi ${sample}.summarized.splice.tsv
```
※ type5 は不要な変異の2カラム目以降をblankにして上書き保存（1列目の値はレポートに使用するので、行削除ではなく値を削除する）

### 3\. レポート再作成の準備
データベースに登録済みの解析結果を削除して report.json, report.pdfをリネームし、analysis statusを101（解析中）にセットする。\
worksheet ツールの resetコマンドを使用。
```
worksheet reset --sample ${sample} –status 101
```
エイリアス未作成の場合
```
singularity exec --bind /data1 /data1/labTools/labTools.sif python /data1/labTools/worksheet/latest/worksheet.py reset --sample ${sample} –status 101
```
**※ Pipeline、reference、コンテナファイル等が初回解析時と同じ場合はcronの自動実行を利用してもよい。**\
初回解析時から変更があった場合は、変更に関連した工程から再実行して解析結果を上書きすることに注意。\
データベースに登録済みの解析結果を削除して report.json, report.pdfをリネームし、analysis statusを100（解析待ち）にセットする。\
worksheet ツールの resetコマンドを使用。
```
worksheet reset --sample ${sample} –status 100
```
エイリアス未作成の場合
```
singularity exec --bind /data1 /data1/labTools/labTools.sif python /data1/labTools/worksheet/latest/worksheet.py reset --sample ${sample} –status 100
```
⇒ cronにより10分以内に解析が開始され、report_json 工程のみ実施される。

### 4\. データベースへの再アップロードとレポート再作成
前工程でanalysis statusを101（解析中）にセットした場合に実行する。解析実行時のPipelineバージョンがデフォルトとは異なる場合は、**解析実行時のPipelineバージョンのmodulesのmain.pyファイルを指定する**こと。
```
singularity shell --bind /data1 $SIF python3 /data1/GxD_${test_type}/Pipeline/modules/report_json/main.py -s ${sample} -d ${WORKDIR}/${test_type}/${batch}/${sample}/Summary -o ${WORKDIR}/${test_type}/${batch}/${sample}/Summary/${sample}.report.json -r ${WORKDIR}/${test_type}/${batch}/${sample}/Summary/${sample}.report.pdf -c True -u 192.168.9.100 -p 3014 -v v1.1.0 --upload true --start_log ${WORKDIR}/${test_type}/${batch}/${sample}/QC/fastp/${sample}.fastp.start.time.log
```
</details>
