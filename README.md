# Understanding Angular Segment Analysis (ASA)

**Walkability研究で使用したASAの計算原理の理解**

FOSS4G 2026のAcademic Trackで、高齢者のWalkability（歩きやすさ）に関する研究を発表しました。
質疑応答で、研究に利用したAngular Segment Analysis（ASA）の計算原理を十分に説明できなかったことが、この学習の出発点です。
本リポジトリでは、[発表用スライド](https://docs.google.com/presentation/d/1Gt-tbDPdKop2Z40APwutNTq_KNTRYnrbnygELLge3MY/edit)を主な基礎資料とし、論文とdepthmapX公式資料を参照しながら、道路ネットワークからNAIN・NACHを求める考え方を整理します。
研究成果の再発表ではなく、研究で使った分析手法を自分の言葉で説明するための技術学習記録です。

## 1. 研究背景：FOSS4G 2026でのWalkability研究

元の研究では、新百合ヶ丘駅周辺と流山おおたかの森駅周辺を対象に、高齢者の歩行環境を比較しました。
丘陵地と比較的平坦な地域では、街路のつながり方だけでなく、坂道による身体的な負担も異なります。
そこで、街路ネットワークの構造と道路勾配の両面から、歩きやすさを検討しました。

| 観点 | 研究で扱った内容 |
|---|---|
| 対象 | 高齢者の歩きやすさ |
| 地域 | 新百合ヶ丘駅周辺／流山おおたかの森駅周辺 |
| 地形の比較 | 丘陵地／比較的平坦な地域 |
| ネットワーク分析 | Space SyntaxのASAを用いたNAIN・NACHの算出 |
| 勾配との関係 | 別途算出した道路勾配をネットワーク指標に重ね合わせて評価 |

**勾配はASAの角度コストや経路探索アルゴリズムに直接組み込んだものではありません。**
ASAによるネットワーク分析と道路勾配の計算を別々に行い、その結果を重ね合わせています。
そのため、NAIN・NACHの値そのものを、高齢者にとっての総合的な歩きやすさと同一視することはできません。

元論文の正式な書誌情報は、[出版社の論文ページ](https://isprs-archives.copernicus.org/articles/L-4-W1-2026/311/2026/)で確認できます。

- 著者・年：Hinako Terado（2026）
- 論文名：*How Does Walkability for Elderly People Differ between Hilly and Flat Urban Areas? A Case Study of Shin-Yurigaoka Station and Nagareyama-Otakanomori Station*
- 掲載誌：*The International Archives of the Photogrammetry, Remote Sensing and Spatial Information Sciences*, L-4/W1-2026, 311–318
- DOI：[10.5194/isprs-archives-L-4-W1-2026-311-2026](https://doi.org/10.5194/isprs-archives-L-4-W1-2026-311-2026)

Walkability研究の概要・分析方法・結果・発表資料・研究過程は、[研究ポートフォリオ（usakooon/research-portfolio）](https://github.com/usakooon/research-portfolio)にまとめています。

このREADMEでは、地域ごとの研究結果の詳細は元論文に譲り、分析に用いたASAの計算原理に焦点を当てます。

## 2. Angular Segment Analysis（ASA）とは？

ASAは、Space Syntaxにおける街路ネットワークの分析方法です。
道路をセグメント（線分）として表現し、隣り合うセグメントへ移るときの**方向転換角度**を移動コストとして扱います。
そのコストの合計が最小となる「最小角度経路」を探索し、Integration（到達性）とChoice（通過性）を評価します。
ここで最小化するのは角度コストであり、道のりや所要時間ではありません。

従来のAxial Analysisは、軸線（axial line）同士の接続と、その間を移るステップ数に着目します。
ASAでは、軸線を交差点で分割したセグメントや道路中心線のセグメントを用い、曲がる角度を区別します。
例えば、直進して次のセグメントへ移る場合には、角度コストは増えません。
このような軸線表現と道路中心線表現の関係は、[Turner（2007）](https://doi.org/10.1068/b32067)で論じられています。

## 3. ASAの計算プロセス

スライドの説明順序に沿って、5つのステップで整理します。

**道路のグラフ化 → Angular Cost → 最小角度経路 → Integration・Choice → NAIN・NACH**

### Step 1：道路ネットワークのグラフ化

セグメントマップは道路の位置・形状を示す地理的な表現です。
一方、ここで用いる計算用グラフでは、**道路セグメントをノード、セグメント間の接続をエッジ**として表します。
つまり、交差点そのものをノードとする一般的な道路グラフとは対応関係が異なります。

| セグメントマップ | 計算用グラフでの対応 |
|---|---|
| 道路セグメント | ノード |
| 交差点などで移動可能なセグメント同士の接続 | エッジ |
| 接続部での方向転換角度 | エッジに付ける角度コスト |

以下のA・B・C・Dは、それぞれ1本の道路セグメントを表します。
説明用に接続とコストを無向グラフへ単純化しており、depthmapX内部の方向を区別する処理まで表したものではありません。

### Step 2：方向転換角度の計算（Angular Cost）

[スライド5枚目](https://docs.google.com/presentation/d/1Gt-tbDPdKop2Z40APwutNTq_KNTRYnrbnygELLge3MY/edit#slide=id.h38a4a76f45796f64_0_17)と同じ順序・記号で説明します。

**1. 道路の座標からベクトルを計算**

交差点に向かう進入方向を $\vec{a}$、交差点から出る退出方向を $\vec{b}$ とします。
同一の平面座標系上で、各ベクトルは「進行先の座標 − 進行元の座標」から求めます。

**2. 角度を求める**

$$
\theta=\cos^{-1}\left(\frac{\vec{a}\cdot\vec{b}}{\lvert\vec{a}\rvert\lvert\vec{b}\rvert}\right)
$$

ベクトルの内積と、三平方の定理・平方根で求めたベクトルの長さから $\cos\theta$ を求め、逆三角関数で角度 $\theta$ を求めます。

**3. 角度をAngular Costに変換**

$$
\mathrm{Angular\ Cost}=\frac{\theta}{90^\circ}
$$

スライドの図のように、交差点へ右向きに進入し、上向きに退出する場合は、θ = 90°なのでAngular Costは1です。

> ベクトルは両方とも進行方向にそろえます。両方を交差点から外へ向けると、そのなす角と方向転換角度は異なります。直進なら0°、直角に曲がれば90°となる向きで計算します。

[Turner（2007, Section 3.1）](https://discovery.ucl.ac.uk/id/eprint/2092/1/2092.pdf)に基づく対応は次のとおりです。

| Direction Change（方向転換角度） | Angular Cost |
|---|---:|
| 0° | 0 |
| 45° | 0.5 |
| 90° | 1.0 |
| 135° | 1.5 |
| 180° | 2.0 |

左右どちらへ曲がる場合も、方向転換の大きさを非負のコストとして扱います。

### Step 3：最小角度経路の探索

スライドと同じ4セグメントの仮想ネットワークを使います。
図の線はセグメント間の接続を示し、線の傾きや長さは実際の道路形状・角度を表しません。

~~~mermaid
graph LR
    A((A)) ---|0.5| B((B))
    B ---|0.5| C((C))
    C ---|1.0| D((D))
    A ---|1.5| D
~~~

AからCへの2つの経路を比較します。

| 経路 | Angular Costの合計 |
|---|---:|
| A → B → C | 0.5 + 0.5 = **1.0** |
| A → D → C | 1.5 + 1.0 = **2.5** |

合計が小さい **A → B → C** が最小角度経路です。
同様の探索を各起点・終点について行い、次の指標を集計します。

### Step 4：IntegrationとChoice

**Integration（到達性）** は、あるセグメントから他のセグメントへ、角度コストの面でどれだけ到達しやすいかを捉えます。
その基礎となるAngular Total Depth（ATD）は、起点から各目的地までの最小角度コストの合計です。

以下の例では、起点以外の3セグメントすべてを集計します。

| 起点＼目的地 | A | B | C | D | ATD |
|---|---:|---:|---:|---:|---:|
| A | — | 0.5 | 1.0 | 1.5 | **3.0** |
| B | 0.5 | — | 0.5 | 1.5 | **2.5** |
| C | 1.0 | 0.5 | — | 1.0 | **2.5** |
| D | 1.5 | 1.5 | 1.0 | — | **4.0** |

例えばAのATDは、0.5 + 1.0 + 1.5 = 3.0です。
**同じ集計対象数で比較する場合**、ATDが小さいほど到達性は高くなります。
ATDそのものがIntegrationの値なのではなく、到達性の指標を求めるための基礎量です。

**Choice（通過性）** は、各起点・終点の最小角度経路の「途中」に、そのセグメントがどれだけ含まれるかを捉えます。
この例では、起点と終点を通過回数に含めず、順序を区別しない6組を1回ずつ数えます。

| 起点–終点 | 最小角度経路 | 途中に含まれるセグメント |
|---|---|---|
| A–B | A → B | なし |
| A–C | A → B → C | B |
| A–D | A → D | なし |
| B–C | B → C | なし |
| B–D | B → C → D | C |
| C–D | C → D | なし |

| Segment | Choice（簡略集計） |
|---|---:|
| A | 0 |
| B | 1 |
| C | 1 |
| D | 0 |

> このChoice表は、スライドに沿って経路の通過回数を数えた学習用の簡略例です。実際のdepthmapXにおける方向性、セグメント長による重み付け、同点経路の処理などを完全に再現した計算結果ではありません。

### Step 5：NAIN・NACHの正規化

ネットワークの規模などによる影響を抑え、異なる街路ネットワークの指標を比較しやすくするために正規化します。
本READMEでは、[depthmapX公式資料「Calculate NACh and NAIn」](https://github.com/SpaceGroupUCL/depthmapX/blob/master/docs/howdoi.md)の次の式を採用します。

**NAIN：Normalised Angular Integration（正規化された到達性）**

$$
\mathrm{NAIN}=\frac{NC^{1.2}}{ATD}
$$

**NACH：Normalised Angular Choice（正規化された通過性）**

$$
\mathrm{NACH}=\frac{\log(ACH+1)}{\log(ATD+3)}
$$

| 記号 | 意味 |
|---|---|
| NC | Node Count：解析対象として集計されるノード（セグメント）の数 |
| ATD | Angular Total Depth：最小角度コストの合計 |
| ACH | Angular Choice：最小角度経路に基づく通過性の集計値 |

実データでは、同じ解析条件で出力されたNode Count・Total Depth・Choiceを使います。
上の簡略Choice表をdepthmapXの出力と同一視して、実解析のNACHを再現したとは扱いません。
また、正規化後も、解析範囲・半径・ネットワークの作成方法などの条件をそろえて比較する必要があります。

> **理論的背景と採用式の区別：** 正規化の理論的背景は[Hillier et al.（2012）](https://discovery.ucl.ac.uk/id/eprint/1389938/)を参照します。ただし、同論文のAppendix 2に掲載されたNAIN関連の数式と、今回採用したdepthmapX公式GitHubの式には表記上の差があるため、同一の式として扱いません。ここで使用する計算式の出典は、上記のdepthmapX公式資料です。

## 4. 今回の学習から理解したこと

- ASAは、道路の接続関係に加えて、セグメント間の方向転換角度を考慮する分析方法です。
- IntegrationとChoiceは、ともに最小角度経路を基礎としますが、前者は他の場所への到達性、後者は経路の途中としての通過性を評価します。
- 正規化したNAINは到達性、NACHは通過性を捉える指標です。
- Walkability研究で両指標を使う理由は、街路構造をこの2つの観点から捉えるためです。計算原理を理解することで、指標の意味と、別途評価する勾配との役割の違いを説明しやすくなりました。

本学習は、ASAの基本的な計算の流れを理解・説明するための整理であり、depthmapXのアルゴリズムを完全に実装・再現したものではありません。

## 5. 参考文献・関連リンク

### 参考文献

- Hillier, B., Yang, T., & Turner, A. (2012). Normalising least angle choice in Depthmap—and how it opens up new perspectives on the global and local analysis of city space. *Journal of Space Syntax, 3*(2), 155–193. [UCL Discovery](https://discovery.ucl.ac.uk/id/eprint/1389938/)
- SpaceGroupUCL. (n.d.). *Calculate NACh and NAIn* [GitHub repository documentation]. GitHub. [公式ドキュメント](https://github.com/SpaceGroupUCL/depthmapX/blob/master/docs/howdoi.md)
- Terado, H. (2026). How does walkability for elderly people differ between hilly and flat urban areas? A case study of Shin-Yurigaoka Station and Nagareyama-Otakanomori Station. *The International Archives of the Photogrammetry, Remote Sensing and Spatial Information Sciences, L-4/W1-2026*, 311–318. [https://doi.org/10.5194/isprs-archives-L-4-W1-2026-311-2026](https://doi.org/10.5194/isprs-archives-L-4-W1-2026-311-2026)
- Turner, A. (2007). From axial to road-centre lines: A new representation for space syntax and a new model of route choice for transport network analysis. *Environment and Planning B: Planning and Design, 34*(3), 539–555. [https://doi.org/10.1068/b32067](https://doi.org/10.1068/b32067)

### 関連リンク

- [Walkability研究ポートフォリオ：研究概要・分析方法・結果・研究過程](https://github.com/usakooon/research-portfolio)
- [発表用スライド：Angular Segment Analysisの理解へ](https://docs.google.com/presentation/d/1Gt-tbDPdKop2Z40APwutNTq_KNTRYnrbnygELLge3MY/edit)
- [FOSS4G 2026で発表した元論文（出版社ページ）](https://isprs-archives.copernicus.org/articles/L-4-W1-2026/311/2026/)
- [depthmapX公式GitHub](https://github.com/SpaceGroupUCL/depthmapX)
- [Turner（2007）の公開本文（UCL Discovery）](https://discovery.ucl.ac.uk/id/eprint/2092/)
