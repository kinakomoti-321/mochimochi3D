---
title: "薄レンズモデル(ThinLens Model)の実装"
date: 2021-10-21T18:41:26+09:00
archives: "2021"
tags: []
draft: false
author: John SMITH
---
# 被写界深度
　ピンホールカメラモデルでは所謂「ボケ」というものが存在せず、どこまで遠くの物体でもくっきりとした画像を出すことができる。しかし、現実で見るような写真は背景や手前のものがボケている（もちろん肉眼でも）。このようになるのは一般的にカメラなどがレンズを通して光を受け取っているためである。
　

　レンズは焦点を持っており、焦点を通る光は平行に、平行な光は焦点を通るように光は曲げられる。このような性質のため、ある一点から来る光はある一点に収束することになる。センサーの位置が丁度その収束点に付けば光が１対１の関係になり、明瞭に記録ができる（ピントが合う）。しかしながら、その物体が前後すると収束点も移動するため、ピントが合う位置以外のある点から来た光は他のセンサーに当たってしまうことになる。これがボケの原因となる。


　ピントが合って明瞭に見える深度は被写界深度と呼ばれ、被写界深度から外れた位置にいる物体はカメラではボケたものとして表現される。このような被写体深度をレイトレでやる場合、カメラのレンズをシミュレーションすることで実現できる。

　こうしたカメラレンズの簡単なモデルとしては薄レンズモデル(ThinLens Model)というものがあります。[ThreeJsでパストレを実装した記事]()でも実装方法を記載したのですが、その時の実装方法では足りないことや絞りの実装などいくつか後で追加したため一通りまとめるために今回記事にしました。この記事ではカメラレイの生成、ウェイトの計算、絞りの実装について載せていこうと思います。

# 薄レンズモデルとは
　薄レンズモデルはカメラのレンズが厚みのない１枚の凸レンズで構成されているものとするモデルです。イメージとしては虫メガネを使うカメラのようなものです。厚みがないので屈折をそのまま計算するわけではなく、レンズの法則から光が曲がる方向を計算していきます。薄レンズモデルは簡単なものの被写体深度が表現でき、ピンホールカメラに比べると結構リアルな画像を出すことができます。
　
　

　薄レンズモデルの処理の流れとしては

1. レンズの位置、大きさなどの計算
2. レンズ上の点をサンプリング
3. 光が向かう方向の計算、カメラレイの生成

というような形で行います。また、サンプリングを行う関係でカメラのレイに重みを付ける必要が出てきます。それも併せてやっていくことになります。

## カメラレイの生成
　まず、レンズの位置などについて求める必要があります。
　一般的なレンズに成り立つ法則としてレンズの法則があります。焦点距離$f$のレンズから$b$の距離にある点が放つ光はレンズを通って反対側にレンズからの距離$a$の点に再び集まります。この時の各距離について理想的に
$$ \frac{1}{a} + \frac{1}{b} = \frac{1}{f} $$

という関係が成り立ちます。これがレンズの法則です。

{{< figure src="../Pic1.png" class="center">}}

　焦点距離$f$というのはレンズの特性を示すパラメータであり、こちらから与える量になります。そして$a$か$b$かを与えれば、自動的にレンズの法則より計算することができます。センサーとレンズの距離$a$をこちらで設定するとなると直感的にピントが合う距離というのがわかりづらいため、レンズから物体までの距離$b$を与えてそれにピントが合うような$a$を作る方が良いと思います

　レンズの法則より$a$は$f$と$b$を与えて
 $$ a = \frac{bf}{b-f} $$
 と求められます。従って、レンズのポジション$\vec{C}$はカメラの向き$\vec{n}$と位置$\vec{Cam}$とした時
 $$ \vec{C} = a\vec{n} + \vec{Cam} $$
 と得ることができます。

　また、レンズのもう一つの特性としてレンズそのものの大きさがあります。レンズの有効な半径$R$に対して焦点距離を用いて
$$ F = \frac{f}{2R} $$

という値が作れます。これはF値(F number)と呼ばれているパラメータで、レンズの明るさを示す指標として用いられるものです。一般的にF値を与えてから、レンズがどの程度の半径を持つかを計算するため、レンズの半径$R$については
$$ R = \frac{f}{2F} $$

と求めます。以上によって$b$,$f$,$F$の３つのパラメータを設定することでレンズの設定が完了します。

　次にカメラのレイを作っていくことになりますが、レンズを通した場合におけるセンサーに入る光はピンホールカメラと違いレンズから様々な方向から来ます。そのため、カメラから出るレイをサンプリングしてやる必要が出てきます。レイのサンプリング方法はレンズ上の点$x_0$をサンプリングして、センサー上の点$x_p$と繋ぎ、レンズに曲げられる方向へのレイを$x_0$から飛ばすといった形で行います。

