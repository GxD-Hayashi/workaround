# イレギュラー発生時の対応
## Case1. PureCN エラー終了時の手順
解析フォルダのファイル操作を伴うため、**全ての工程は gxd_pipeline ユーザーで実行する。**
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

## Case2. STAR-SEQR 超過時の手順
解析フォルダのファイル操作を伴うため、**全ての工程は gxd_pipeline ユーザーで実行する。**
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
*STAR-SEQR実行中の場合のみ実施。**すでにタイムアウトしている場合はスキップする。**
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





