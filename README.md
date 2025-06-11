# special-case
## PureCNエラー終了時の対応手順
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
