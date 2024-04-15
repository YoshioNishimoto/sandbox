# 導入

ここでは、[PySCF](https://github.com/pyscf/pyscf) (Python-based Simulations of Chemistry Framework)という量子化学計算パッケージについて、量子化学計算を始めたばかりでも理解できるような内容で説明していきたいと思います。
よりPySCFについて勉強したい方は、PySCFについての[総説（review）](https://onlinelibrary.wiley.com/doi/abs/10.1002/wcms.1340)をご参照ください。
と言っても、どちらかというと理論開発プラットフォームとしての面に着目しているため、必要な前提知識が多く、やや難易度が高いかもしれません。

## PySCFの特徴（特長）

PySCFはPythonというプログラム言語で書かれています。
他の多くの量子化学計算プログラム（有名なプログラムで言えば、GaussianやGAMESSなど）がFortranやC(++)言語をベースにしています。
これらのプログラムでは、入力ファイル（input file）を作成する際に、プログラムにより異なる書き方を勉強する必要があります。
また、自分でプログラムを変更したい・新しい機能を付け加えたいというときには、プログラム言語を勉強し、かなり気合いを入れて既存のコードの使い方を勉強し、プログラムを書き換え、コンパイル・リンクをする（そして大抵はデバッグをする）必要があります。
新しい量子化学計算プログラムを使う際には、この二点の障壁を乗り越える必要がありますが、年をとるにつれて障壁を乗り越えるのに必要な時間が長くなっていきます。

PySCFは、それらの障壁をある程度下げていると言って良いと思います。
まず、入力ファイルの作成に際し、新しく覚えることが少ないです（ただし、Pythonを習得していれば）。
また、新しい機能を付け加えるにあたっても、「かなり気合いを入れて既存のコードの使い方を勉強」する必要が少なくなり、コンパイル・リンクをする必要もありません。
さらにPythonのインストールさえしてしまえばPySCFのインストール自体は割と簡単にできる（外部ライブラリをあまり使用していないため）ため、「軽く量子化学計算を使ってみたい」あるいは「軽く理論開発をしてみたい」というのに良いかと思います。

細かいことですが、Pythonのようなインタープリタ方式（コンパイルが必要ない）言語の実行速度は大抵遅いのですが、
PySCFでは時間がかかる部分をFortranやC(++)言語（コンパイル方式と言います）を用いて高速化しています。

## PySCFで何ができるか

PySCF･･･というより、まずは量子化学計算プログラム全般で何ができるかについて、簡単にご説明します。

まずは、分子（あるいは結晶）の「エネルギー」を計算することができます。エネルギーを計算することができると、原子を手で少し動かしてエネルギーを計算して･･･という操作を繰り返すことにより、ポテンシャルエネルギーカーブや[ポテンシャルエネルギー面](https://ja.wikipedia.org/wiki/%E3%83%9D%E3%83%86%E3%83%B3%E3%82%B7%E3%83%A3%E3%83%AB%E3%82%A8%E3%83%8D%E3%83%AB%E3%82%AE%E3%83%BC%E6%9B%B2%E9%9D%A2) （potential energy surface; PES）を描くことができます。
コレが分かると、（時間はかかりますが）分子がどのような形になっていると安定（エネルギーが低い状態。そして、その配座をとる確率が高い状態）になるかが分かるようになります。

エネルギーだけ計算できても、あまり嬉しくありません。
もちろん高精度計算の場合は、エネルギー計算だけでも価値があると考えています（個人の意見です）。
先ほど「分子がどのような形になっていると安定」と書きましたが、PESを描くことなく分子の形を計算できないでしょうか。
これは、「構造最適化」によって一度の計算ジョブで計算することができます。
安定な構造はPESでの極小値に相当するため、PESでの勾配（の反対方向）に沿って原子を動かしていけば、極小値を見つけることができそうです（これが局所極小値か大域的極小値かは別の話ですが）。

他にはプロパティの計算でしょうか。
例えば、分子軌道係数を利用して分子軌道を描くことも可能です。
最高占有軌道（highest occupied molecular orbital; HOMO）や最低非占有軌道（lowest unoccupied molecular orbital; LUMO）を図示して、どのように反応しそうかということも言えなくはないと思います。
また、結合次数（例えばMayer's bond order analysis）を計算することも可能ですし、双極子モーメント（dipole moment）なども計算できます。
他にも励起エネルギー、二次微分を計算することで基準振動解析、より高次の微分を計算することで分極率を計算すると言ったことも可能です。

まずはこの「エネルギー」計算と「構造最適化」計算が、最も身近な計算になるかと思います。
上記のプロパティの計算や、溶媒効果を取り入れた計算、分子動力学シミュレーション（ただし、これはPySCFではできなさそう）も可能です。
ただし、「なんか分からんけど計算やってみたい」という使い方はできません。
何の計算をするか、目的を持つ必要があります。

さて、PySCFで何ができるかと言いますと、上で挙げた大抵のことができます。
このチュートリアルを通して、様々な手法を用いてのエネルギー計算、構造最適化、プロパティの計算、励起エネルギーの計算までできるようになれば良いと思います。

## チュートリアルでカバーすること

あくまで予定です。

- 初歩的なPySCFの使い方
  - 入力ファイルの作成
  - 出力ファイルの見方
- 様々な手法でのエネルギー計算
  - Hartree--Fock
  - density functional theory
  - second-order M\oller--Plesset perturbation theory
  - configuration interaction
  - (coupled cluster)
  - multiconfigurational self-consistent field (MCSCF)
  - complete active space SCF (CASSCF)
  - multireference perturbation theory (MRPT; SC-NEVPT2)
- 構造最適化
  - Hartree--Fock
  - density functional theory
  - second-order M\oller--Plesset perturbation theory
- プロパティの計算
  - 分子軌道のプロット
  - 結合次数
  - 基準振動解析
  - 励起エネルギー

PySCF内部を変更する、プログラミングという高度なことはしません。私がPythonを使えないので。
手法についてはごく簡単に触れるとは思いますが、詳細までは触れられないと思います。
数式の入力に際しLaTeXのコマンドを直接入れられないので面倒になりました。

## 可視化ソフトウェア

分子軌道等、PySCFの計算結果を可視化するためには、可視化ソフトウェアが必要になります。
あるいは、計算したい分子を組み立てるためにも、可視化ソフトウェアを用いる方が便利です。
[かなり多くの種類のソフトウェア](https://en.wikipedia.org/wiki/List_of_molecular_graphics_systems)があるのですが、
ここでは私が使っている・メインで使っていたことがあるソフトウェア（全て英語です）をご紹介いたします。

|ソフトウェア|cubeファイルの可視化|分子モデリング|無償|
|-|-|-|-|
|[MOLDEN](http://cheminf.cmbi.ru.nl/molden/)|〇|×|〇|
|[visual molecular dynamics (VMD)](https://www.ks.uiuc.edu/Research/vmd/)|〇|×|〇|
|[MolView](http://molview.org/)|×|〇|〇|
|[ChemCraft](https://www.chemcraftprog.com/)|〇|〇|△|
|[GaussView](https://gaussian.com/gaussview6/)|〇|〇|×|
|[Avogadro](https://avogadro.cc/)|〇|〇|〇|

とは言え、使いやすいもの・気に入ったものを自分で見つけてください。
私はChemCraft (licensed)とVMDをメインで使っています。
この「量子化学ハンズオン」では、皆さんは主に[Avogadro](https://avogadro.cc/)を用いていくことになると思います。
とりあえずこちらをインストールしておきましょう。
私は使ったことがないので分かりませんが、下記によるとAvogadroを用いて分子モデリング（コンピュータ上で分子を組み立てて、たぶんxyzファイルの生成も可能）も可能なようです。

- [分子モデリングソフト Avogadro を使ってみる](https://yutarine.blogspot.com/2008/11/avogadro.html)

基本的には、有償というだけあってGaussViewが最も高性能です。
多くの計算サーバーには既にインストールされていると思います。
ChemCraftは期限付無償とか、そんな感じだったかと思います。

MOLDENとVMDは、たぶん分子を組み立てることができないです。
MOLDENは軽量なのが特長です。
VMDは割ときれいな構造を書くことができます。

ただ、残念ながらPySCFの出力ファイルを直接読み込んでくれるソフトウェアはないと思います。
とりあえず分子軌道を可視化する際は、PySCFからmoldenファイル（やcubeファイル）を出力し、それを可視化ソフトウェアで読み込むことになります。
これは[分子軌道の可視化](05_MO_visualization.md)で説明します。

## Pythonのインストール

早速PySCFのインストールをしたいところですが、PySCFを使うためには[Python](https://www.python.jp/)が使えなければなりません。Pythonを使うためにはいくつか方法があると思いますが、主に三つのうちどちらかでしょう。

- 手元のパソコンに直接Pythonをインストールする
- 手元のパソコンにLinux（など）をインストールし、その仮想OS上でPythonを使う
- リモートの計算機でPythonを使う

ここでは上二つのうち、どちらかになると思います。ですが、ここではPySCFを使うことが目的なので、Pythonを使うことができるだけでは不十分となります。Linux上で環境を構築する方が拡張性や汎用性の高いため、こちらの方が良いと思います。私はMacintoshユーザーではないので分かりませんが、Macintoshの場合は特に何もしなくてもPythonが使えるのでしょうか。

Windowsの場合は、Windows Subsystem for Linux (WSL)という機能を用いると、簡単にLinuxをインストールすることができます。自分で検索するか、例えば適当に検索して出てきた以下のようなページを参考にして、インストールしてみてください。

- [Windows 10 用 Windows Subsystem for Linux のインストール ガイド](https://docs.microsoft.com/ja-jp/windows/wsl/install-win10)
- [Windows 10でLinuxを使う](https://qiita.com/whim0321/items/093fd3bb2dd287a72fba)
- [WSLでWindows 10にLinux仮想環境を構築](https://www.pc-koubou.jp/magazine/21475)

## PySCFのインストール

基本的にはPySCF公式の[Installation](https://pyscf.org/install.html) のページを参照していただきたいところですが、英語のページを読み進めるのは厳しい場合もあるかと思いますので、適当に検索してヒットした日本語のページをご紹介しておきます。なお、前節でLinuxをインストールした場合は、PySCFもLinux上にインストールしましょう。

- [PySCFを使った水素分子H2の計算例](https://ma.issp.u-tokyo.ac.jp/app-post/1783)
- [PySCFのインストール方法 (Windows)](http://kagakukome.blog.jp/archives/5569169.html)

また、私は使ったことありませんが、Google Colaboratoryとやらで動かすこともできるようです（[PySCFのインストール方法(Google Colaboratory)](http://kagakukome.blog.jp/archives/5845583.html)）。

```
# メモ
sudo apt-get install python3-pip
which pip
pip install pyscf
```

インストールが正しくできているかは、次回の[01 一点計算1](01_sp_input.md)を参照して実際に計算を行い、計算値が正しく得られるかを確認しましょう。
環境によっては`python`コマンドではなく`python3`コマンドを用いる必要があるので注意してください。