　今回の実装では極座標について、一様乱数$u,v$を用いて
$$r_0 = Ru$$
$$\theta_0 = 2\pi v$$

というようにレンズ状の点をサンプリングしました。ワールド空間上でのサンプル点はカメラのローカル基底$\vec{n_x},\vec{n_y}$を用いれば、
$$ \vec{x_0} = C + r_0 \cos{\theta_0} \vec{n_x} + r_0 \sin{\theta_0}\vec{n_y}  $$
と求めることができます。

　次にこのレンズ上の点に向かう光はレンズを通ってどの方向に飛ぶのかを考えなくてはなりません。レンズの性質として、ある点からの光はレンズを通って１つの収束点へと向かいます。即ち、その収束点に向かうような方向を計算することになります。

　レンズの法則より収束点$\vec{P}$はレンズから$b$の距離にあります。また、レンズの中心を通る光というのは曲げられることがなく直進します。この時の位置関係は図のようになります。

　レンズを通る光の方向ベクトル$\vec{e}$はセンサーの点$\vec{x_p}$とレンズの位置$\vec{C}$を結ぶ単位ベクトルであるため
$$ \vec{e} = \mathrm{Normalize}(\vec{C}-\vec{x_p}) $$
　
と求めることができます。これを用いると幾何的な関係から収束点$\vec{P}$が以下のように求められます。
$$ \vec{P} = \vec{C} + \frac{\cos{\theta}}{L}\vec{e}$$

以上によってレイの方向ベクトル$\vec{d}$は
$$ \vec{d} = \mathrm{Normalize(\vec{P} - \vec{x_0})}$$

と求めることが出来ます。結果としてカメラのレイとしては
$$ \mathrm{Ray}(\vec{x_0},\vec{d}) $$

となります。このようなレイを作ることで薄レンズモデルのカメラを作ることができます。

## ウェイトの計算
　カメラレイの生成には乱数を使用しており、カメラレイはパストレーサーとしてパスのウェイトを持つ必要があります。カメラウェイトの導出に関しては@shockerさんの記事に非常に細かく解説されております。

[被写界深度 (Depth of Field)](https://rayspace.xyz/CG/contents/DoF/)

こちらの記事によるとカメラのウェイト$W_{camera}$は
$$ W_{camera} = \frac{|\vec{\omega}_{x_0\rightarrow x_1} \cdot \vec{n}|}{P_{lens}(x_0) P_\sigma(x_0 \rightarrow x_1)}$$
それぞれについては
- $\vec{\omega}_{x_0\rightarrow x_1}$ : $x_0$ から $x_1$への方向ベクトル
- $\vec{n}$ :　レンズの法線
- $P_{lens}(x_0)$ : レンズ上の点$x_0$を取る確率密度
- $P_\sigma(x_0\rightarrow x_1)$ : $x_0$ から $x_1$の方向へサンプルする確率密度

となっている。
 
　まず、$P_{lens}(x_0)$については$r_0$と$\theta_0$の極座標のサンプリングを
$$r_0 = r_{lens}u$$
$$\theta_0 = 2\pi v$$
で行っていたため、
$$P_{lens}(x_0) = P_{lens}(r_0,\theta_0) = \frac{2\pi}{r}$$
という形になる（一様サンプリングをしても良かったかも）。

　次に、$P_\sigma(x_0\rightarrow x_1)$についてだがこちらは結構複雑になっている。$x_0$が決まれば一義的に決まるから単に１になるのではないかと考えたが、実際はそうではなく以下のような式で表される。

$$ P_\sigma(x_0\rightarrow x_1) = \frac{a^2}{|\vec{\omega}_{x_0\rightarrow x_1}|^3} P_A(x_p)$$ 

$a$は画素からレンズまでの距離です（レンズの法則での$a$)。この導出は@shockerさんの記事をご覧ください。

　$P_A(x_p)$とは何かというとセンサーの点$x_p$を選ぶ確率密度であり、このカメラウェイトを計算する際は、カメラの画素の点をサンプリングしたものとして考える必要があります。しかしながら、カメラの画素上のサンプリングは固定されているか、アンチエイリアスによって一様サンプリングされていることが大半です。

そのため、$P_A(x_p)$は一般的には定数で表されることになると考えられます。
$$ P_A(x_p) = P_A = const. $$

後述の理由でP_Aの値は特に気にする必要はありません。

以上をまとめるとカメラウェイトは
$$ W = \frac{|\vec{\omega}_{x_0\rightarrow x_1} \cdot \vec{n}|^4r}{2\pi a^2} P_A$$

