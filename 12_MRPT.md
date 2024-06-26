# 多参照摂動理論

多参照摂動理論（multi(-)reference perturbation theory; MRPT）の計算をしてみます。
どのような計算かというと、MP2に似ています。
MP2はHartree--Fock法、すなわち一つのスレーター行列式で記述した波動関数を参照して摂動理論を適用した計算を行いました（単参照摂動理論）。
一方MRPTでは、MCSCF法（実際の実装としてはCASSCF法であることが多い）に対して摂動理論を適用した計算を行います。
おそらく、複数のスレーター行列式（または配置状態関数）を参照して摂動計算を行うため、多参照摂動理論と呼ばれています。
CASSCFの段階で静的電子相関を取り入れ、摂動理論により動的電子相関を取り入れるというのが方針で、
どちらの電子相関もバランス良く取り入れられるため、精度の高い計算ができると言われています。
[単参照摂動理論](07_SR_corr.md)で出てきた配置間相互作用や結合クラスターを多配置波動関数に応用することもできます（MRCIやMRCCと略される）が、計算コストは非常に高いです。

詳細までは説明しませんが、MRPTには様々な種類があります。
摂動論は基本的にゼロ次ハミルトニアンの定義は任意（もちろん、計算のしやすさを考慮したり、摂動が小さくなるように選びますが）なので、
ゼロ次ハミルトニアンの定義の方法により、様々なMRPTが定義できます。
一番有名なのは、CASPT2 (complete active space second-order perturbation theory)という手法でしょう。
他にもNEVPT2 (*N*-electron valence state)やMCQDPT2 (multi-configuration quasi-degenerate)という手法もよく知られています。
ちなみに、これらのMRPTで一つのスレーター行列式に対しての計算を行うと、MP2の結果に一致します。

