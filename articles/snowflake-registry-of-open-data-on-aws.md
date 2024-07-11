---
title: "SnowflakeでVCFファイルのバリアントフィルタリング ~Registry of Open Data on AWSの利用~"
emoji: "😎"
type: "tech"
topics: [bigdata, WholeGenomeSequence, Bioinformatics, snowflake, aws]
published: false
---

## はじめに
バイオインフォマティシャン兼データエンジニアのこれえだです。

シーケンシング技術の進歩によりゲノムデータの生成速度が急速に増加し、データの生成コストも劇的に低下しているためバイオ系データはムーアの法則でどんどん増えています。SRAデータの容量に関して言えば、2007年5月時点では47.04 GBだったものが、2024年2月では27.93 PBと62万倍に増えています。
![](/images/snowflake-registry-of-open-data-on-aws/growth_biodata.png)
*増え続けるSRAデータ 出典：https://www.ncbi.nlm.nih.gov/sra/docs/sragrowth/*

またDNA, RNA, エピゲノム, タンパク質、タンパク質や化合物の構造情報、相互作用情報などバイオデータの種類は多く容量が大きくなり、バイオデータを管理するための、ストレージコストと管理コストの増大が課題となっています。
![](/images/snowflake-registry-of-open-data-on-aws/kind-of-biodata.png)
*多岐に渡るバイオデータの種類 出典：https://www.semanticscholar.org/paper/The-European-Bioinformatics-Institute%E2%80%99s-data-2014-Brooksbank-Bergman/1c11992577c22af41ef1e861656d146fdd5d0f53*

Snowflakeのヘルスケア業界での利活用は日本国内であまり活発ではありません。しかし、Snowflakeのメリットを活かせば多くのバイオデータ解析を行うことができます。今回は次世代シーケンサから得られたWhole genome sequenceデータといったゲノムデータからバリアントコールされたVCFファイルをSnowflake内に取り込んで解析する状態にする方法を解説しようと思います。Registry of Open Data on AWSで公開されているデータはS3に存在するため、snowflakeで簡単に外部ステージ統合して利用することができます。今回はこちらのやり方を中心に解説していこうと思います。

## Registry of Open Data on AWSとは
Registry of Open Data on AWS（AWSのオープンデータレジストリ）とは、Amazon Web Services（AWS）が提供するプラットフォームで、研究者、データサイエンティスト、開発者が自由にアクセスできるオープンデータセットのカタログです。このレジストリには、さまざまな分野のオープンデータが含まれており、データを探して使用するのが簡単になります。バイオインフォマティクスに関するデータも多く存在しています。

![](/images/snowflake-registry-of-open-data-on-aws/aws_open.png)
*Registry of Open Data on AWS*

今回はこちらに紹介されている1000genome projectのDRAGENデータとNIHが用意したClinVarのデータを使って公共データの外部ステージ登録とVCFデータのアノテーション付けを行い、SQLでバリアントフィルタリングすることに挑戦していきます。

## 公共データの外部ステージ登録とVCFデータのアノテーション付け

大まかな解析の流れは以下のようになります。

1. Open Data RegistryのDRAGEN 1000-Genomesプロジェクトの8人分ゲノムデータが入ったS3を外部ステージ登録する
2. snowflakeのテーブルに取り込む
3. Panelの情報が入ったS3を外部ステージ登録し、アノテーションを実行する

![](/images/snowflake-registry-of-open-data-on-aws/dragen_clinvar.png)
*DRAGENとClinVarのデータが入ったS3を外部ステージ登録しsnowflakeで解析*

### DRAGENのS3を外部ステージ登録

