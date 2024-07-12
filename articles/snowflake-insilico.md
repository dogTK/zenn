---
title: "StreamlitとSnowflake Notebooksを用いたIn silico創薬"
emoji: "💊"
type: "tech"
topics: [lifescience, insilico, Bioinformatics, snowflake, drug]
published: false
---

## はじめに
バイオインフォマティシャン兼データエンジニアのこれえだです。

近年、高度なコンピュータ技術とアルゴリズムの発展により、大規模なデータセットの解析が可能になりました。これにより、化合物の特性や相互作用を迅速に予測することができます。In Silico創薬とは、コンピュータを用いたシミュレーションや解析によって新しい薬の開発を行う手法です。従来の創薬プロセスは非常に高コストで時間がかかります。In Silico創薬では、初期段階での高額な実験を減らし、コンピュータシミュレーションによって有望な候補を絞り込むことができます。この手法は、従来の実験ベースの方法と比較して効率的かつ経済的であるため、近年大きな注目を集めています。

SnowflakeでもSnowflakeのハイコンピューティングリソースを利用してIn silico創薬を行うことが可能です。今回は、StreamlitとSnowflake Notebooksを用いたIn silico創薬の一部を解説しようと思います。

## StreamlitとSnowflake Notebooksについて
まず今回のIn silico創薬に使うStreamlitとSnowflake Notebooksについて解説します。どちらもSnowflakeが持つネイティブな機能です。それぞれを解説していきます。

### Streamlitの特徴
StreamlitはPythonのオープンソースライブラリで、データをインタラクティブなWebアプリケーションとして素早く簡単に可視化することができます。これはHTML、CSS、JavascriptなどのWebアプリケーション開発に必要な知識がなくても構築可能です。SnowflakeではSnowflake上でStreamlitアプリを構築、展開、共有できるStreamlit in Snowflakeという機能も持っています。
![](/images/snowflake-insilico/snowflake_in_streamlit.png)
*Streamlit in Snowflake*

### Snowflake Notebooksの特徴
Snowflake NotebooksはPython および SQL 用のインタラクティブなセルベースのプログラミング環境を提供するSnowsightの開発インターフェイスになります。Streamlit などの他のライブラリを使用して、データをインタラクティブに視覚化することが可能です。Snowflakeにすでに存在するデータを探索したり、ローカル ファイル、外部クラウド ストレージ などからSnowflake にアップロードすることが可能です。
![](/images/snowflake-insilico/snowflake_notebooks.png)
*Snowflake Notebooks*

## StreamlitとSnowflake Notebooksを用いたバーチャルスクリーニング
それでは実際にsnowflakeでIn silico創薬をしていきたいと思います。今回は化合物の類似度を用いたバーチャルスクリーニングを紹介します。特許や論文で薬の候補となる化合物が発表された場合、より有望な類似化合物を探したいことがあります。ここではインフルエンザ治療薬であるノイラミニダーゼ阻害薬「ラニナミビル」に類似した化合物を、ZINC DBのデータからスクリーニングし、タニモト係数で類似度がTop10のものだけをピックアップしてきます。

![](/images/snowflake-insilico/zinc_virtual.png)
*ZINC DBのデータからバーチャルスクリーニング*

ZINC (ZINC Is Not Commercial) Databaseは、無料で利用できる化合物データベースで、主にバーチャルスクリーニングや薬剤発見の研究に使用されます。このデータベースは、特に計算化学やバイオインフォマティクスの分野で広く利用されています。ZINC DBには、何百万もの商業的に入手可能な化合物の3D構造が含まれており、ドラッグライクな性質を持つものや、リード化合物として有望なものが多く含まれています。また、化合物データは、PDB、SDF、MOL2、SMILESなどの多様なフォーマットでダウンロード可能です。

## In silico創薬「化合物類似度の評価」
まずは、化合物類似度の評価を評価するサンプルコードを紹介します。下記例では、Cc1ccccc1：トルエン　Clc1ccccc1：トリクロロベンゼン の類似度をタニモト係数を使って計測する例です。
```python
import streamlit as st
from rdkit import Chem, DataStructs
from rdkit.Chem import AllChem, Draw
from rdkit.Chem.Draw import IPythonConsole
from PIL import Image
import io

mol1 = Chem.MolFromSmiles("Cc1ccccc1")
mol2 = Chem.MolFromSmiles("Clc1ccccc1")

img1 = Draw.MolToImage(mol1)
img2 = Draw.MolToImage(mol2)

st.image(img1, caption="Molecule 1")
st.image(img2, caption="Molecule 2")

DataStructs.TanimotoSimilarity(fp1, fp2)
```
メインで使うライブラリとしてRDKitを使っています。RDKitは、オープンソースの化学情報学ライブラリで、分子の操作、シミュレーション、分析、描画などを化学情報学、バイオインフォマティクス、薬剤開発などの分野で使われています。その他モジュールは以下の通りです。

**Chem**: 分子の作成や操作を行うモジュール。
**DataStructs**: 分子間の類似度計算を行うためのモジュール。
**AllChem**: 構造描画や計算に関する高度な機能を提供。
**Draw**: 分子の描画を行うモジュール。
**IPythonConsole**: IPythonでの分子描画をサポートするモジュール。

