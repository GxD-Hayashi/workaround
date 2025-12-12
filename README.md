# イレギュラー発生時の対応
以下の事象が確認された場合、**GMに報告・了承を得た上で**適宜データを編集し、解析を再実行してください。\
記載する手順は2025年12月時点のものです。またSOPではありません(=正式な手順ではありません)ので、状況に合わせて適宜変更してください。\
また、解析フォルダのファイル操作を伴うため、**全ての工程は gxd_pipeline ユーザーで実行してください。**

## 解析が途中終了しているかどうかの確認方法
 ① worksheet ツールの checkコマンドで解析実行中(ANALYSIS STATUS=101)になっているSampleIDを確認する \
 ② OncoStation の Clinical Report ページで解析の進捗を表示する項目「Progress」が2つめで止まっているSampleIDを確認する \
 ③ qstatコマンドを利用して実際に投入されている実行中のジョブIDを確認する ※タイムラグがあるので、qstatは数回実行して確認する
```
worksheet check ‐fc <flowcellid>
qstat -r | grep Full | cut -f1 -d "." | sort | uniq
```
①③ または ②③ のSampleIDを比較し、解析実行中のはずが実際にはジョブが投入されていない場合、解析が途中で終了している可能性が高い。\
途中終了している検体があった場合は以下の手順で原因を特定する。

#### 1\. 変数の設定
```
WORKDIR=/data1/data/result
test_type=
batch=
sample=
```
test_type : 解析種別。eWESまたはWTS \
batch : バッチフォルダ名 \
sample : 当該検体のSample ID

#### 2\. ログファイルの確認
解析結果を格納しているフォルダに移動してログファイルを確認する 
```
cd $WORKDIR/$test_type/$batch/$sample/Logs
ll -t *.err | less
```
更新履歴が新しい順で .err ファイルが表示される。※最新の .err を作成した rule がパイプラインの途中終了の原因とは限らないことに注意。\
*.err ファイルの中身を確認する。**最終行が「1 of 1 steps (100%) done」でないものは正常終了できなかったもの。**\
*.err のファイル名からどの工程で止まったかを推測し、中断された原因に沿って対応する。