PySCFではSC-NEVPT2という手法を扱うことができます。
これは[NEVPT2](https://en.wikipedia.org/wiki/N-electron_valence_state_perturbation_theory)の一種で、
strongly contracted (SC)と呼ばれる内部縮約の手法を用いて計算コストを下げています。
本来はそれぞれの電子配置に対して二電子励起を考えるような所ですが、
内部縮約を行い縮約した関数を定義して活性空間内の軌道の数に対応する次元にコストを下げています。
PySCFの実装は単状態NEVPT2のみで、擬縮退NEVPT2 (QD-NEVPT2)ではないと思います。
このため他の状態が近接する場面では、若干おかしな事になる場合があります（例えばLiFの10 Angstromあたり）。

## SC-NEVPT2の計算

入力ファイルの書き方は一意ではないですが、とりあえずこんな感じで計算してみました。
NEVPT2の場合は余計なパラメータが必要ないため、入力ファイルの作成自体はMP2と同様に簡単です。
ですが、MRPTの中ではSC-NEVPT2の実装は割と簡単とは言え、それでも実際に実装するとなるとかなり面倒です。
```python
  1 from pyscf import gto, scf, mcscf, mrpt
  2 
  3 mol = gto.M(verbose=4,
  4             atom= [["C",(0.00000000, 0.00000000,-1.24468680)],
  5                    ["C",(0.00000000, 0.00000000, 1.24468680)],
  6                    ["H",(0.00000000, 1.72797970,-2.31603770)],
  7                    ["H",(0.00000000,-1.72797970,-2.31603770)],
  8                    ["H",(0.00000000, 1.72797970, 2.31603770)],
  9                    ["H",(0.00000000,-1.72797970, 2.31603770)]],
 10             basis="6-31G(d)",unit="Bohr")
 11 mf = scf.RHF(mol)
 12 mf.kernel()
 13 
 14 mc = mcscf.CASSCF(mf,2,2)
 15 mc.kernel()[0]
 16 
 17 nev = mrpt.NEVPT(mc)
 18 nev.kernel()
```
`import mrpt`を加え、さらに最後に二行を加えただけです。
まず11・12行目でHartree--Fockの計算を行い、14・15行目で行うCASSCF計算の準備をしています。
この段階で、スレーター行列式一つに対応する（最適な）分子軌道係数を得ることができます。
そして、14・15行目でCASSCF計算を行い、今度は多配置波動関数に対応する分子軌道係数とCI係数を得ています。
このCASSCFの結果を用いて、17・18行目でSC-NEVPT2の計算を行っています。

出力ファイルとして、以下の行が追加されるでしょう。
```
143 Sr    (-1)',   E = -0.00000000028849
144 Si    (+1)',   E = -0.00000000000000
145 Sijrs (0)  ,   E = -0.14241075396093
146 Sijr  (+1) ,   E = -0.00688327014528
147 Srsi  (-1) ,   E = -0.04457371967076
148 Srs   (-2) ,   E = -0.00425348385977
149 Sij   (+2) ,   E = -0.00097363926188
150 Sir   (0)' ,   E = -0.03304801594663
151 Nevpt2 Energy = -0.232142883133741
```
`Sr(-1)'`等は、励起の仕方を意味しています。
MP2の場合は占有軌道と非占有軌道の二種類しかなく、二電子を励起することができる唯一の可能性は、二電子とも占有軌道から非占有軌道に励起するのみです。
しかし、MRPTはCASSCFを参照としているため、占有軌道・非占有軌道だけではなく非整数値の電子により占有される小数占有軌道（活性空間のこと）が存在し、3つのグループができます。
このため、二電子の励起の方法に、上記出力ファイルの通り8つの組み合わせが出てきます。
例えば`Sr(-1)'`は、active->activeの一電子励起＋active->virtualの一電子励起という意味です（Sの添え字が非活性空間の励起の方法を示し、かっこ内の数字が活性空間内の電子の増減を示す）。
他の励起クラスに興味があれば[原論文](https://doi.org/10.1063/1.1515317)を参照してください。
右の列が、摂動計算によるエネルギーの補正です（単位は原子単位）。
それで、8つの励起クラスの補正を足すと、
151行目の`Nevpt2 Energy = -0.232142883133741`という補正（単位は原子単位）が得られます。
なお、ここでは補正であるため、正しい（？）NEVPT2のエネルギーは、
CASSCFと補正の和を計算し、-78.291501748206641 hartreeとなります。

***

SA-CASSCFに対するSC-NEVPT2計算は、
[前回](11_MCSCF.md)取り扱ったbenzeneのSA2-CASSCF(6e,6o)/6-31G(d)（一重項のみ取り扱った計算）に対して、
行ってみます。
たぶん、次のような感じでできるかと思います。
```python
 20 cas_list = [17,20,21,22,23,30]
 21 mc = mcscf.state_average_(mcscf.CASSCF(mf,6,6),[0.5,0.5,0.0,0.0,0.0])
 22 mc.fcisolver.spin = 0
 23 mc.fix_spin_(ss=0)
 24 mo = mcscf.sort_mo(mc, mf.mo_coeff, cas_list)
 25 mc.kernel(mo)[0]
 26 mo = mc.mo_coeff
 27 
 28 mc = mcscf.CASCI(mf, 6, 6)
 29 mc.fcisolver.nroots = 5
 30 mc.fcisolver.spin = 0
 31 mc.fix_spin_(ss=0)
 32 mc.kernel(mo)
 33 
 34 mrpt.NEVPT(mc,root=0).kernel()
 35 mrpt.NEVPT(mc,root=1).kernel()
```
出力ファイルをまとめます。
|状態| CASSCFエネルギー | SC-NEVPT2の補正 | SC-NEVPT2エネルギー |
|-|-|-|-|
|S<sub>0</sub> | -230.773332 | -0.696364 | -231.469697 |
|S<sub>1</sub> | -230.590910 | -0.681672 | -231.272587 |

これによると、SC-NEVPT2による励起エネルギーは、実験よりかなり過大評価することがわかります。
必ずしもMRPTが良い結果を与えるわけではないです･･･。
NEVPT2は特にπ-π*励起を過大評価しやすかったりします。

## 演習問題

- 分子軌道をcore, active, virtualの三つに分けて二電子励起を考えると、9種類（または10種類）の励起の組み合わせができそうですが、
CASを参照としたSC-NEVPT2では上記の通り8つの励起クラスのみ考えます。
考えていない1つの励起クラスはどのようなものでしょう。
そして、なぜ考えなくても良いのでしょうか。
- CASSCFやSC-NEVPT2を用いて、基底状態N<sub>2</sub>のポテンシャルエネルギーカーブを計算してみましょう。
[単参照電子相関理論](07_SR_corr.md)のところでも計算したと思いますので、特に原子間距離が長い領域に着目して単参照電子相関理論と結果を比較してみましょう。
- 2-methylpyrimidineのn-π*垂直励起エネルギーは、CASSCFとSC-NEVPT2の結果を比較すると何が言えるでしょうか。実験値は3.78 eV（L. Alvarez-Valtierra et al. J. Phys. Chem. A 111, 12802 (2007).)らしいですが、これは実は0-0遷移エネルギーなので垂直励起エネルギーとは少し違います。
- Cytosineのπ-π\*垂直励起エネルギーは実験的に4.6 eVあたりであることが知られています（M. P. Fülscher et al. J. Am. Chem. Soc. 117, 2089-2095 (1995).のTable 4）。できるだけ再現してみてください。計算方法は指定しませんが、π-π\*励起になっているかは確認しましょう。
- 余裕があれば、別のソフトウェア（BAGEL, OpenMolcas, ORCAとか）を使ってCASPT2の計算を行ってみましょう。