となります。これをパストレーサーで計算した寄与に掛けて上げればウェイトの実装は完成です。

　コサイン項が入ることで現実のカメラで現れるエフェクトの１つ、口径食(vignetting)が現れます。これは画面端辺りが暗くなる現象で、より現実に近い表現を作ることができます。

　しかしながら、これをそのままやると実は真っ暗になります（場合によっては極端に明るくなるかも）。この式が間違ってるというわけではなく、真っ暗になることが現実的に正しいためです。現実のカメラはセンサーに感度がつけられており、かなり暗い光に大して良く見えるように感度を高める形で画像を作っています。このため、ウェイトにさらに感度(sensitivity)を追加して調整する必要があるそうです。

　感度の追加自体は単なる係数を付けてあげればいいので、

$$ W = \frac{|\vec{\omega}_{x_0\rightarrow x_1} \cdot \vec{n}|^4r}{2\pi a^2} P_A \times Sensitivity$$

という形で簡単に実装できます。この時、$P_A$は定数であるため、Sensitivityの値に含めることができて最終的には

$$ W = \frac{|\vec{\omega}_{x_0\rightarrow x_1} \cdot \vec{n}|^4r}{2\pi a^2} Sensitivity$$

という計算式を用いれば実装ができます。
一応,サンプリングによって確率密度は変わるので、確率密度を書けば

$$ W = \frac{|\vec{\omega}_{x_0\rightarrow x_1} \cdot \vec{n}|^4}{P{lens}(x_0) a^2} Sensitivity$$

ともなります。これをパスの重みとして最後にかけてあげることでレンズのウェイトが実装できます。

## 絞りの実装
　カメラの絞りはカメラのレンズの前に覆いかぶさる形で光を遮蔽することで、カメラに入る光の量を調整します。なので、レイトレでの実装方法は単に絞りに当たらないレンズ上の点を選択するだけで実装が行えます。

　よくあるカメラの絞りのイメージは六角形や円形だったりします。今回は六角形の絞りの実装を行いましたが、単に絞りの形状さえ変えれば他の形の絞りも作ることができます。

　私は２つの実装方法を試してみましたので、両方の実装方法を記載します。

### 距離関数での判定
　レンズ上の点のサンプリングは２次元上で行ってましたが、この時距離関数を使ってサンプル点が絞りにあるかどうかを判定することで行う方法です。

　六角形の距離関数はIQさんの以下のサイトに載っており、点が六角形の外にある時は正を、中にある時は負を返すような関数になっています。この関数を絞りの形状として、レンズ上のサンプルした点で距離関数に入れた時、正を返せば即ち絞りに遮られる点として寄与を取らないようにすれば絞りを作ることができます。

　しかしながら、この方法では寄与が全くない点をサンプルするため、収束が遅くなります。再びサンプリングして絞りに当たらない点を取るまでやるということも考えられますがどちらにしろ無駄な処理が増えてしまいます（利点を挙げるとすればどんな形状でもサンプルできることぐらい）。そのため、寄与がある点だけをサンプリングするようにした方法として次の方法を取りました。