化合物の分子構造をSMILES表記から生成していきます。Chem.MolFromSmiles 関数を使用して、SMILES表記から分子オブジェクトを作成します。次に、分子のフィンガープリントを計算します。ここではMorganフィンガープリント（ECFP）を使用します。生成した分子構造の画像を描画し、Streamlitを使って表示しています。最後にタニモト係数でトルエンとトリクロロベンゼンの類似度の評価を行っています。結果としては「0.5384615384615384」でした。

## In silico創薬「バーチャルスクリーニング」
それでは実際にバーチャルスクリーニングに入っていきます。コードをまずは全文一気に紹介します。

```python
# cell 7
spl = Chem.rdmolfiles.SmilesMolSupplier("EAED.smi")
len(spl)

# cell 8
laninamivir = Chem.MolFromSmiles("CO[C@H]([C@H](O)[C@H](O)CO[C@H]1OC(=C[C@H](NC(=N)N)[C@H]1NC(=O)C)C(=O)O")
laninamivir_fp = AllChem.GetMorganFingerprint(laninamivir, 2)

def calc_laninamivir_similarity(mol):
    fp = AllChem.GetMorganFingerprint(mol, 2)
    sim = DataStructs.TanimotoSimilarity(laninamivir_fp, fp)
    return sim

# cell 9
similar_mols = []
for mol in spl:
    sim = calc_laninamivir_similarity(mol)
    if sim > 0.2:
        similar_mols.append((mol, sim))

# cell 10
similar_mols.sort(key=lambda x: x[1], reverse=True)
mols = [x[0] for x in similar_mols[:10]]

# cell 11
def visualize_mols(mols, grid_size=(5, 2)):
    st.header("Visualized Molecules")

    img_per_row, img_per_col = grid_size
    mol_imgs = [Draw.MolToImage(mol) for mol in mols]

    width, height = mol_imgs[0].size
    canvas_width = width * img_per_row
    canvas_height = height * img_per_col

    canvas = Image.new('RGB', (canvas_width, canvas_height))

    for i, img in enumerate(mol_imgs):
        row = i // img_per_row
        col = i % img_per_row
        x = col * width
        y = row * height
        canvas.paste(img, (x, y))

    buf = io.BytesIO()
    canvas.save(buf, format='PNG')
    st.image(buf.getvalue(), use_column_width=True)

visualize_mols(mols)
```


![](/images/snowflake-insilico/notebooks_virtual_screening.gif)
*実行風景*

## SnowflakeでIn silico創薬をするメリット
SnowflakeでIn silico創薬のバーチャルスクリーニングを試してみたことで、色々とメリットがあることに気づいたのでまとめてみようと思います。

Snowflakeは、クラウドベースのデータプラットフォームとして、データウェアハウス、データレイク、データエンジニアリング、データサイエンスなどの機能を統合的に提供しています。以下に、Snowflakeが提供する機械学習フレームワークやデータ管理のメリットについて詳細に解説します。

1. 機械学習フレームワークの利用
Snowpark MLなどデータの移動や複雑なインフラの設定、ローカルでの環境構築が不要で迅速かつ効率的な機械学習の実行が可能
統合された環境:

Snowpark MLなどのフレームワークを利用することで、データの移動やインフラの設定が不要となります。これにより、データサイエンティストやアナリストは、ローカル環境を構築せずにSnowflake上で直接機械学習モデルを構築・訓練・評価することができます。
利点: 時間とリソースの節約、データ移動に伴うセキュリティリスクの低減。
スケーラブルな計算リソース:

Snowflakeのクラウドインフラは、自動的にスケーリングするため、大規模なデータセットや複雑な機械学習モデルの訓練にも対応可能です。ユーザーは必要に応じてリソースを拡張でき、計算能力を最大限に活用できます。
利点: 大規模データ処理の迅速化、高パフォーマンスの実現。
2. 大量の化合物データセットを効率よく格納
創薬プロセスにおいて必要となる大量の化合物データや生物学的データを一元管理が可能。めぼしい化合物を類縁体スクリーニングに素早くかける事ができる
中央集権的データ管理:

Snowflakeは、化合物データや生物学的データを一元的に管理するための堅牢なプラットフォームを提供します。これにより、データの可視性が向上し、研究者は必要なデータに迅速にアクセスできます。
利点: データの整合性確保、データアクセスの迅速化。
高速なデータ検索とフィルタリング:

Snowflakeの強力なクエリエンジンを使用して、膨大な化合物データセットを高速に検索およびフィルタリングすることができます。これにより、研究者は有望な化合物を迅速に特定し、次のステップに進むことができます。
利点: スクリーニングプロセスの高速化、研究サイクルの短縮。
3. ハイコンピューティングリソースが要求される解析まで一貫して解析が可能
SPCSを利用すれば、MDやドッキングシミュレーションといったハイコンピューティングリソースが要求される解析までエンドツーエンドで解析することが可能
エンドツーエンドの解析ワークフロー:

Snowflakeは、SPCS（Snowflake Powered Compute Service）を活用することで、分子動力学（MD）シミュレーションやドッキングシミュレーションなど、高度な計算リソースを必要とする解析を実行できます。これにより、初期のデータ収集から最終的なシミュレーションまでのすべてのステップを一貫して行うことができます。
利点: 一貫したワークフローの確立、解析精度の向上。
高い計算能力とスケーラビリティ:

Snowflakeのクラウドインフラは、高い計算能力とスケーラビリティを提供します。これにより、研究者は計算負荷の高いシミュレーションを効率的に実行できます。
利点: 大規模シミュレーションの実行、計算リソースの効率的な利用。

## 終わりに
いかがだったでしょうか。