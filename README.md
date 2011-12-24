# PyWiiRemote

## 何をやるのか？
任天堂Wiiのゲームコントローラと言えば皆さんご存知のWiiリモコンには、3軸の加速度センサとIRカメラ(赤外線を撮影出来る)が入っています。また、今までWiiモーションプラスとして外部拡張コネクタに接続するという形で使うことができた3軸ジャイロセンサは、現在発売されているWiiリモコンプラスでは本体に取り入れられました。このWiiリモコンは通常Wii本体とBluetoothという無線通信の規格で接続し、データのやり取りをするのですが、Wii本体でなくPCと接続することでWiiリモコンからの様々なデータをPCから取得することも出来ます。しかもWiiリモコンは、普通に売っているセンサ類より安く、しかもある程度の精度はあるため、なかなかお得なデバイスなのです。そこで、Wiiリモコンに入っている加速度センサとジャイロセンサから取得できるデータをもとにしてWiiリモコンの姿勢を計算する、というプログラムをPythonという言語で作成してみました。
加速度センサ・ジャイロセンサから取得した値から求まる姿勢には、どういう動きの時は正確に測定できる、あるいはできないといった、それぞれの長所と短所があります。最終的にはその2つのセンサからのデータを合わせて、どちらか片方の時より実際のWiiリモコンの動きに近くなるよう工夫してみました。

## 動作方法
* *tiny_hid_dll.dllは、<http://www.kako.com/neta/2006-019/2006-019.html>から使わせていただいております。*
* Windowsのみ対応。
* 必要なモジュール
 * [VPython](http://vpython.org/)
 * [Pygame](http://pygame.org)


## 各センサについて
近年発達したMEMSという技術によって超小型化された各センサはそれぞれ

* 加速度センサ - `加速度 (１秒にどれだけ動く速度が上がるか)`
* ジャイロセンサ - `角速度 (１秒にどれだけ回転するか)`

を測ることが出来ます。これらは

* カーナビやエアバッグ、カメラの手ぶれ補正
* 傾けて画面の表示向きの切り替え

などに利用され、今話題のスマートフォンにも大抵これらのセンサが内蔵されており、ユーザーが直感的な操作ができるようにうまく利用されています。

## 姿勢を計算する方法
![Wiiリモコンの座標系](https://github.com/pheehs/PyWiiRemote/raw/master/coordinate_system.png "Wiiリモコンの座標系") [参照](http://www.cg-ya.net/i-media/wiiremote/spec.html)  
Wiiリモコンでは上図のような座標系となります。
3次元的な回転を表す時には

* Yaw(水平角) = (周囲を見渡したときの視線の動き方)
* Pitch(垂直角) = (頷いたときの視線の動き方)
* Roll(回転角) = (首をかしげたときの視線の動き方)

という軸が使われます。

### 加速度センサの場合
#### 原理
加速度センサから姿勢を求めるには重力を利用します。重力は常に地球の中心、つまり下方向にかかる重力加速度として測れば、データとして見ることが出来ます。3軸のそれぞれの方向からどれだけの角度ずれた方向に重力がかかっているか、つまりはセンサ自身の座標系においての実際の下方向がどっちにあるのかがわかるので現実世界の座標系から見たセンサの傾きを計算することが可能です。  
![加速度センサで姿勢を計算](https://github.com/pheehs/PyWiiRemote/raw/master/acc_sensor.png "加速度センサで姿勢を計算") [参照](http://www008.upp.so-net.ne.jp/funfly/adxl202.html)  
この図はその計算の仕組みを表しています。
gは重力加速度を表します。この図で言えば、センサの左方向にかかる加速度を重力加速度で割り、アークサインに渡すことでθが求まります。

#### 欠点
* 重力以外の力も加わった場合に、重力とその力が合わさった加速度しか分からないため、その合わさった加速度を重力加速度だとして計算し、ずれが生じます。
* Yaw方向に動かしたときは重力加速度は変わらないのでPitchとRoll方向の傾きのみ検出できます。

#### 利点
* 静止している状態ではセンサの姿勢が正確に分かります。

### ジャイロセンサの場合
#### 原理
角速度に時間をかけて角度の変化を求め、静止した状態で測った初期位置にその変化を足すことで姿勢を計算します。

#### 利点
* 加速度センサと違って、Yaw方向の動きも計測出来ます。

#### 欠点
* 変化を足していくので一度誤差が入るとその影響がずっと残ります。この誤差は、キャリブレーションをすることで幾分か抑えることは出来ます。Wiiモーションプラス対応のWiiのソフトで「動作がおかしかったら、数秒待ってください」といった注意書きが出てくるのは、バイアス(静止時の初期値)を設定しなおすことでキャリブレーションをしているのだと予想されます。

### 2つを合わせる場合
結局のところ、

* 加速度センサ方式  
 -> **現在の姿勢の角度**を求めるもの
* ジャイロセンサ方式  
 -> 回転によって**変化した角度**を求めるもの

という違いがあるのです。
この特性から、もう一方のセンサよりも算出した誤差が小さくなるのは

* 加速度センサ  
 -> ほぼ静止、あるいは重力以外の力をほとんど加えずにゆっくり動いているとき
* ジャイロセンサ  
 -> センサに動きがあるとき

すると、２つのセンサからのデータをうまく組み合わせる方法が見えてきます。
このプログラムでは

* **３軸の加速度の平均がルート2G以下のとき**  
 - (加速度：ジャイロ = )10:0
* **３軸いずれかの加速度の符号が逆になったとき**  
 - 0:10
* **角速度から求めた回転角が３軸とも36度以下のとき**（細かく振動させた時）  
 - 0:10
* **加速度から求めた姿勢角がどの２つの軸の組み合わせでも90度以下のとき**  
 - 10:0
* **加速度が各軸2G 未満のとき**  
 - 1:9
* **加速度が各軸2G 以上のとき**  
 - 0:10

といった条件による場合分けをして精度を上げました。
また、加速度方式とジャイロ方式を一緒に処理するために、加速度方式で得られた姿勢角から前回の姿勢角との差分と、ジャイロ方式の回転角を組み合わせるという形をとっています。  
ただ、この方式のせいか本来はジャイロ方式における誤差が蓄積してしまっても、静止状態に戻せば本来は加速度方式で正確な姿勢が求まるはずなのですが、静止状態でもずれていくことがあります。  
まだ誤差が目立つので、試行錯誤を重ねればよりよいロジックを創り出せると思います。

## 参考資料
* [wikipedia Wiiのコントローラ](http://ja.wikipedia.org/wiki/Wii%E3%81%AE%E3%82%B3%E3%83%B3%E3%83%88%E3%83%AD%E3%83%BC%E3%83%A9)
* [WiiBrew](http://wiibrew.org/wiki/Wiimote Wiimote)
HIDによるBluetoothでの通信がほとんどまとめられている。
* [ジャイロ＋加速度センサの統合](http://moyane.blog25.fc2.com/blog-entry-288.html)
* [WiiYourself!でWiiリモコンプラスを使う際の注意点](http://www.cg-ya.net/i-media/wiiremote/wiiyourself_wiiremoteplus.html)
Wiiリモコンプラスでジャイロのデータにアクセスする方法。
* [小ネタ](http://www.kako.com/neta/2006-019/2006-019.html)
このサイトのtiny_hid_dll.dllを使ってWiiリモコンとの通信をした。
* [加速度センサ，角速度センサのしくみ](http://www.cqpub.co.jp/dwm/contents/0117/dwm011700700.pdf)
MEMS技術と各センサについて触れられている。
* [慣性センサ用語集](http://www.xbow.jp/inertial.html)