### 六角形のサンプリング
　こちらの方法は直接レンズのサンプリングを六角形のサンプリングにしてしまう方法です。方法についてはこちらのサイトを参考にさせて頂きました。

 [３次元空間、複数三角形内に均一に、点をばらまく](https://qiita.com/Ushio/items/dc4ce75337360de51e00)

　n角形のサンプリングはn個の三角形のサンプリングという形で実装することができ、六角形も同様に６つの三角形を用意し、その中から一つを選び三角形のサンプリングをするという形でサンプリングが可能です。

　具体的な手順としては
1. 一様乱数で0,1,2,3,4,5の番号を作る
2. 番号に応じた三角形で一様サンプリングを行う
3. pdfを計算する
  
  という形で行う。

  私の実装では一様乱数で手順1を行い、事前に作った三角形を番号分回転させることで番号に応じた三角形を作りました。

  三角形の一様サンプリングはそれぞれの頂点が$\vec{A},\vec{B},\vec{C}$として、一様乱数を$u,v$とした時、

  $$ \vec{P} = \vec{A}(1 - \sqrt{u}) + \vec{B}(\sqrt{u}(1 - v)) + Cv\sqrt{u} $$

  と行うことでできる。この時の確率密度$P_{tri}$は三角形の面積の逆数になるため、外積で
  $$ P_{tri} =\frac{2}{|(\vec{B} - \vec{A})\times (\vec{C} - \vec{A})|}$$

  と求めることができます。また、六角形のサンプリングであったため、これにさらに1/6をかけて
  $$ P_{hex} = \frac{1}{3|(\vec{B} - \vec{A})\times (\vec{C} - \vec{A})|} $$

  となる。サンプリングそのものが変わるため、ウェイトの方にこの確率密度を与えることで実装ができます。

  　このように六角形のサンプリングが可能なため、光が通る点のみをサンプリングするできるようになります。これを使用した絞りは距離関数に比べ収束が目に見えて早くなります。n角形や円形のサンプリングもできるため、絞りの実装はこちらのようなサンプリングによる実装の方が良いと思います。

# 実装
GLSLでの実装を行いました。
```
Ray thinLensCamera(vec2 uv,vec3 atlook,vec3 camerapos,inout bool ap,inout float weight){

 //カメラの各パラメータ (外部から受け取っている)
    float f = cameraLens.x;
    float F = cameraLens.y;
    float b = cameraLens.z;

    float a = b * f / (b - f);
    
    //カメラのローカル基底
    vec3 up = vec3(0,1,0);
    vec3 cw = normalize(atlook);
    vec3 cu = normalize(cross(cw,up));
    vec3 cv = normalize(cross(cu,cw));
    
    //カメラのポジション
    vec3 X0 = camerapos + uv.x * cu + uv.y * cv;
    vec3 C = camerapos + cw * a;
    vec3 e = normalize(C - X0);

    //レンズ上の位置サンプリング
    //ランダム関数
    vec2 xi = hash23(vec3(iTime * uv.x, iTime * iTime * uv.y , iTime * iTime * iTime));
    xi = hash22(xi);
    vec3 S;
    vec3 P;
    float pdf = 1.;
    if(!TestCheck){
      float phi = 2.0 * M_PI * xi.x;
      float r = xi.y * f / (2.0 * F);
      S = C + r * cos(phi) * cu + r * sin(phi) * cv;
      P = C + e * b / dot(e,cw);
      pdf = 2.0 * M_PI / r;
      
      ap = cameraAperture(vec2(r * cos(phi),r * sin(phi)),f/(2.0 * F));
    }
    else{
        float LensR = f/(2.0 * F) * cameraAp;
        float TriN = float(floor(xi * 6.0));
        //三角形の点
        vec2 TriA = vec2(0.0);
        vec2 TriB = rotate(vec2(0.0,LensR),TriN * 2.0 * M_PI / 6.0);
        vec2 TriC = rotate(vec2(0.0,LensR),(TriN + 1.0) * 2.0 *M_PI/6.0);
        
        //三角形の一様サンプリング
        xi = hash22(xi);
        float sw = sqrt(xi.x);
        vec2 TriP = TriA * (1.0 - sw) + TriB * (sw*(1.0 - xi.y)) + TriC * sw * xi.y;
        S = C + TriP.x * cu + TriP.y * cv;
        P = C + e * b / dot(e,cw);
        
        //pdf
        pdf = 2.0 * 1.0 / (abs(length(cross(vec3(TriB,0),vec3(TriC,0))))*6.0);
        ap = false;
    }
    //カメラレイの生成
    Ray camera;
    camera.origin = S;
    camera.direction = normalize(P - S);
    
    //ウェイト
    float cosine = abs(dot(camera.direction,cw)); 
    weight = cosine * cosine *cosine * cosine / (pdf * a*a);
    return camera;
}
```
# 終わりに
 　レイトレの長所としてどんな現実のカメラでも再現でき、被写体深度などカメラ特有のエフェクトを正確に作ることができることがあります。薄レンズモデルは特に被写体深度という一番目立つエフェクトを作ることができるレイトレならではを感じさせるよい簡単なモデルです。
 　

　しかしながら、実際の一眼カメラなどは幾つものレンズが組み合わさっており、薄レンズモデルのように１枚のレンズだけを使うカメラはありません。レンズフレアなどはそうした複数レンズ特有のエフェクトであり、より写実的な絵を出したいとなると現実のレンズ設計をシミュレーションすることとなってきます。

　より発展的なカメラモデルを知りたいとなれば、yumcyaWizさんの記事が複数レンズのカメラモデルについてご解説なさっていますのでそちらをご覧ください。

 [写真レンズのレイトレーシング](https://blog.teastat.uk/post/2020/09/raytracing-of-photographic-lens/) by yumcyaWiz


# 参考文献
[レイトレース：薄いレンズのカメラ](https://t-pot.com/program/121_RayTraceThinLens/index.html)

[３次元空間、複数三角形内に均一に、点をばらまく](https://qiita.com/Ushio/items/dc4ce75337360de51e00) by Ushio

[被写界深度 (Depth of Field)](https://rayspace.xyz/CG/contents/DoF/) by  shocker