はじめにDRAGENのS3のURI (s3://1000genomes-dragen-3.7.6/data/individuals/hg38-graph-based）をSnowflakeのステージに登録していきます。その後にSQLでS3に入っているデータセットをDIRECTORY関数用いたクエリで確認できます。

```sql
create or replace stage dragen_all
  directory = (enable = true)
  url = 's3://1000genomes-dragen-3.7.6/data/individuals/hg38-graph-based'
  file_format = (type = CSV compression = AUTO)
;

SELECT * FROM DIRECTORY( @dragen_all )
where relative_path rlike '/HG0011[0-7].*.hard-filtered.vcf.gz'
;
```

![](/images/snowflake-registry-of-open-data-on-aws/DRAGEN_S3.png)
*外部ステージ登録したDRAGENステージにクエリした結果*

### DRAGENをsnowflakeのテーブルに取り込みクエリをする
DRAGENを取り込むためのテーブルを定義していきます。テーブルはSnowflakeでパフォーマンス最適化されているのでSQLでデータ確認が高速で可能です。

```sql
create or replace table GENOTYPES_BY_SAMPLE (
   CHROM       varchar,
   POS         integer,
   ID          varchar,
   REF         varchar,
   ALT         array  ,
   QUAL        integer,
   FILTER      varchar,
   INFO        variant,
   SAMPLE_ID   varchar,
   VALS        variant,
   ALLELE1     varchar,
   ALLELE2     varchar,
   FILENAME    varchar
);

-- これが長い。
insert into GENOTYPES_BY_SAMPLE (
   CHROM       ,
   POS         ,
   ID          ,
   REF         ,
   ALT         ,
   QUAL        ,
   FILTER      ,
   INFO        ,
   SAMPLE_ID   ,
   VALS        ,
   ALLELE1     ,
   ALLELE2     ,
   FILENAME    )
with file_list as (
   SELECT file_url, relative_path FROM DIRECTORY( @dragen_all )
   where relative_path rlike '/HG0011[0-7].*.hard-filtered.vcf.gz'
 order by random()
)
select
   replace(CHROM, 'chr','')         , 
   POS           ,
   ID            ,
   REF           ,
   split(ALT,','),
   QUAL          ,
   FILTER        ,
   INFO          , 
   SAMPLEID      ,
   SAMPLEVALS    ,
   allele1(ref, split(ALT,','), SAMPLEVALS:GT::varchar) as ALLELE1,
   allele2(ref, split(ALT,','), SAMPLEVALS:GT::varchar) as ALLELE2,
   split_part(relative_path, '/',3)
from file_list,
    table(ingest_vcf(BUILD_SCOPED_FILE_URL(@dragen_all, relative_path), 0)) vcf
;
```
*ingest_vcf関数は以前紹介した[こちら](https://zenn.dev/t_koreeda/articles/b4c2519e4824c9)の記事のUDFを使っています。

サンプルごとのバリアントコールのサンプリング深度（DP）値の分布を要約し、この重要な品質指標の5、10、50、90、95パーセンタイル値を示しています。
![](/images/snowflake-registry-of-open-data-on-aws/DRAGEN_insert.png)
*DRAGENを取り込んだテーブルへのクエリ*

### ClinVarの入ったS3を外部ステージ登録
アノテーションにはClinVarデータを使用していきます。こちらもRegistry of Open Data on AWSによって管理がされています。

```sql
create or replace stage clinvar
   url = 's3://aws-roda-hcls-datalake/clinvar_summary_variants'
   file_format = (type = PARQUET)
;

create or replace table CLINVAR (
 chrom                  varchar(2),
 pos                    number    ,
 ref                    varchar   ,
 alt                    varchar   ,
 ALLELEID               number    ,
 chromosomeaccession    varchar   ,        
 CLNSIG                 varchar   ,
 clinsigsimple          number    ,  
 cytogenetic            varchar   ,
 geneid                 number    ,
 genesymbol             varchar   ,
 guidelines             varchar   ,
 hgnc_id                varchar   ,
 lastevaluated          varchar   ,  
 name                   varchar   ,   
 numbersubmitters       number    ,     
 origin                 varchar   ,
 originsimple           varchar   , 
 otherids               varchar   ,
 CLNDISB                array     ,
 CLNDN                  array     ,
 rcvaccession           array     , 
 CLNREVSTAT             varchar   ,
 RS                     number    ,
 startpos               number    ,
 stoppos                number    ,
 submittercategories    number    ,        
 testedingtr            varchar   ,
 type                   varchar   ,
 variationid            number    ,
 full_annotation        variant  
);


insert into CLINVAR
select
   $1:chromosome                 ::varchar(2)  chrom,
   $1:positionvcf                ::number      pos,
   $1:referenceallelevcf         ::varchar     ref,
   $1:alternateallelevcf         ::varchar     alt,
   $1:alleleid                   ::number      ALLELEID,
   $1:chromosomeaccession        ::varchar     chromosomeaccession,
   $1:clinicalsignificance       ::varchar     CLNSIG,
   $1:clinsigsimple              ::number      clinsigsimple,
   $1:cytogenetic                ::varchar     cytogenetic,
   $1:geneid                     ::number      geneid,
   $1:genesymbol                 ::varchar     genesymbol,
   $1:guidelines                 ::varchar     guidelines,
   $1:hgnc_id                    ::varchar     hgnc_id,
   $1:lastevaluated              ::varchar     lastevaluated,
   $1:name                       ::varchar     name,   
   $1:numbersubmitters           ::number      numbersubmitters,
   $1:origin                     ::varchar     origin,
   $1:originsimple               ::varchar     originsimple,
   $1:otherids                   ::varchar     otherids,
   split($1:phenotypeids::varchar,  '|')       CLNDISB,
   split($1:phenotypelist::varchar, '|')       CLNDN,
   split($1:rcvaccession::varchar,  '|')       rcvaccession,
   $1:reviewstatus               ::varchar     CLNREVSTAT,
   $1:rsid_dbsnp                 ::number      RS,
   $1:start                      ::number      startpos,
   $1:stop                       ::number      stoppos,
   $1:submittercategories        ::number      submittercategories,
   $1:testedingtr                ::varchar     testedingtr,
   $1:type                       ::varchar     type,
   $1:variationid                ::number      variationid,
   $1                            ::variant     full_annotation
FROM '@clinvar/variant_summary/'
where $1:assembly = 'GRCh38'
order by chrom, pos
;
```
### ClinVarでのバリアントフィルタリング
ClinVarでのバリアントフィルタリングを実際にやってみましょう。遺伝性大腸がんに関連するすべての位置を調べます。染色体や位置を明示的に指定する必要はなく、ClinVarへの結合によって暗黙的にフィルタリングが適用されることに注意してください。

```sql
select   g.chrom, g.pos, g.ref, allele1, allele2, count (sample_id)
from genotypes_by_sample g
join clinvar c
   on c.chrom = g.chrom and c.pos = g.pos and c.ref = g.ref
where
   array_contains('Hereditary nonpolyposis colorectal neoplasms'::variant, CLNDN)
group by 1,2,3,4,5
order by 1,2,5
;
```

ClinVarのCLNDNカラムを扱う場合、カラムに複数の値が含まれる可能性があるため、フィルター条件でSnowflakeのARRAY_CONTAINS関数を使用して目的の指標を指定します。次に、バリアントコールのALLELE1またはALLEL2がアノテーションと一致するバリアント値のみを明らかにすることで、クエリを絞り込むことができます。

```sql
select   g.chrom, g.pos, g.ref, c.alt clinvar_alt, genesymbol, allele1, allele2, count (sample_id)
from genotypes_by_sample g
join clinvar c
   on c.chrom = g.chrom and c.pos = g.pos and c.ref = g.ref
   and ((Allele1= c.alt) or (Allele2= c.alt))
where
   array_contains('Hereditary nonpolyposis colorectal neoplasms'::variant, CLNDN)
group by 1,2,3,4,5,6,7
order by 1,2,5
;
```

## SnowflakeでVCFファイルのバリアント解析をするメリット

### フィルタリングワークフローの管理
バリアント解析は、CSVやVCFなどの大量の中間ファイルが生成されることが一般的です。これらのファイルは、データの変換、フィルタリング、解析の各ステップで生成され、それぞれ異なる形式や構造を持つことが多いです。Snowflakeを使用すると、これらの中間ファイルをすべてテーブルとして保存し、一元管理することができます。テーブル形式で保存することで、データの可視化、検索、フィルタリング、分析が容易になり、データ管理が効率化されます。また、Snowflakeのスキーマに基づいてデータを整理できるため、データの整合性と品質を保つことができます。

### SQLによる簡便なデータフィルタリング
Snowflakeを使用すると、SQLを利用してデータフィルタリングを行うことができます。SQLは、データベース管理システムで広く使われている標準言語であり、データの選択、フィルタリング、集計、結合など、さまざまな操作が簡単に行えます。Snpsiftなどの専用ツールと比較すると、SQLは以下の点で優れています。

- 柔軟性：複雑なクエリを記述して多様なフィルタリング条件を指定できます。
- 再利用性：クエリを保存しておけば、何度でも同じ操作を再実行することができます。
- 統合性：異なるデータソースからのデータを一度に扱い、統合した形でフィルタリングできます。
- スケーラビリティ：大量データに対しても高いパフォーマンスでフィルタリング処理を実行できます。

### Snowflake上でのストレージ圧縮メリットを受けれる
Snowflakeは、構造化データだけでなく、VCFファイルを含む非構造化データもサポートしています。Snowflakeのデータストレージは、以下のようなメリットがあります。

- 自動圧縮：データを取り込む際に自動的に圧縮が行われ、ストレージ容量が節約されます。これにより、コストの削減が期待できます。
- 高効率のデータアクセス：圧縮されたデータは、必要に応じて迅速に解凍されるため、データアクセスの速度が向上します。
- データ管理の簡便性：非構造化データを一元管理でき、必要なデータを迅速に検索、取得することが容易になります。
- セキュリティとコンプライアンス：Snowflakeのセキュリティ機能により、データの安全性が確保され、コンプライアンス要件も満たすことができます。

## 終わりに（おまけ：バイオデータとLLMの話）
公共データとSnowflakeの外部ステージ登録は非常に相性が良く、うまく活用すればSnowflakeのストレージコストを抑えつつ、データの種類を増やすことができます。冒頭で述べた通り、バイオデータは今後ますます増加していくことが予想されます。また、昨今では大規模言語モデル（LLM）による開発も盛んになっています。医療データのPDFなど非構造化データを扱うことができれば、生物医学用語を理解するLLMモデルの構築も可能になります。SnowflakeのCortex LLMを活用したライフサイエンスデータの取り扱いは、今後ますます興味深いテーマとなるでしょう。バイオ系には公開データが豊富にあり、それを使ってLLMを構築することができます。Snowflakeを活用して、バイオデータを効果的に管理していきましょう。