|エラーログファイル                 |エラーの原因              |対応                      |
|:---------------------------------|:------------------------|:-------------------------|
|\*.[sampleID].purecn_merge_\*.err |採用する bin size が決定できなかった |[case2](#case2) |
|\*.[sampleID].purecn_purecn_\*.err|purecn 実行エラー         |[case2](#case2)           |
|\*.[sampleID].merge_cff_\*.err    |Fusion不検出による解析中断 |[case3](#case3)           |
|\*.[sampleID].starseqr_\*.err     |STAR-SEQR 超過により解析が進まない ※ジョブは実行中     |[case4](#case4) |
|\*.[sampleID].starseqr_\*.err     |breakpointの候補が1つもなかったため処理が中断された     |[case5](#case5) |
|上記以外                           |同じノードに高負荷なジョブが投入されたことによる中断     |[case1](#case1) |

## 開発チームへのデータ提供
レビューで結果保留になった場合などに、開発チームへデータを提供して調査してもらうケースがあります。\
RUO Strage (/data3/CAP/) に提供するデータをコピーしたあと、開発チームに連絡するようGMに依頼してください。\
提供するデータについて、よくあるケースを以下に示します。
<details>
  <summary> 
    More Details
  </summary>

#### 【eWES】SNV & InDel について問い合わせる場合
再計算した *.bam と *.bam.bai を提供します。
```
WORKDIR=/data1/data/result
batch=
sample=

mkdir -p /data3/CAP/[提供する年月日(8桁数字)]
rsync -avzru $WORKDIR/eWES/$batch/$sample/Preprocessing/align/${sample}.tumour.recaled.bam* /data3/CAP/[提供する年月日(8桁数字)]/
```
#### 【WTS】Fusion について問い合わせる場合
Fusion 検出時の途中ファイルと、STAR-FusionでFusion検出時に作成されるSAMをBAMに変換し、indexを作成して提供します。
```
WORKDIR=/data1/data/result
batch=
sample=

mkdir -p /data1/work/[提供する年月日(8桁数字)]
rsync -avzru $WORKDIR/WTS/$batch/$sample/Fusion/${sample}.fusion.filtered.tsv /data1/work/[提供する年月日(8桁数字)]/
rsync -avzru $WORKDIR/WTS/$batch/$sample/Fusion/Arriba/${sample}.fusions.tsv /data1/work/[提供する年月日(8桁数字)]/
rsync -avzru $WORKDIR/WTS/$batch/$sample/Fusion/STAR-Fusion/star-fusion.fusion_predictions.abridged.coding_effect.tsv /data1/work/[提供する年月日(8桁数字)]/
samtools view -bh $WORKDIR/WTS/$batch/$sample/Fusion/STAR-Fusion/STAR_align_starfu/${sample}.star-fusion.Aligned.out.sam | samtools sort -@ 12 -o /data1/work/[提供する年月日(8桁数字)]/${sample}.star-fusion.Aligned.out.bam -
samtools index /data1/work/[提供する年月日(8桁数字)]/${sample}.star-fusion.Aligned.out.bam
mv /data1/work/[提供する年月日(8桁数字)] /data3/CAP/[提供する年月日(8桁数字)]
```
</details>

その他、開発の要求に応じてデータを送付してください。※個人情報保護の観点から、要求されたデータの提供についてはGMに許可をもらうこと

<a id="case1"></a>
## case1. 高負荷による実行停止
高負荷なプロセスが同時間帯にひとつの計算ノードに投入され、メモリが超過して当該ノードで実行中のジョブがkillされた結果、パイプラインが途中終了することがある。\
サーバーの空き容量が十分にあればそのまま再実行すればよい。
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
```
test_type : 解析種別。eWESまたはWTS \
batch : バッチフォルダ名 \
sample : 当該検体のSample ID

### 2\. 解析の再実行
```
sh $WORKDIR/$test_type/$batch/$sample/run.sh
```
</details>

<a id="case2"></a>
## case2. PureCN エラー終了時の手順
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

<a id="case3"></a>
## case3. Fusion不検出による解析中断 
WTS Pipeline Fusion解析工程において、Arriba, STAR-Fusion, STAR-SEQR の出力結果のうち、いずれか1つ以上のツールでFusionが検出されず rule: convert_cff で出力されるcffが空ファイルとなった場合にエラー終了する。
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
qstatで当該検体の解析が実行中でないことを確認したのち、Logsフォルダに出力されている最新の \*.${sample}.merge_cff_*.err の中身を確認し、merge_cff 工程がエラー終了していることを確認する。
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
仮想環境から出る
```
conda deactivate
```
解析の続きを実行する
```
sh ${WORKDIR}/${batch}/${sample}/run.sh
```
</details>

<a id="case4"></a>
## case4. STAR-SEQR 超過時の手順
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
tail -4 `ls -t ${WORKDIR}/${batch}/${sample}/Logs/*.${sample}.starseqr_[0-9]*.err | head -1`
```
または、ログファイルに以下の文字列が出現することを確認する。(chimeric transcriptsの書き出し終了フラグ)
>	INFO - Writing chimeric transcripts
```
grep "INFO - Writing chimeric transcript" `ls -t  ${WORKDIR}/${batch}/${sample}/Logs/*.${sample}.starseqr_[0-9]*.err | head -1`
```

### 3\. 実行ジョブの削除 ※すでにタイムアウトしている場合はスキップする
qstat -r で実行中のジョブを確認し、job name が [sample].starseqr_[0-9] のものがあれば、以下の手順でジョブを終了させる。
```
while true; do
  PID=`ps x | grep ${sample} | grep "python /opt/STAR-SEQR-0.6.7/starseqr.py" | awk '{print $1}'`
  if [ "$PID" = "" ]; then break; fi
  kill -SIGINT $PID
  sleep 30s
done
```
※ qdelでジョブを強制終了した場合、Snakemake から見ると「未完了ジョブ（incomplete job）」扱いになります。\
Snakemake は incomplete job を検出すると、ダミーファイルを作成しても、安全のために強制再実行(forced execution)されます。\
そのため、プロセスを中断させることで「未完了ジョブ（incomplete job）」扱いさせなくします。

### 4\. ダミーファイルの作成
STAR-SEQR の結果ファイルが未作成のため、後続の convert_cff工程でエラー終了するので、ダミーファイルを作成して続行できるようにする。
```
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}.Aligned.sortedByCoord.out.bam
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}.Chimeric.out.junction
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}.Chimeric.out.sam
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}_STAR-SEQR_candidates.txt
touch ${WORKDIR}/${batch}/${sample}/Benchmark/Fusion/${sample}.starseqr.tsv
```

### 5\. 後工程の実行
snakemake実行用の環境に入る。
```
source /data1/iGeniPipe/miniconda3/bin/activate cs
```
snakemake dry run で実行されるコマンドを表示し、starseqrが実行されないことを確認する。
```
snakemake --dry-run --snakefile /data1/GxD_WTS/Pipeline/workflow/Snakefile --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch}
```
仮想環境からでる。
```
conda deactivate
```
解析の再実行を実施する。
```
sh ${WORKDIR}/${batch}/${sample}/run.sh
```

### 6\. ダミーファイルの作成
次の工程（convert_cff）は実行されますが、STAR-SEQR は Fusion不検出と同じ挙動となるため、その次の工程（merge_cff）でエラー終了します。（[case3.Fusion不検出による解析中断](#case3) と同じ挙動）\
そのため、ダミーファイルを作成して merge_cff が実行されるようにします。
```
yes NA | head -n 17 | paste -sd '\t' > ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion/${sample}.star_seqr.cff
```

### 7\. 後工程の実行
snakemake実行用の環境に入る。
```
source /data1/iGeniPipe/miniconda3/bin/activate cs
```
snakemake dry run で実行されるコマンドを表示し、merge_cff 以降が実行されることを確認する。
```
snakemake --dry-run --snakefile /data1/GxD_WTS/Pipeline/workflow/Snakefile --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch}
```
仮想環境からでる。
```
conda deactivate
```
解析の再実行を実施する。
```
sh ${WORKDIR}/${batch}/${sample}/run.sh
```
</details>

<a id="case5"></a>
## case5. STAR-SEQR 停止による解析中断
WTS Pipeline Fusion解析工程において、STAR-SEQRでbreakpointの候補が1つもないと処理が中断されるため、次のステップ(convert_cff)が実行されない。※リード数がかなり少ない場合などに起こる
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
qstatで当該検体の解析が実行中でないことを確認したのち、Logsフォルダに出力されている最新の \*.${sample}.starseqr_*.err の中身を確認し、starseqr 工程がエラー終了していることを確認する。
```
cd $WORKDIR/$batch/$sample/Logs
ll -t *.${sample}.starseqr_*.err
```

### 3\. ダミーファイルの作成
次の工程(convert_cff)でSTAR-SEQRの結果として参照するファイルを作成する。
```
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}.Aligned.sortedByCoord.out.bam
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}.Chimeric.out.junction
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}.Chimeric.out.sam
touch ${WORKDIR}/${batch}/${sample}/Fusion/STAR-SEQR/${sample}_STAR-SEQR/${sample}_STAR-SEQR_candidates.txt
touch ${WORKDIR}/${batch}/${sample}/Benchmark/Fusion/${sample}.starseqr.tsv
```

### 4\.  後工程の実行
snakemake実行用の環境に入る
```
source /data1/iGeniPipe/miniconda3/bin/activate cs
```
snakemake dry run で実行されるコマンドを確認する。convert_cff 以降が実行されることを確認する。
```
snakemake --dry-run --snakefile $SNAKEFILE --directory /data1/GxD --profile /data1/GxD_WTS/Pipeline/profiles/all.q --config patient_id=${sample} output_dir=${WORKDIR}/${batch} 
```
仮想環境からでる。
```
conda deactivate
```
解析の再実行を実施する。
```
sh ${WORKDIR}/${batch}/${sample}/run.sh
```

### 5\. ダミーファイルの作成
STAR-SEQRは不検出として扱われるため、次のステップ(merge_cff) で解析が中断される。 [case3.Fusion不検出による解析中断](#case3) を参照してcffファイルを作成する。※ STAR-Fusion、Arribaでも不検出の可能性が高いので、適宜ファイルを作成する。
```
yes NA | head -n 17 | paste -sd '\t' > ${WORKDIR}/${batch}/${sample}/Fusion/Metafusion/${sample}.star_seqr.cff
```

### 6\.  後工程の実行
解析の再実行を実施する。
```
sh ${WORKDIR}/${batch}/${sample}/run.sh
```
</details>

## case6. Fusion figure のレイアウトエラー（文字切れなど）
WTS Pipeline Fusion解析工程において、解析自体は正常終了しているが、domain名が長い、domainの数が多いなどの場合に Fusion Fugure のレイアウトが崩れることがある。
<details>
  <summary> 
    More Details
  </summary>

### 1\. 変数の設定
```
WORKDIR=/data1/data/result
batch=
sample=
SIF=/data1/GxD_WTS/Pipeline/containers/inhouse.sif
```
batch : バッチフォルダ名 \
sample : 当該検体のSample ID

### 2\.Fugure の作成を依頼する
ITチーム経由で開発チームに体裁を修正したFigureの作成を依頼する。\
Figure作成に必要な途中ファイルを送付するよう指示があるので、ITチーム経由で送付する。

### 3\. データの編集
開発チームに作成してもらった .png ファイルを以下の場所に格納する。\
同名のファイルがPipelineで作成されているので、上書きするか元のファイルを他のフォルダに避難させる。※元ファイルを同じフォルダに置かない。
```
/data1/data/result/WTS/${batch}/${sample}/Summary/Domains/
```

### 4\. レポート再作成の準備
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

### 5\. データベースへの再アップロードとレポート再作成
前工程でanalysis statusを101（解析中）にセットした場合に実行する。解析実行時のPipelineバージョンがデフォルトとは異なる場合は、**解析実行時のPipelineバージョンのmodulesのmain.pyファイルを指定する**こと。
```
singularity shell --bind /data1 $SIF python3 /data1/GxD_${test_type}/Pipeline/modules/report_json/main.py -s ${sample} -d ${WORKDIR}/${test_type}/${batch}/${sample}/Summary -o ${WORKDIR}/${test_type}/${batch}/${sample}/Summary/${sample}.report.json -r ${WORKDIR}/${test_type}/${batch}/${sample}/Summary/${sample}.report.pdf -c True -u 192.168.9.100 -p 3014 -v v1.1.0 --upload true --start_log ${WORKDIR}/${test_type}/${batch}/${sample}/QC/fastp/${sample}.fastp.start.time.log
```
</details>

## case7. 解析結果の修正とレポートの再作成（手作業）
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
