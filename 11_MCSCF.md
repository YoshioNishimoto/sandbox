# 多配置SCF計算

[多配置（multi(-)configuration(al)）SCF（MCSCF）計算](https://pyscf.org/user/mcscf.html)をやります。
これまでの手法は、単一のスレーター行列式を使ってのSCF計算、
あるいはその結果を用いた電子相関やプロパティ計算を取り扱ってきました。
MCSCFでは、複数のスレーター行列式を用います。
プログラムによってはスレーター行列式ではなく配置状態関数（configuration state function; CSF）を用いる場合もありますが、特に区別せずスレーター行列式とします。
PySCFはスレーター行列式を用いています。

MCSCFはいろいろ難しくなります。
理論的な面では、波動関数を定義するパラメータが増えます。
単一のスレーター行列式での波動関数パラメータは、（通常）分子軌道係数のみでした。
MCSCFでは、複数のスレーター行列式を足し合わせる際にCI (configuration interaction)計算を行うため、線形結合を取る係数（CI係数）が追加で必要になります。
最適化するパラメータが増えるため、SCFの収束が難しくなりますし、実装（プログラム）も複雑になります。

具体的な計算の面では、入力ファイルの作成が面倒になります。
どのようなスレーター行列式を含めるかを指定する必要があります。
あまり多くのスレーター行列式を計算に含めると、計算時間・メモリが多くなってしまいます。
しかも、計算に含めるスレーター行列式があまり正しくない（下記の「活性空間」の取り方が良くない）と、
SCF計算が収束しなかったりします。

複数のスレーター行列式を含める計算を一般的にMCSCFと呼びますが、
最近はその特殊な形である[complete active space SCF (CASSCF)](https://en.wikipedia.org/wiki/Multi-configurational_self-consistent_field#Complete_Active_Space_SCF)という手法が用いられることが多いです。
この手法は、[活性空間（active space）](https://ja.wikipedia.org/wiki/%E5%AE%8C%E5%85%A8%E6%B4%BB%E6%80%A7%E7%A9%BA%E9%96%93)と呼ばれる空間をユーザーが指定すると、
プログラムが勝手にその空間でのfull CIに対応するスレーター行列式を生成し、
その行列式全てを用いてMCSCF計算を行うというものです。
単一のスレーター行列式を扱う手法の場合は、分子軌道を二電子の占有軌道と非占有軌道の二つのスペースに分ければ良いです。
一方で複数のスレーター行列式を扱う手法の場合は、二電子の占有軌道と非占有軌道、そして占有数を変化させる（full CI計算を行う）活性空間の三つのスペースに分けます。

活性空間は、その空間に含まれる電子と分子軌道の数を指定します。
例えば、8電子6軌道のCASSCF計算は、CASSCF(8e,6o)やCASSCF(8,6)と書かれたりします（"SCF"を省略する場合もあります）。
ここでは、CASSCFのみを取り扱います。
いちいち用いるスレーター行列式を自分で指定するのではなく、電子や軌道の数を指定するだけで活性空間を定義して計算を行うことができます。
部分空間とは言えfull CI計算をするので、電子と分子軌道の数が増えると計算コストが急激に増加します。
目安としては、(12e,12o)以上は結構時間がかかります。
これを解決する方法としてrestricted active space (RAS) SCFや[DMRG](https://pyscf.org/user/dmrgscf.html)というのがありますがここでは使いません。

では「どう活性空間を選べば良いか」ですが、よくわかりません。分子により異なります。
「多配置性が出そうな分子軌道」や「計算したい物性に関係する分子軌道」を選ぶというのが答えになると思います。
例えば芳香族系ならπ軌道とπ*軌道、第一遷移金属系ならd軌道を含めるという雰囲気です。
これまで用いてきた手法は特に手法や対象とする分子についての知識はさほど必要なく、ブラックボックス的に取り扱うことができました。
しかしMCSCFの計算は、上記の理由からブラックボックス的に取り扱うことはできず、注意深く取り扱う必要があります。

## CASSCF計算

またethyleneを使って計算してみます。
```python
from pyscf import gto, scf, mcscf

mol = gto.M(verbose=4,
            atom= [["C",(0.00000000, 0.00000000,-1.24468680)],
                   ["C",(0.00000000, 0.00000000, 1.24468680)],
                   ["H",(0.00000000, 1.72797970,-2.31603770)],
                   ["H",(0.00000000,-1.72797970,-2.31603770)],
                   ["H",(0.00000000, 1.72797970, 2.31603770)],
                   ["H",(0.00000000,-1.72797970, 2.31603770)]],
            basis="6-31G(d)",unit="Bohr")
mf = scf.RHF(mol)
mf.kernel()

mc = mcscf.CASSCF(mf,2,2)
mc.kernel()[0]
```
一度Hartree--Fockの計算（`mf = scf.RHF(mol)`）をして分子軌道係数を得て、その後でCASSCFの計算（`mc = mcscf.CASSCF(mf.2,2)`）をします。
このように、Hartree--Fock計算をして、その分子軌道係数を用いてCASSCFの計算をするのは割と一般的な手続きです。
局在化軌道（非占有軌道を回転させてCASSCFのinitial guessにするという、謎の手法も存在する）を用いる場合もあります。
ここでのCASSCF計算は`mf = mcscf.CASSCF(mf,2,2)`としており、計算の詳細を渡す際に活性空間も一緒に定義しています（最後の`2,2`の部分）。
(2e,2o)と定義するかに見えますが、どうやら(2o,2e)（2つめの引数が軌道の数、3つめの引数が電子の数）と定義しているようです。

出力としては以下のような感じになります。
```
 89 ******** <class 'pyscf.mcscf.mc1step.CASSCF'> ********
 90 CAS (1e+1e, 2o), ncore = 7, nvir = 27
...
123 nroots = 1
124 pspace_size = 400
125 spin = None
126 CASCI E = -78.0505587988761  S^2 = 0.0000000
127 Set conv_tol_grad to 0.000316228
128 macro iter 1 (21 JK  4 micro), CASSCF E = -78.059327647349  dE = -0.0087688485  S^2 = 0.0000000
129                |grad[o]|=0.034  |grad[c]|= 0.008535979996907405  |ddm|=0.0334
130 macro iter 2 (7 JK  3 micro), CASSCF E = -78.0593588650729  dE = -3.1217724e-05  S^2 = 0.0000000
131                |grad[o]|=0.00305  |grad[c]|= 3.446796483031416e-05  |ddm|=0.00248
132 macro iter 3 (1 JK  1 micro), CASSCF E = -78.0593588650729  dE = -4.2632564e-14  S^2 = 0.0000000
133                |grad[o]|=1.55e-05  |grad[c]|= 5.098473772638283e-08  |ddm|=8.6e-08
134 1-step CASSCF converged in 3 macro (29 JK 8 micro) steps
135 CASSCF canonicalization
136 Density matrix diagonal elements [1.91500509 0.08499491]
137 CASSCF energy = -78.0593588650729
138 CASCI E = -78.0593588650729  E(CI) = -1.23783621090438  S^2 = 0.0000000
```
まずは9行目に、空間の詳細が書いてあります。
`CAS (1e+1e,2o)`は、もちろんCASSCF(2e,2o)と指定していることを意味しています。
`1e+1e`は、おそらくα軌道とβ軌道の電子数を表示していると考えられます。
`ncore`はスレーター行列式に関わらず常に二電子占有されている軌道の数です。
`nvir`は常に電子が占有されない軌道の数です。
正しく定義されているか、確認してください。
126行目の`CASCI E = -78.0505587988761  S^2 = 0.0000000`は、
Hartree--Fockの結果から活性空間でfull CIを行った結果です。
ここでは、まだ分子軌道係数がHartree--Fock法で得た値のままです。

そこから128行目から133行目でCASSCFの繰り返し計算を行い、
最終的な結果は137行目の`CASSCF energy = -78.0593588650729`となります。
Hartree--Fockのエネルギーより低下していることが分かると思います。
136行目の`Density matrix diagonal elements [1.91500509 0.08499491]`は、軌道の占有数です。
Hartree--Fockの占有数だと2.0 (HOMO)と0.0 (LUMO)しかありませんが、励起した電子配置を考えることにより
だいたい元LUMO(?)に0.085電子が励起される電子構造が得られることになるという雰囲気です。
MCSCFで出てくる軌道は主にnatural molecular orbital （自然分子軌道）と呼ばれるもので、一電子電子密度を対角化するようにして得ることができます。
このようにすると、上記のように軌道の占有数が小数で表される電子状態を表現することができます。
一方、Hartree--Fockで出てくるような、フォック行列を対角化するような分子軌道は、canonical molecular orbital （カノニカル分子軌道）と呼ばれています。

CASSCFに似た言葉としてCASCI（138行目）という言葉も登場します。
これは、与えられた分子軌道係数を用いてCAS空間でfull CIを行う（CI係数の最適化）ことに対応します。
一方CASSCFは、分子軌道係数の最適化とCI係数の最適化を平行（交互に行うか同時に行うかはアルゴリズムによる。と思う）して行っています。
CASSCFでは分子軌道係数の最適化も行うため、CASCIよりもCASSCFのエネルギーが低くなります。
CASSCFで収束させた分子軌道を用いてCASCI計算を行った場合は、二つのエネルギーが一致します。

## 活性空間の指定

では、次はbenzeneの計算をしてみます。
次の構造で計算を使ってやってみます。
```python
from pyscf import gto, scf, mcscf

mol = gto.M(verbose=4,
            atom= [["C",(-0.013893521, 0.000000000,-1.393796327)],
                   ["C",(-1.169962214,-0.354690359,-0.695632327)],
                   ["C",( 1.142175172, 0.354690359,-0.695632327)],
                   ["H",(-2.069387320,-0.630640582,-1.238805327)],
                   ["H",( 2.041600279, 0.630640582,-1.238805327)],
                   ["C",(-1.169962214,-0.354690359, 0.700697673)],
                   ["C",( 1.142175172, 0.354690359, 0.700697673)],
                   ["H",(-2.069387320,-0.630640582, 1.243870673)],
                   ["H",( 2.041600279, 0.630640582, 1.243870673)],
                   ["C",(-0.013893521, 0.000000000, 1.398861673)],
                   ["H",(-0.013893521, 0.000000000, 2.485208673)],
                   ["H",(-0.013893521, 0.000000000,-2.480143327)]],
            basis="6-31G(d)")
mf = scf.RHF(mol)
mf.kernel()
```
active spaceはどう定義すれば良いでしょう。
正しい「電子と分子軌道の数」を用いて計算すると、おそらく`CASSCF energy = -230.76211819929`を得られると思われます。
が、さらに正しい「活性空間」を定義して計算すると、おそらく`CASSCF energy = -230.775273783315`になると思われます。
CASSCFは変分的な方法なので、計算条件が同じであれば、よりエネルギーが低くなる『正しい「活性空間」』を用いた計算が正しい（それが着目したい物理量にふさわしいかは別）です。
活性空間の定義には電子と分子軌道の数が必要と述べましたが、実はどの軌道を活性空間に入れるかを追加で考慮する必要があります。

どの軌道を活性空間に入れるかは、以下のようにして指定します。
aa, bb, ..., ffには、軌道の番号を入れましょう。
```python
cas_list = [aa,bb,cc,dd,ee,ff]
mc = mcscf.CASSCF(mf,6,6)
mo = mcscf.sort_mo(mc, mf.mo_coeff, cas_list)
mc.kernel(mo)[0]
```
`cas_list`の中には、活性空間に入れる分子軌道のインデックス（これは1から始まっています）を入力します。
benzeneではCASSCF(6e,6o)をするべきなので、6つの分子軌道（3つはHartree--Fockで占有軌道（=6電子）、3つは非占有軌道になります）を指定します。
`aa,bb,cc,dd,ee,ff`には何が入るでしょうか。
このために、[分子軌道の可視化](05_MO_visualization.md)が必要となります。
教科書で出てくるようなπ軌道・π*軌道を見つけて、それらの数字を入れて計算を行い、正しいエネルギーが得られるか確認しましょう。

## 状態平均CASSCF

上の計算は基底状態のみを取り扱う単状態（single-stateまたはstate-specific）でのCASSCF（SS-CASSCF）というものです。
このセクションでは状態平均（state-averaged） CASSCF (SA-CASSCF)をやります。
どういうときにSA-CASSCFが必要かと言われると難しいですが、励起状態を考える場合には通常SA-CASSCFにするものかと思います。
というのも、SS-CASSCFで励起状態を計算すると、root-flippingという問題が起きSCFが収束しづらくなります。
また、励起エネルギーを計算する場合には基底状態のみならず励起状態でも波動関数パラメータを最適化するべきなので、SA-CASSCFを選択するべきかと思われます。
S<sub>1</sub>まで考える場合は、多くの場合二状態の平均を取ります（が、他の励起状態やPESの扱い方に応じて変化します）。

とりあえず簡単に計算するためには、上記の例から二行目を変更します。
```python
cas_list = [aa,bb,cc,dd,ee,ff]
mc = mcscf.state_average_(mcscf.CASSCF(mf,6,6),[0.5,0.5])
mo = mcscf.sort_mo(mc, mf.mo_coeff, cas_list)
mc.kernel(mo)[0]
```
二行目が変わっていると思います。
これで、一つ目の状態と二つ目の状態を1:1の割合で平均（`[0.5,0.5]`）を取って計算することになります。
出力ファイルを見ると以下のような感じになるかと思います。
```
CASSCF energy = -230.703195285598
CASCI E = -230.703195285598  E(CI) = -6.39080007759441  S^2 = 1.0000000
CASCI state-averaged energy = -230.703195285598
CASCI energy for each state
  State 0 weight 0.5  E = -230.773810504445 S^2 = 0.0000000
  State 1 weight 0.5  E = -230.63258006675 S^2 = 2.0000000
```
最終的なエネルギーは`CASSCF energy = -230.703195285598`（二つのエネルギーのちょうど平均）です。
で、平均された二つの状態は最後の二行に示されているとおりです。
ですが、二つ目の状態のスピンが`S^2 = 2.0000000`となっているとおり、
純粋な一重項ではなくなっていることが分かります。

純粋な一重項とするためには、以下の通り3行目・4行目を加えれば良いらしいです。
```python
cas_list = [aa,bb,cc,dd,ee,ff]
mc = mcscf.state_average_(mcscf.CASSCF(mf,6,6),[0.5,0.5])
mc.fcisolver.spin = 0 
mc.fix_spin_(ss=0)
mo = mcscf.sort_mo(mc, mf.mo_coeff, cas_list)
mc.kernel(mo)[0]
```
ですが、こちらはGAMESS-USの結果と合わないです･･･。
状態平均の指定を`[0.5,0.5,0.0,0.0,0.0]`とすれば、GAMESS-USの結果と合います。
状態平均に必要な状態しか計算しないと、必要なベクトルを加える前にDavidson iterationが収束してしまうからだと思われます。

## 演習問題

- [単参照での励起状態計算](10_excited.md)と同様に、SA-CASSCF(2e,2o)を用いてethyleneの吸収・蛍光の波長を計算してみましょう。
CASSCFでも構造最適化が可能とは思いますが、新しいPySCFでどのように動くかよく分からないので、前回TD-B3LYPを用いて計算したと思われる構造を用いて良いと思います。CASSCFの構造最適化でも良い。
- [分子軌道の可視化](05_MO_visualization.md)で取り扱った、1,3-butadiene (C<sub>4</sub>H<sub>6</sub>)と1,3,5-hexatriene (C<sub>6</sub>H<sub>8</sub>)でも同様に垂直励起エネルギーの計算を行ってみましょう。CASSCFの計算後に出てくる、natural orbital（とoccupation number）も示してください。
ただし、当然異なる活性空間を定義する必要があるため、きちんと軌道を見ながら定義しましょう。
- 上でSA-CASSCFの計算を行うとスピン多重度が1となるような計算をしておきながら`S^2 = 2.0000000`となった状態は、どのようなスレーター行列式により記述できるでしょうか。二電子二軌道モデルで教えてください。
- CASSCFのsize-extensivity（[単参照電子相関理論](07_SR_corr.md)の演習問題を参照）を検証してみましょう。CASSCFはsize-extensiveな手法です。計算条件は自分で設定してください。ただし、多配置性がほとんどでない系はやめましょう（活性空間の定義も難しくなる）。一般的には、HOMO--LUMOギャップが大きな分子は多配置性が出にくいです。
- 2-methylpyrimidineの第一励起はn-π*励起になった記憶があります。
どのように活性空間を設定すると、この状態を正しく記述できそうでしょうか。
活性空間に入れるべき分子軌道（できればCASSCF計算が収束した後のnatural orbital）を示しましょう。
