# 単参照電子相関理論

大仰なタイトルですが･･･。
ここではまとめて三つの単参照電子相関理論（single-reference electron correlation theory）を扱います。

* [配置間相互作用法](https://ja.wikipedia.org/wiki/%E9%85%8D%E7%BD%AE%E9%96%93%E7%9B%B8%E4%BA%92%E4%BD%9C%E7%94%A8%E6%B3%95)（CI; configuration interaction）
* [結合クラスター法](https://ja.wikipedia.org/wiki/%E7%B5%90%E5%90%88%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E6%B3%95)（CC; coupled cluster）
* 摂動理論（PT; perturbation theory, 特に[MP2](https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%A9%E3%83%BC%EF%BC%9D%E3%83%97%E3%83%AC%E3%82%BB%E3%83%83%E3%83%88%E6%B3%95)）

ここでは計算の方法や特徴を示すだけです。
それぞれの理論の詳細はSzabo先生の教科書のそれぞれChapter 4,5,6やHelgaker先生の鈍器（Molecular Electronic-Structure Theory）などに譲るとします。

「単参照」は「一つのスレーター行列式で記述した」ということを示しています[^1]。
一つのスレーター行列式というのは、ここではHartree--Fock法の事に対応します（Szabo: Chapter 3あたりが詳しいと思います）。
Hartree--Fock法の計算結果（すなわち分子軌道係数）を元にして、電子相関理論の計算（電子を励起させた電子配置を考える）を行います。
このため、総称としてpost Hartree--Fock法と呼ばれたりもします。

「電子相関」は、電子が多数存在することにより生じる量子的な効果です。
Hartree--Fock法では電子間の相互作用を平均場近似により考える理論です。
一方で、正確な波動関数は電子の相互作用を多体的に取り扱うべきですが、それが難しいため平均場近似を用いています。
その多体的な効果を取り入れるために電子相関を計算し、より精度の高いエネルギー・物性値などを得ることができるようになります。
Hartree--Fock法でも分子・電子の99%を記述できると言われていますが、
残りの1%が化学精度（1 kcal/mol?）を達成するのに必要です。
例えば分散項がその例で、非経験的に取り入れるためには電子相関の考慮が必要になります。

Per-Olov Löwdinの定義によると、電子相関エネルギーは正確な（すなわちfull CI）エネルギーとHartree--Fock法のエネルギーとの差であると考えられています[^2]。
full CIは電子相関エネルギーを完全に取り入れることができるため、
full CIの計算が常にできれば計算化学の研究の大部分はそこで終わりです。
それができないため様々な電子相関理論が提案されてきました。

電子相関は、形式的に「動的電子相関（dynamic electron correlation）」と「静的電子相関（static or non-dynamic electron correlation）」に分けて考えます。
ここで取り扱う単参照電子相関理論は、前者の動的電子相関をある程度正しく取り入れることができますが、
後者の静的電子相関はほとんど考慮することができません。
静的電子相関を取り扱うためには、多配置（multi-configuration(al)）のself-consistent field法（[MCSCF](11_MCSCF.md)）や、
さらに高度な多参照電子相関理論（multi-reference electron correlation theory; [MRPT](12_MRPT.md)）を用います。
原理的には、full CIに次いで多参照電子相関理論が最も精度が高くなるでしょう（実用的かは別の話）。
最初に「形式的に」と書いたのは、実際には電子相関を分けることができないためです。
例えば単参照のCIやCCでも、励起電子数を上げていくとfull CIに到達します。
full CIは電子相関を完全に取り入れることができる[^3]ので、当然動的・静的電子相関の両方を完全に取り入れることができます。
動的電子相関を取り入れられる手法の極限では、静的電子相関も入ってくるわけです。

[4回目](04_dft.md)に取り扱った密度汎関数理論（DFT; density functional theory）は、全く別の方法で電子相関を取り扱います。
計算の際に「交換相関汎関数」を指定した（例えばB3LYP）と思いますが、アレにより電子相関を取り扱っています。
前述の電子相関理論（波動関数理論）は励起電子数を上げることで系統的にfull CIの解に近づくことができますが、
DFTではそのような系統的なアプローチは発見されていません。
だからといってDFTの精度が良くないというわけではありません。
多くの場合、低い計算コストでかなり実験値に近い計算結果を返してくれます。

***

PySCFを用いた単参照電子相関理論の計算自体は、特に難しいことはありません。
少しのキーワード・行を追加するだけで計算を行ってくれます。
ここでは以下の計算レベルで計算を行ってみます。

* CI: CISD, full CI
* CC: CCSD, CCSD(T)
* PT: MP2

計算のインプットを書くのは簡単ですが、理論・プログラムは難しいです。
とりあえず使ってみてください、という感じです。
また、インプットの書き方も一通りではないので、自分のやりやすい方法を見つけてください。

計算手続きに関してもう少し説明すると、電子相関の計算は一般的に二つのステップを踏みます。
まずはHartree--Fock計算を行い一電子波動関数（用いる電子配置は既に決まっているので、分子軌道と言っても良い）を得ます。
そして、その波動関数を用いて電子相関の計算（励起した電子配置を考える）を行います。
Hartree--Fock法による波動関数は上の定義では電子相関がゼロと考えるため、この波動関数を用いて電子相関計算を行うことで、電子相関を二重に計算してしまう可能性をなくします。
このため、普通は最初のHartree--Fock計算はDFT計算であってはいけません。
電子相関の一部を二重に計算してしまう可能性があるためです。

## 配置間相互作用法の計算

ここではCISDとfull CIの計算を行ってみます。
CISDの"S"と"D"は、それぞれsingle excitationとdouble excitationを意味しています。
一電子励起・二電子励起と、励起する電子の数を示しているわけです。
なので、三電子励起まで含める場合にはCISDTになるでしょう（PySCFではたぶん行えない計算です）。
full CIは、考え得る全ての電子配置を考えます。
ですが問題は、電子数・分子軌道の数が大きくなるにつれて急速に計算コストが上昇していくことです。
full CIが取り扱えるのは、現実的（手元のパソコンでできるような計算）には1~3原子で中ぐらいの基底関数程度でしょう。

truncated CI（full CI以外のCI。truncatedは励起電子配置の打ち切りを意味しています）の良くない特徴としては、[size-extensivityとstrict separability（とsize-consistency）](https://ja.wikipedia.org/wiki/%E5%A4%A7%E3%81%8D%E3%81%95%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6%E3%81%AE%E7%84%A1%E7%9F%9B%E7%9B%BE%E6%80%A7)が満たされない点です（詳しくはSzabo: Chapter 4）。
このため（か分かりませんが）、単参照の計算ではCCやPTの方が見る機会が多いと思います（個人の感想です）。
下で用いるfull CI, CCSD, CCSD(T), MP2は、どちらの性質も満たす手法です（正しい？）。

まずはCISDの計算から行います。
入力ファイルの作成はいろいろ方法がありますが、
とりあえず[マニュアル](https://pyscf.org/user/ci.html)のとおりの計算をしてみます。
```python
  1 import pyscf
  2 mol = pyscf.M(atom = 'H 0 0 0; F 0 0 1.1',basis = 'ccpvdz',verbose=4)
  3 mf = mol.HF().run()
  4 mycc = mf.CISD().run()
```
`verbose=4`を加えました。
3行目でHartree--Fock法による計算をして、4行目でCISDの計算を行っています。
Hartree--Fock結果は置いておいて、CISDの計算は以下の出力で行っています。
```
 74 ******** <class 'pyscf.ci.cisd.RCISD'> ********
 75 CISD nocc = 5, nmo = 19
 76 max_cycle = 50
 77 direct = 0
 78 conv_tol = 1e-09
 79 max_cycle = 50
 80 max_space = 12
 81 lindep = 0
 82 nroots = 1
 83 max_memory 4000 MB (current use 71 MB)
 84 Init t2, MP2 energy = -0.211367462762819
 85 RCISD converged
 86 E(RCISD) = -100.1961829757622  E_corr = -0.2087855354134396
```
一番大事なのは、最後の行です（単位は原子単位）。
`E(RCISD)`は、(restricted) CISDのエネルギーを指しています。
結果の報告はこの数字を用いれば良いです。
一方`E_corr`はCISDにおける電子相関エネルギーです。
Hartree--Fockのエネルギーと`E_corr`を足すと、`E(RCISD)`になるはずです。

ちなみに、CI計算は励起状態のエネルギーも直接計算することができます。
現在は82行目で`nroots=1`となっており基底状態しか計算できませんが、
例えば入力ファイルで最後の行を`mycc = mf.CISD().run(nroots=3)`とすると、三つ目の状態(S<sub>2</sub>)のエネルギーまで計算してくれます。
```
RCISD root 0  E = -100.1961829757787
RCISD root 1  E = -99.78621716209804
RCISD root 2  E = -99.78621716209804
```
S<sub>1</sub>とS<sub>2</sub>（それぞれ第一励起状態と第二励起状態）が縮重していて、どちらも基底状態（`root 0`が基底状態に対応）より11.16 eV高いエネルギーということが分かりました。

で、同じ計算条件（cc-pVDZの基底関数を用いて）でfull CIの計算を行おうと思っても、たぶんそのままではできません。
たかが電子10個、分子軌道19個の計算なのに、メモリがたくさん必要です。
なので、6-31Gを使って行います（電子10個、分子軌道11個）。
二原子分子でもfull CIの計算は難しいです。

とりあえず次のように作成してみました。
```python
  1 from pyscf import gto, scf, ci, fci
  2 
  3 mol = gto.M(atom = 'H 0 0 0; F 0 0 1.1',basis = '6-31G',verbose=4)
  4 mf = scf.HF(mol)
  5 mf.kernel()
  6 
  7 mycisd = ci.CISD(mf)
  8 mycisd.kernel()
  9 
 10 myfci = fci.FCI(mol, mf.mo_coeff)
 11 print('E(FCI) = %.12f' % myfci.kernel()[0])
```
7・8行目で上と同じCISD計算を、10・11行目でfull CIの計算を行っています。
出力として
```
 90 RCISD converged
 91 E(RCISD) = -100.0954565405312  E_corr = -0.1361353732494051
 92 E(FCI) = -100.102114656648
```
が得られ、この`E(FCI) = -100.102114656648`（原子単位）がfull CIのエネルギーです。
理論的にはfull CIのエネルギーが、与えられた計算条件での下限値となります。
ただし、摂動理論の場合はfull CIのエネルギーを下回る場合があります。
ここではCISDのエネルギーよりfull CIのエネルギーの方が低くなっており、正しいでしょう。

## 結合クラスター法の計算

私自身は結合クラスターと表現しないので、正しいのか分かりません。
coupled clusterと言っています。
PySCFでは、CCSDとCCSD(T)が人間によって実装され、任意のCCが自動生成されたプログラムにより実行できる感じです。
ここでは人間により実装された物のみを用います。
CCSDのSとDはCIの場合と同じです。
CCSD(T)の(T)が括弧書きされているのは、三電子励起を摂動的に取り扱うことを意味しています。

CCの特徴は、やはりその精度でしょう。
特にCCSD(T)は計算化学のgold standard（金字塔？）とも呼ばれており、「これをやっておけば間違いない」という手法です（間違う場合もあります）。
ですが、その分計算コストも高いです。
CCSD(T)の場合はは、分子軌道の数が二倍大きくなると計算コストが128 (=2<sup>7</sup>)倍増えるとされています。

CIとCCはどちらも見た目（文字のこと）は似たような手法ですが、CISDの場合は一電子励起と二電子励起を露わに取り扱うのに対して、
CCSDの場合は二電子励起の積の形で四電子励起までを取り扱う（disconnected quadruple excitation）というところが違います。
演算子の観点から言うと、CIの場合は励起演算子を直接Hartree--Fockの波動関数に作用させるのに対して、CCの場合はべき乗励起演算子を作用させます。
さらにクラスター展開を行うこ、二電子励起の演算子までしか入っていなくても、積の形で四電子励起を表す項が出てきます。
この項は直接四電子励起（connected quadruple excitation）を扱うよりも寄与が大きいらしいため、わざわざ四電子励起を扱わなくても精度の改善が期待できます。
さらに、disconnectedな項を取り入れることで、truncatedで問題となるsize-extensivityの問題も解決されます。

上のHF（フッ化水素）を用いてCCSDとCCSD(T)の計算をしてみます。
```python
  1 from pyscf import gto, scf, cc
  2 
  3 mol = gto.M(atom = 'H 0 0 0; F 0 0 1.1',basis = 'cc-pVDZ',verbose=4)
  4 mf = scf.HF(mol)
  5 mf.kernel()
  6 
  7 mycc = cc.CCSD(mf)
  8 mycc.kernel()
  9 
 10 from pyscf.cc import ccsd_t
 11 ccsd_t.kernel(mycc, mycc.ao2mo())
```

7・8行目でCCSD計算を、10・11行目でCCSD(T)計算を行っています（本当はMO積分を再利用できるはずですが･･･）。

```
 83 ******** <class 'pyscf.cc.ccsd.CCSD'> ********
 84 CC2 = 0
 85 CCSD nocc = 5, nmo = 19
 86 max_cycle = 50
 87 direct = 0
 88 conv_tol = 1e-07
 89 conv_tol_normt = 1e-05
 90 diis_space = 6
 91 diis_start_cycle = 0
 92 diis_start_energy_diff = 1e+09
 93 max_memory 4000 MB (current use 53 MB)
 94 Init t2, MP2 energy = -100.198764903112  E_corr(MP2) -0.211367462762818
 95 Init E_corr(CCSD) = -0.211367462762861
 96 cycle = 1  E_corr(CCSD) = -0.211501932672108  dE = -0.000134469909  norm(t1,t2) = 0.0247832
 97 cycle = 2  E_corr(CCSD) = -0.215138153362452  dE = -0.00363622069  norm(t1,t2) = 0.00999383
 98 cycle = 3  E_corr(CCSD) = -0.21592338954904  dE = -0.000785236187  norm(t1,t2) = 0.00428212
 99 cycle = 4  E_corr(CCSD) = -0.216389035997118  dE = -0.000465646448  norm(t1,t2) = 0.0014889
100 cycle = 5  E_corr(CCSD) = -0.216345879581836  dE = 4.31564153e-05  norm(t1,t2) = 0.000284001
101 cycle = 6  E_corr(CCSD) = -0.216336822967369  dE = 9.05661447e-06  norm(t1,t2) = 5.5372e-05
102 cycle = 7  E_corr(CCSD) = -0.216337562669644  dE = -7.39702275e-07  norm(t1,t2) = 1.56736e-05
103 cycle = 8  E_corr(CCSD) = -0.216337630463756  dE = -6.77941129e-08  norm(t1,t2) = 2.12784e-06
104 CCSD converged
105 E(CCSD) = -100.2037350708125  E_corr = -0.2163376304637564
106 CCSD(T) correction = -0.00241369116149246
```
CIとは異なり、なんか繰り返し計算を行っています（本当はCIも繰り返し計算を行っています）。
出力ファイルの見方はCIとほとんど同じです。
105行目で、CCSDのエネルギーを表示してくれています（原子単位）。
106行目は、CCSD(T)の補正項を表示しています。
CCSD(T)のエネルギーは、CCSDのエネルギーにこの補正項を足すことで得られます（すなわちここでは`E(CCSD(T)) = 100.20614876197349246`ということになります）。
full CIの計算コストに比べればずいぶんマシですが、それでも普通にやると基底関数が50あたりになると、かなり厳しいかもしれません。

## 摂動理論の計算

一言で摂動理論と言っても、ゼロ次ハミルトニアンの定義が任意であるためいろいろですが、
多くの場合MP2 (second-order Møller--Plesset perturbation theory)のことをを指しています。
ここでもMP2の計算を行ってみます。
MP2は二電子励起を摂動的に取り扱います。
より多くの励起電子配置を取り扱う場合は、MP3, MP4, ...と続きます。
PySCFではMP2のみ実装されています。

MP2の特徴は、比較的低い計算コストで、それなりの精度を出すことでしょう。
ただ、摂動展開が必ずしも収束するとは限りませんし、精度もせいぜい「それなり」です。
結合エネルギーや分散力を過大評価しがちです。
SCS-MP2 (spin-component-scaled)というのもありますが、PySCFではできそうにないです。

MP2の計算は過去に演習問題として取り扱った気もしますが、上と同様にフッ化水素を用いてやってみます。
```python
  1 from pyscf import gto, scf, mp
  2 
  3 mol = gto.M(atom = 'H 0 0 0; F 0 0 1.1',basis = 'cc-pVDZ',verbose=4)
  4 mf = scf.HF(mol)
  5 mf.kernel()
  6 
  7 mymp = mp.MP2(mf)
  8 mymp.kernel()
```
で、出力として以下が得られます。
```
 78 ******** <class 'pyscf.mp.mp2.MP2'> ********
 79 nocc = 5, nmo = 19
 80 max_memory 4000 MB (current use 53 MB)
 81 E(MP2) = -100.198764900659  E_corr = -0.211367460310054
```
まぁ、既にCI/CC計算でこの値は得られていましたね。
CI/CCでは励起強度(?; t-amplitude)を繰り返し計算により得るのですが、その励起強度の初期値としてMP2の強度を用いるためです（たぶん）。
普通のMP2は繰り返し計算を行わずに計算ができるため、計算コストは割と低めです（それでもHFやDFTよりは時間がかかる）。

***

ここで紹介した手法は、二原子分子とかなら良いのですが、少し原子（基底関数）のサイズが大きくなると、
急激に計算時間が増大していきます。
一方でDFTは汎関数にもよりますが、それほど急激に計算コストが増大しません。

波動関数理論の計算で一つ時間がかかるのは二電子積分のAO->MO変換で、これは5乗のスケーリング（系のサイズが2倍になると、計算時間は32 (=2<sup>5</sup>倍）になる）です。これは[density fitting](https://pyscf.org/user/df.html)を用いることで計算時間を減らせます。
取り上げるかは分かりません。
また、対称性を用いることにより様々な計算を省略できるのですが、これも取り上げるかは分かりません。

前段落のAO->MO変換は単なる計算の準備であり、加えて電子相関エネルギーを計算しなくてはいけません。
もちろん電子相関エネルギーの計算にも時間がかかります。
具体的な式（式を見ればスケーリングを推測できます）は各種文献等を参照して欲しいところですが、
例えばCISDは6乗のスケーリング、CCSDは6乗のスケーリング、CCSD(T)は7乗のスケーリング、MP2は4乗のスケーリングと、
それぞれの手法で大体どのようなペースで計算コストが増大するかが決まっています。
しかも、CIとCCは一般的なアルゴリズムでは繰り返し計算を行う必要があるため、さらに収束までに必要な繰り返し計算の回数分だけ計算コストがかかります。

このスケーリングの問題のみならず、
波動関数理論では基底関数の収束性が悪い（大きな基底関数を使わないといけない）というのも難しい問題で、
このためさらに計算時間が長くなりやすく、取り扱える系のサイズが制限されがちです。
それでも、DFTとは異なり手法の精度を系統的に改善できるという性質は重要で、今なお（日本ではともかく）理論開発は活発です。

おそらくここで紹介した全ての手法で一次の性質（特に勾配）の計算が可能です。
使ってみてください。

## 演習問題

* 適切に計算条件を設定し、これまでに用いた手法の計算時間を計測してみましょう。プログラムの書き方によって計算効率が変わるので比較は難しいですが、このページで紹介した中で一番計算時間が短いのはMP2で、長いのはfull CIになるでしょう。
* 非占有分子軌道の軌道エネルギーは占有分子軌道の軌道エネルギーよりも高いため、励起した電子配置を取り入れるとエネルギーが高くなる気がします。もちろんそうなる場合もあります（励起状態計算）が、ここで扱った手法の相関エネルギーは負になっているので、エネルギーは低くなっています。励起した電子配置を取り入れるとエネルギーが低くなる理由を教えてください。
* Size-extensivity（strict separability）について検証してみましょう。どんな分子でも良いのですが、例えばCH<sub>4</sub>を100 Angstromぐらい離して計算したときの相互作用エネルギーを計算してみましょう。正しい理論なら、ほとんどゼロになるはずですが、特にCISDの場合はどうでしょうか。
* 適当な基底関数（あまりに大きいと計算時間がかかる。特にfull CIはメモリ・計算時間に注意。N<sub>2</sub>が厳しければfull CIはしなくて良いです）とこれまでに紹介した手法（CISDは使わなくて良い）を組み合わせて、He<sub>2</sub>やN<sub>2</sub>のポテンシャルエネルギーカーブを描いてみましょう[^4]。特に、N<sub>2</sub>の場合にCCSD(T)は正しそうでしょうか。
* CISDとCCSDは一電子励起に対応する文字が入っており一電子励起の効果が取り込まれているのに、MP2で一電子励起を取り扱わないのはなぜでしょう。また、MP0エネルギーとMP1エネルギーが存在するとして、それらはどのようなエネルギーに対応するでしょうか。ちなみに、MP2法ではゼロ次ハミルトニアンとしてフォック演算子を用いています。
* 時間があれば(SCS-)MP2を実装してみましょう。

[^1]: これ自体は単配置の説明です。単配置と単参照で具体的にどのように区別して定義されているのか実は理解していませんが、単参照をもう少し真面目に書くと「単配置波動関数を基にして（参照して）電子相関計算を行った」場合に単参照電子相関計算を行ったと言うと考えています（正しいかは不明）。
[^2]: Hartree--Fock法の電子相関がゼロと表現する人がいますが、たぶん正しくありません。
交換項が電子相関の一部と考えることができるからです。
定義を明確にした上で、ゼロと表現するなら良いと思います。
[^3]: full CI自体は正しい理論ですが、それは時間非依存・非相対論での理論です。
また、我々の計算では基底関数のサイズが無限に大きくすることができません。
このため（他の理由もあるでしょうが）、full CIで得られた結果が必ず正しいというわけではありません。
与えられた基底関数で、時間非依存・非相対論の枠組みの中では正しい解を与えます。
[^4]: 議論に用いることができるグラフにすること。見た目で重なっているグラフでは議論ができない。
