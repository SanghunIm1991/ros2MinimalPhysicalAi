# 間章: Gazeboの物理シミュレーションの基本（フェーズ5の補足）

この章は、フェーズ5-4（[`docs/phase5_4_gazebo_plant.md`](phase5_4_gazebo_plant.md)）までを読み終えた人向けの、任意の章である。フェーズ5では、Gazeboに付属する2輪車のワールドを写し、車輪にトルクをかけて走らせた。車両の形や重さ、地面との摩擦は、付属のファイルに書かれたものをそのまま使い、中身にはほとんど触れなかった。けれども、自分のロボットをシミュレーションしたくなったら、ロボットの形・重さ・関節を自分で書き、地面や物との接触がどう計算されるかを知っておく必要がある。この章では、箱を落とす、坂で滑らせる、球を跳ねさせる、腕を振り下ろす、という小さな体験を通して、ワールドとロボットの書き方と、物理エンジンが何を計算しているかをひととおり見ていく。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy + Gazebo Harmonic（フェーズ5-0で導入したもの）
- 前提: フェーズ5-4（[`docs/phase5_4_gazebo_plant.md`](phase5_4_gazebo_plant.md)）。フェーズ6・7の本文は、この章に依存しない
- 所要目安: 1〜2コマ
- 言語: なし（XMLのファイルを書き、コマンドで動かす。PythonやC++のコードは書かない）

> **この章の位置づけ**: 小さな体験つきの読み物である。体験に使うファイルは、どれも数十行のXMLで、パッケージやビルドは要らない。練習のファイルは、ワークスペース（`~/ros2_ws`）とは別のフォルダ `~/gz_practice` に置く。サンプルは学習の手がかりとして最小限に書いたもので、公式チュートリアルの転載ではない（要素の名前と書き方は公式の文書と同等。出典は末尾に記載する）。センサの節は、今後書き足す予定である。

> **期待する結果の出どころ**: `gz sdf`・`xacro`・`check_urdf` の表示は、この章の作成時に実際に実行した表示。2〜3節の位置の数値（`gz model` の表示）は、作成時にGazeboを画面なしで起動し、一時停止したまま決めた刻みだけ進めて読んだ表示。4-4節の角度（`ros2 topic echo` の表示）とノードのログは、作成時に画面なしのGazeboで同じ手順を実行し、15秒進めた後に読んだ表示で、ログの時刻と `stamp` は実行ごとに異なる。5節のノード・コントローラ・アクションの表示と角度、5-4節の表の角度は、作成時に画面なしのGazeboで同じ手順を実行して読んだ表示（5-3節のT1〜T4の起動のログと最初の `spawner` のログは、ほかの表示と別の回の実行のもの）で、ログの時刻・`stamp`・アクションの `ID` は実行ごとに異なる。Gazeboの画面での見え方は、その数値から筆者が想定したもの。

## 0. この章で得てほしいこと

1. Gazeboのワールドの記述（SDF）の入れ子（ワールド・モデル・リンク・関節）と、リンクが持つ3つの情報（慣性・衝突・見た目）を説明できる。
2. 物理エンジンが刻みごとに動きを計算していることと、刻み幅（`max_step_size`）と実時間の倍率（`real_time_factor`）の意味を説明できる。
3. 摩擦係数と反発係数を書き換えて、滑り方と跳ね方が変わることを、手計算の値と比べて確かめられる。
4. ロボットの記述（URDF）とSDFの違いを説明でき、xacroで書いた2関節のアームをGazeboに出して、関節の角度をROS2のトピックで読める。
5. ros2_controlの部品（ハードウェアの部品・インターフェース・コントローラ・コントローラマネージャ）の役目を説明でき、位置と軌道のコントローラでアームを動かせる。

完了条件: 2〜4節の体験で、Gazeboが計算した位置・角度が、手計算の値（落下・坂・反発・関節の制限）とどこまで合い、どこがなぜずれるかを説明できる。5節の体験で、アームが位置のコントローラの目標の角度に止まり、軌道のコントローラの軌道を少し遅れて追うことを確かめ、その理由を説明できる。

## 1. 全体像

![ワールドの記述（SDF）をgz simのサーバーが読み込み、物理エンジン（DART）とシステムが刻みごとに動きを計算して、様子を画面（GUI）に送る。ロボットの記述（URDF）は、xacroから展開して作り、createでSDFに変換してワールドに出す。gz simとROS2のノードは、ros_gz_bridgeがトピックを中継してつなぐ](img/interlude_gazebo_overview.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart LR
    W["ワールドの記述<br/>SDF（.sdf）"] -->|"読み込む"| S["gz sim（サーバー）<br/>物理エンジン（DART）<br/>＋システム（プラグイン）"]
    X["xacro<br/>（.urdf.xacro）"] -->|"展開する"| U["ロボットの記述<br/>URDF（.urdf）"]
    U -->|"create"| S
    S -->|"様子"| G["画面（GUI）"]
    S <-->|"gzのトピック"| B["ros_gz_bridge"]
    B <-->|"ROS2のトピック"| R["ROS2のノード<br/>robot_state_publisher 等"]
```

</details>

Gazeboでシミュレーションを動かす部品は、大きく4つある。

- **記述のファイル**: 何がどこにあるかを書いたファイル。ワールド全体はSDF（Simulation Description Format）で書く。ロボット1台は、SDFでも書けるが、ROS2の世界ではURDF（Unified Robot Description Format）で書くことが多い。URDFは、そのままでは長くなりやすいので、マクロで短く書けるxacroで書いて、URDFに展開する（4節）。
- **サーバー（`gz sim`）**: 記述を読み込み、物理エンジンで、決まった刻み幅ごとに、すべての物の位置と速度を計算し直す。物理エンジンの既定はDART（3-1節）。物理の計算そのもの、外からのコマンドの受け付け、状態の配信などは、システム（プラグイン）という部品が受け持ち、ワールドやモデルの記述に `<plugin>` で書いて足す。
- **画面（GUI）**: サーバーから届く状態を3Dで描き、再生・一時停止などの操作を受け付ける。`gz sim -s` で起動すると、画面を開かずにサーバーだけが動く。
- **ブリッジ（`ros_gz_bridge`）**: Gazeboのトピック（gz-transport）とROS2のトピックは別の仕組みなので、間で型を変換しながら中継する（フェーズ5-0の4-1節で使った `parameter_bridge`）。

フェーズ5の車両は、この全部を付属のワールドファイルとlaunchファイルで一度に起動していた。この章では、記述のファイルを自分で書き、1つずつ起動して、それぞれの役割を確かめる。

## 2. SDFでワールドとモデルを書く

### 2-1. 入れ子の構造

SDFのファイルは、次のように入れ子になったXMLである。単位は、長さがメートル、質量がキログラム、角度がラジアン、時間が秒（SI単位）。

| 要素 | 意味 | 主な中身 |
|---|---|---|
| `<world>` | ワールド全体 | 物理の設定（`<physics>`）、重力（`<gravity>`）、光（`<light>`）、システム（`<plugin>`）、モデル（`<model>`） |
| `<model>` | ひとまとまりの物（地面、箱、ロボット） | 置く位置（`<pose>`）、動かない物かどうか（`<static>`）、リンク、関節、システム |
| `<link>` | 形と質量を持つ1つの部品（剛体） | 慣性（`<inertial>`）、衝突の形（`<collision>`）、見た目（`<visual>`） |
| `<joint>` | リンクとリンクのつなぎ目（関節） | 親と子のリンク、回る軸（`<axis>`）、動ける範囲（`<limit>`）、減衰（`<dynamics>`） |

リンクの3つの中身は、役割がはっきり分かれている。

- **慣性（`<inertial>`）**: 質量（`<mass>`）と慣性モーメント（回転のしにくさ、`<inertia>`）。物理エンジンが、力から加速度を計算するのに使う。
- **衝突（`<collision>`）**: 物理エンジンが、ぶつかったかどうか（接触）を判定する形。摩擦や反発の係数も、ここの `<surface>` に書く（3節）。
- **見た目（`<visual>`）**: 画面に描く形と色。物理の計算には使われない。

衝突と見た目を分けて書くのは、計算を軽くするためである。見た目は細かな形（メッシュ）にしても、衝突は箱や円柱のような単純な形で近似する、という使い分けがよく行われる。この章のサンプルは、どちらも同じ単純な形にしている。

`<pose>` は `x y z ロール ピッチ ヨー` の6つの数で、位置（m）と向き（rad。x軸・y軸・z軸の周りの回転）を表す。何も指定しなければ、親（ワールドに直接置いたモデルなら、ワールド）から見た位置になる。

### 2-2. 体験: 箱と球を落とす

地面の上2 mの高さに、箱と球を1つずつ置き、落としてみる。まず練習のフォルダを作る。

```bash
mkdir -p ~/gz_practice

cd ~/gz_practice
```

ファイルを作る方法（`touch` と `nano`、またはVS Code）は、[サンプルコードを練習環境に置く方法](howto_place_code.md)と同じである。

ファイル: `gz_practice/drop.sdf`

<!-- file: gz_practice/drop.sdf -->
```xml
<?xml version="1.0" ?>
<!-- 地面の上に、箱と球を2 mの高さから落とすだけのワールド -->
<sdf version="1.11">
  <world name="drop">
    <physics name="1ms" type="ignored">
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1.0</real_time_factor>
    </physics>
    <gravity>0 0 -9.8</gravity>

    <plugin filename="gz-sim-physics-system" name="gz::sim::systems::Physics"/>
    <plugin filename="gz-sim-user-commands-system" name="gz::sim::systems::UserCommands"/>
    <plugin filename="gz-sim-scene-broadcaster-system" name="gz::sim::systems::SceneBroadcaster"/>

    <light type="directional" name="sun">
      <cast_shadows>true</cast_shadows>
      <pose>0 0 10 0 0 0</pose>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <direction>-0.5 0.1 -0.9</direction>
    </light>

    <model name="ground">
      <static>true</static>
      <link name="link">
        <collision name="collision">
          <geometry><plane><normal>0 0 1</normal><size>20 20</size></plane></geometry>
        </collision>
        <visual name="visual">
          <geometry><plane><normal>0 0 1</normal><size>20 20</size></plane></geometry>
          <material><diffuse>0.7 0.7 0.7 1</diffuse></material>
        </visual>
      </link>
    </model>

    <model name="box">
      <pose>0 0 2 0 0 0</pose>
      <link name="link">
        <inertial>
          <mass>1.0</mass>
          <inertia>
            <ixx>0.00667</ixx><iyy>0.00667</iyy><izz>0.00667</izz>
            <ixy>0</ixy><ixz>0</ixz><iyz>0</iyz>
          </inertia>
        </inertial>
        <collision name="collision">
          <geometry><box><size>0.2 0.2 0.2</size></box></geometry>
        </collision>
        <visual name="visual">
          <geometry><box><size>0.2 0.2 0.2</size></box></geometry>
          <material><diffuse>0.2 0.4 0.9 1</diffuse></material>
        </visual>
      </link>
    </model>

    <model name="ball">
      <pose>1 0 2 0 0 0</pose>
      <link name="link">
        <inertial auto="true">
          <density>500</density>
        </inertial>
        <collision name="collision">
          <geometry><sphere><radius>0.1</radius></sphere></geometry>
        </collision>
        <visual name="visual">
          <geometry><sphere><radius>0.1</radius></sphere></geometry>
          <material><diffuse>0.9 0.3 0.2 1</diffuse></material>
        </visual>
      </link>
    </model>
  </world>
</sdf>
```

ファイルの中身を、上から順に読む。

- `<physics>`: 物理エンジンの設定。1刻みを0.001秒（1 ms）にし、現実の時間と同じ速さで進める（3-1節）。`type` は今のGazeboでは使われないので、`ignored` と書いておく。
- `<gravity>`: 重力加速度のベクトル（m/s²）。z軸の負の向き（下向き）に9.8。書かなければ、この値が既定である。
- `<plugin>` の3行: このワールドで動かすシステム。物理の計算（`Physics`）、モデルを出したり消したりするコマンドの受け付け（`UserCommands`）、画面への状態の配信（`SceneBroadcaster`）。`Physics` を書き忘れると、何も落ちない。
- `ground`: `<static>true</static>` の動かないモデル。無限に広い平面（`<plane>`）を衝突の形にし、見た目は20 m四方にする。
- `box`: 一辺0.2 m、質量1 kgの箱。慣性モーメントは、直方体の式 $I = m(a^2 + b^2)/12$ に $m = 1$、 $a = b = 0.2$ を入れた値（約0.00667 kg·m²）。
- `ball`: 半径0.1 mの球。質量と慣性モーメントを書く代わりに、`<inertial auto="true">` と密度（500 kg/m³。水の半分）を書き、衝突の形から自動で計算させている（2-3節）。

文法を確かめてから、Gazeboで開く。

```bash
gz sdf -k drop.sdf
```

**期待する結果**:

```text
Valid.
```

`Valid.` と出れば、SDFの文法として正しい。要素の名前の綴りや入れ子を間違えると、どこが違うかが表示される。

```bash
# T1
cd ~/gz_practice

gz sim drop.sdf
```

**期待する結果**:

Gazeboの画面が開き、灰色の地面の上に、青い箱と赤い球が浮いて見える。`gz sim` は一時停止した状態で始まるので、まだ落ちない。左下の▶を押すと、2つは同時に落ち始め、約0.6秒で地面に着いて止まる。止まったら、ウィンドウを閉じる（またはT1で `Ctrl+C`）。次の手順は落ちる前の状態から始めるので、もう一度 `gz sim drop.sdf` で開き直す。

画面の再生ボタンの代わりに、決めた刻みの数だけ進めることもできる。別のターミナルから、ワールドを操作するサービスに「300刻み（0.3秒）進めて、また止める」を送り、箱の位置を読む。

```bash
# T2
gz service -s /world/drop/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 300, pause: true'

gz model -m box -p
```

**期待する結果**:

```text
$ gz service -s /world/drop/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 300, pause: true'
data: true

$ gz model -m box -p

Requesting state for world [drop]...

Model: [8]
  - Name: box
  - Pose [ XYZ (m) ] [ RPY (rad) ]:
    [0.000000 0.000000 1.557530]
    [0.000000 -0.000000 0.000000]
```

`data: true` は、サービスが要求を受け付けたという応答。`Pose` の1行目が位置（x, y, z）、2行目が向き（ロール, ピッチ, ヨー）で、箱の中心の高さ $z$ が2 mから1.5575 mに下がった。`Model: [8]` の番号は、Gazeboの中でモデルに付けられた番号で、ファイルの書き方によって変わる。画面では、箱と球が少しだけ下がって止まって見える。

`multi_step` も、`<real_time_factor>` に従って現実の時間と同じ速さで進む。300刻みなら0.3秒かかるので、`gz model` は、画面の動きが止まってから実行する。同じように、さらに300刻み、400刻みと進めて読むと、高さは次のように変わる（球も、箱と同じ高さになる）。

| 進めた時間 | 箱の中心の高さ（Gazebo） | 自由落下の式 $z = 2 - \frac{1}{2} g t^2$ |
|---|---|---|
| 0秒 | 2.000000 m | 2.000 m |
| 0.3秒 | 1.557530 m | 1.559 m |
| 0.6秒 | 0.233060 m | 0.236 m |
| 1.0秒 | 0.095492 m | （地面に着いている） |

**読み方**: 

- 0.3秒・0.6秒の高さは、自由落下の式と、1〜3 mmの差で合う。差は、物理エンジンが連続した時間でなく、0.001秒の刻みで計算していることから来る。Gazeboの既定の物理エンジン（DART）は、刻みごとに、まず重力で速度を更新し、その新しい速度で位置を進める（半陰的オイラー法。考え方は[Tips集の9-5節](tips.md#9-5-力を先に更新する形が好まれる場面と精度を上げる方法)）。この計算で $n$ 刻み進めた高さは $z = 2 - g \Delta t^2 \frac{n(n+1)}{2}$ になり、 $n = 300$、 $\Delta t = 0.001$ を入れると1.557530 m、 $n = 600$ では0.233060 mと、表のGazeboの値と小数第6位まで一致する。
- 1.0秒の時点で、箱の中心は0.1 m（箱の半分の高さ）でなく0.0955 mにあり、地面に約4.5 mmめり込んでいる。物理エンジンは、接触した物どうしが少し重なることを許し、重なりを刻みごとに少しずつ押し戻す（作成時に3秒まで進めると、0.0962 mまで戻った）。重なりを一気に押し戻すと、跳ね飛ぶような大きな力が出てしまうためである。ロボットの足が地面に数ミリ沈んで見えるのも、同じ理由による。

### 2-3. 慣性を自動で計算させる

`ball` のように `<inertial auto="true">` と書くと、Gazeboは衝突の形と密度から、質量と慣性モーメントを計算する。計算した値は、`gz sdf -p` に `--expand-auto-inertials` を付けると表示できる（`-p` はファイルを読み込んで、解釈した結果のSDFを表示するオプション）。

```bash
gz sdf -p --expand-auto-inertials drop.sdf
```

**期待する結果**（`ball` の部分だけを抜粋）:

```text
    <model name='ball'>
      <pose>1 0 2 0 0 0</pose>
      <link name='link'>
        <inertial auto='true'>
          <density>500</density>
          <pose>0 0 0 0 0 0</pose>
          <mass>2.0943951023931957</mass>
          <inertia>
            <ixx>0.0083775804095727845</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>0.0083775804095727845</iyy>
            <iyz>0</iyz>
            <izz>0.0083775804095727845</izz>
          </inertia>
        </inertial>
```

質量は、球の体積 $\frac{4}{3}\pi r^3$ に密度をかけた $500 \times \frac{4}{3}\pi \times 0.1^3 \approx 2.094$ kg、慣性モーメントは球の式 $\frac{2}{5} m r^2 \approx 0.00838$ kg·m² で、どちらも手計算と合う。

質量と慣性モーメントは、物理の計算の土台である。慣性モーメントを実際より小さく書くと、少しの力で激しく回転し、シミュレーションが不安定になる（よくある失敗）。単純な形なら自動の計算に任せるか、公式で計算した値を書く。メッシュの形の部品では、CADなどで求めた値を書く。

## 3. 物理でできること

### 3-1. 物理エンジンと刻み幅

`<physics>` の主な設定は、次のとおり（既定値は、`gz sdf -d` で表示されるSDFの仕様で確かめた）。

| 要素 | 既定値 | 意味 |
|---|---|---|
| `<max_step_size>` | 0.001 | 1刻みで進めるシミュレーションの時間（秒）。小さいほど正確だが、計算が増える |
| `<real_time_factor>` | 1 | 現実の時間に対する、シミュレーションの時間の進む速さの目標。1なら現実と同じ速さ |
| `<real_time_update_rate>` | 1000 | 現実の1秒あたりに計算する刻みの数の目標。0にすると、できる限り速く進める |

画面の右下には、実際に出ている速さ（RTF、Real Time Factor）が表示される。計算が重くて目標に追いつかないと、RTFは1より小さくなる（フェーズ5-0の2-1節）。フェーズ7-3では、学習を速く進めるため、起動の引数で刻みの数の目標を大きくして、現実より速く進めた。

物理エンジンは、`gz sim` の `--physics-engine` で選べる。ヘルプの表示（`gz sim --help`）のとおり、既定はDART（`gz-physics-dartsim-plugin`）で、Jazzyの導入では、ほかにBullet（`gz-physics-bullet-featherstone-plugin` 等）と、衝突の判定を簡単にして速くしたTPEが入っている。エンジンによって、使える機能（関節の種類、摩擦の扱い等）と結果が少しずつ違う。この章の数値は、すべてDARTのものである。

### 3-2. 体験: 摩擦の違う坂

20度の坂に、摩擦係数の違う2つの箱を置く。

ファイル: `gz_practice/slope.sdf`

<!-- file: gz_practice/slope.sdf -->
```xml
<?xml version="1.0" ?>
<!-- 20度の坂に、摩擦係数の違う2つの箱を置くワールド -->
<sdf version="1.11">
  <world name="slope">
    <plugin filename="gz-sim-physics-system" name="gz::sim::systems::Physics"/>
    <plugin filename="gz-sim-user-commands-system" name="gz::sim::systems::UserCommands"/>
    <plugin filename="gz-sim-scene-broadcaster-system" name="gz::sim::systems::SceneBroadcaster"/>

    <light type="directional" name="sun">
      <pose>0 0 10 0 0 0</pose>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <direction>-0.5 0.1 -0.9</direction>
    </light>

    <model name="ground">
      <static>true</static>
      <link name="link">
        <collision name="collision">
          <geometry><plane><normal>0 0 1</normal><size>20 20</size></plane></geometry>
        </collision>
        <visual name="visual">
          <geometry><plane><normal>0 0 1</normal><size>20 20</size></plane></geometry>
          <material><diffuse>0.7 0.7 0.7 1</diffuse></material>
        </visual>
      </link>
    </model>

    <!-- 坂: 長さ4 mの板を、y軸の周りに20度（0.349066 rad）傾けて固定する -->
    <model name="slope">
      <static>true</static>
      <pose>0 0 1 0 0.349066 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry><box><size>4 2 0.1</size></box></geometry>
        </collision>
        <visual name="visual">
          <geometry><box><size>4 2 0.1</size></box></geometry>
          <material><diffuse>0.6 0.5 0.4 1</diffuse></material>
        </visual>
      </link>
    </model>

    <!-- 滑りやすい箱（摩擦係数0.2）。位置は坂の座標系で、板の上面に載せる -->
    <model name="box_slippery">
      <pose relative_to="slope">-1 -0.5 0.15 0 0 0</pose>
      <link name="link">
        <inertial>
          <mass>1.0</mass>
          <inertia>
            <ixx>0.00667</ixx><iyy>0.00667</iyy><izz>0.00667</izz>
            <ixy>0</ixy><ixz>0</ixz><iyz>0</iyz>
          </inertia>
        </inertial>
        <collision name="collision">
          <geometry><box><size>0.2 0.2 0.2</size></box></geometry>
          <surface>
            <friction><ode><mu>0.2</mu><mu2>0.2</mu2></ode></friction>
          </surface>
        </collision>
        <visual name="visual">
          <geometry><box><size>0.2 0.2 0.2</size></box></geometry>
          <material><diffuse>0.2 0.6 0.9 1</diffuse></material>
        </visual>
      </link>
    </model>

    <!-- 滑りにくい箱（摩擦係数0.8） -->
    <model name="box_grippy">
      <pose relative_to="slope">-1 0.5 0.15 0 0 0</pose>
      <link name="link">
        <inertial>
          <mass>1.0</mass>
          <inertia>
            <ixx>0.00667</ixx><iyy>0.00667</iyy><izz>0.00667</izz>
            <ixy>0</ixy><ixz>0</ixz><iyz>0</iyz>
          </inertia>
        </inertial>
        <collision name="collision">
          <geometry><box><size>0.2 0.2 0.2</size></box></geometry>
          <surface>
            <friction><ode><mu>0.8</mu><mu2>0.8</mu2></ode></friction>
          </surface>
        </collision>
        <visual name="visual">
          <geometry><box><size>0.2 0.2 0.2</size></box></geometry>
          <material><diffuse>0.9 0.6 0.2 1</diffuse></material>
        </visual>
      </link>
    </model>
  </world>
</sdf>
```

drop.sdfとの違いを読む。

- `<physics>` と `<gravity>` を書いていない。どちらも3-1節の表と2-2節の既定値（刻み0.001秒、重力9.8 m/s²）になる。
- 坂は、厚さ0.1 mの板を、`<pose>` のピッチ（y軸の周りの回転）で0.349066 rad（20度）傾けたもの。y軸の周りに正の向きに回すと、x軸の正の側が下がるので、x軸の正の向きが下り坂になる。
- 箱の `<pose relative_to="slope">` は、坂のモデルの座標系から見た位置である。坂の座標系では、板の上面が $z = 0.05$ にあるので、一辺0.2 mの箱の中心を $z = 0.15$ に置けば、ちょうど上面に載る。傾いた坂の上の座標を手で計算しなくて済み、坂の角度を変えても箱が一緒に付いてくる。
- 摩擦係数は、衝突の形の `<surface>` の `<friction>` に書く。`<ode>` は、Gazeboの昔の物理エンジン（ODE）の名残の名前だが、DARTでもこの値が使われる。`<mu>` と `<mu2>` は、接触面の中の直交する2つの向きの摩擦係数で、ふつうは同じ値にする。書かなければ、どちらも1である。

坂の上の箱が滑り出すかどうかは、斜面の角度 $\theta$ の正接 $\tan\theta$ と、摩擦係数 $\mu$ の比べっこで決まる。 $\tan 20^\circ \approx 0.364$ なので、 $\mu = 0.2$ の箱は滑り、 $\mu = 0.8$ の箱は止まったままのはずである。滑る箱の、斜面に沿った加速度は $a = g(\sin\theta - \mu\cos\theta) = 9.8 \times (0.342 - 0.2 \times 0.940) \approx 1.51$ m/s² で、1秒で $\frac{1}{2} a t^2 \approx 0.755$ m 滑る。

```bash
# T1
cd ~/gz_practice

gz sim slope.sdf
```

```bash
# T2
gz service -s /world/slope/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 1000, pause: true'

gz model -m box_slippery -p

gz model -m box_grippy -p
```

**期待する結果**（T2の表示。進める前の位置は、どちらも x が−0.888390、z が1.482970）:

```text
$ gz service -s /world/slope/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 1000, pause: true'
data: true

$ gz model -m box_slippery -p

Requesting state for world [slope]...

Model: [12]
  - Name: box_slippery
  - Pose [ XYZ (m) ] [ RPY (rad) ]:
    [-0.178212 -0.500000 1.224490]
    [-0.000000 0.349066 0.000000]

$ gz model -m box_grippy -p

Requesting state for world [slope]...

Model: [16]
  - Name: box_grippy
  - Pose [ XYZ (m) ] [ RPY (rad) ]:
    [-0.888390 0.500000 1.482970]
    [-0.000000 0.349066 -0.000000]
```

**読み方**: 画面では、1秒の間に、青い箱だけが坂を滑り下り、橙の箱はそのまま止まっている。青い箱の x は−0.888390から−0.178212へ0.710178 m進んだ。斜面に沿った距離に直すと $0.710178 / \cos 20^\circ \approx 0.756$ m で、手計算の0.755 mと合う。ピッチは0.349066 radのままなので、転がらずに、坂と平行なまま滑っている。この後も進めると、青い箱は坂の下の端から地面に落ちる。モデルの番号（`[12]`・`[16]`）は、ファイルの書き方によって変わる。

**2つの面の摩擦係数の組み合わせ方**: 接触では、坂の面と箱の面の2つの摩擦係数がある。この坂は摩擦係数を書いていないので1で、箱の0.2と組み合わせた結果が0.2になった。DARTでは、2つのうち**小さいほう**が使われる（作成時に、坂の摩擦係数を0.5にして確かめた。2つの積が使われるなら0.1になり、滑る距離が変わるはずだが、結果は同じだった）。物理エンジンによって組み合わせ方が違うので、別のエンジンを使うときは、同じように確かめる。

### 3-3. 体験: 反発

反発係数の違う2つの球を、跳ね返す地面に落とす。

ファイル: `gz_practice/bounce.sdf`

<!-- file: gz_practice/bounce.sdf -->
```xml
<?xml version="1.0" ?>
<!-- 反発係数の違う2つの球を、跳ね返す地面に落とすワールド -->
<sdf version="1.11">
  <world name="bounce">
    <plugin filename="gz-sim-physics-system" name="gz::sim::systems::Physics"/>
    <plugin filename="gz-sim-user-commands-system" name="gz::sim::systems::UserCommands"/>
    <plugin filename="gz-sim-scene-broadcaster-system" name="gz::sim::systems::SceneBroadcaster"/>

    <light type="directional" name="sun">
      <pose>0 0 10 0 0 0</pose>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <direction>-0.5 0.1 -0.9</direction>
    </light>

    <!-- 地面の反発係数を1にする（球の値との積が使われる） -->
    <model name="ground">
      <static>true</static>
      <link name="link">
        <collision name="collision">
          <geometry><plane><normal>0 0 1</normal><size>20 20</size></plane></geometry>
          <surface>
            <bounce><restitution_coefficient>1.0</restitution_coefficient></bounce>
          </surface>
        </collision>
        <visual name="visual">
          <geometry><plane><normal>0 0 1</normal><size>20 20</size></plane></geometry>
          <material><diffuse>0.7 0.7 0.7 1</diffuse></material>
        </visual>
      </link>
    </model>

    <!-- よく跳ねる球（反発係数0.8） -->
    <model name="ball_bouncy">
      <pose>0 0 2 0 0 0</pose>
      <link name="link">
        <inertial auto="true"><density>500</density></inertial>
        <collision name="collision">
          <geometry><sphere><radius>0.1</radius></sphere></geometry>
          <surface>
            <bounce><restitution_coefficient>0.8</restitution_coefficient></bounce>
          </surface>
        </collision>
        <visual name="visual">
          <geometry><sphere><radius>0.1</radius></sphere></geometry>
          <material><diffuse>0.9 0.3 0.2 1</diffuse></material>
        </visual>
      </link>
    </model>

    <!-- あまり跳ねない球（反発係数0.3） -->
    <model name="ball_dull">
      <pose>1 0 2 0 0 0</pose>
      <link name="link">
        <inertial auto="true"><density>500</density></inertial>
        <collision name="collision">
          <geometry><sphere><radius>0.1</radius></sphere></geometry>
          <surface>
            <bounce><restitution_coefficient>0.3</restitution_coefficient></bounce>
          </surface>
        </collision>
        <visual name="visual">
          <geometry><sphere><radius>0.1</radius></sphere></geometry>
          <material><diffuse>0.3 0.3 0.8 1</diffuse></material>
        </visual>
      </link>
    </model>
  </world>
</sdf>
```

反発係数 $e$（`<restitution_coefficient>`、0〜1）は、ぶつかった直後の速さが、ぶつかる直前の何倍になるかを表す。既定は0（まったく跳ねない）。跳ね上がる高さは速さの2乗に比例するので、落ちた高さの $e^2$ 倍になる。球の底は1.9 mの高さから落ちるので、 $e = 0.8$ なら $0.64 \times 1.9 \approx 1.22$ m、 $e = 0.3$ なら $0.09 \times 1.9 \approx 0.17$ m まで跳ね上がるはずである。

```bash
# T1
cd ~/gz_practice

gz sim bounce.sdf
```

```bash
# T2
gz service -s /world/bounce/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 1100, pause: true'

gz model -m ball_bouncy -p
```

**期待する結果**:

```text
$ gz service -s /world/bounce/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 1100, pause: true'
data: true

$ gz model -m ball_bouncy -p

Requesting state for world [bounce]...

Model: [8]
  - Name: ball_bouncy
  - Pose [ XYZ (m) ] [ RPY (rad) ]:
    [0.000000 0.000000 1.316000]
    [0.000000 -0.000000 0.000000]
```

**読み方**: 画面では、2つの球が同時に落ち、約0.6秒で地面に当たる。赤い球は高く跳ね上がり、青い球は少しだけ跳ねてすぐに落ち着く。1.1秒の時点（跳ね返りの頂点の近く）で、赤い球の中心は1.316 m、底は半径0.1 mを引いた1.216 mで、手計算の1.22 mと合う。作成時に0.1秒ごとに読んだところ、青い球の中心は0.8秒の時点で最も高い0.267 m（底は0.167 m）で、手計算の0.17 mと合った。赤い球はこの後も跳ね続け、2回目は底が $0.64 \times 1.216 \approx 0.78$ m の高さまで（作成時の値では、2.0〜2.1秒に中心が約0.88 m）と、跳ねるたびに低くなる。

**2つの面の反発係数の組み合わせ方**: DARTでは、2つの面の反発係数の**積**が使われる。そのため、このワールドでは地面の反発係数を1にしている。作成時に、地面の反発係数を書かずに（既定の0で）同じ球を落としたところ、 $0.8 \times 0 = 0$ で、まったく跳ねなかった。摩擦（3-2節。小さいほう）と組み合わせ方が違うので、跳ねさせたいのに跳ねないときは、相手の面の値を確かめる。

### 3-4. そのほかにできること

物理のふるまいは、ここまでの要素のほかにも、関節の設定と、システム（プラグイン）で足せる。Gazeboに付属する例のワールド（`/opt/ros/jazzy/opt/gz_sim_vendor/share/gz/gz-sim8/worlds/`）に、それぞれの書き方の手本がある。

| できること | 書く場所 | 例のワールド |
|---|---|---|
| 関節の動ける範囲・力と速さの上限 | 関節の `<axis>` の `<limit>` | この章の4-3節 |
| 関節の減衰（速さに比例する抵抗）・摩擦・ばね | 関節の `<axis>` の `<dynamics>`（4節で使う。フェーズ7-3でも使う） | `joint_position_controller.sdf` |
| 関節に力（トルク）をかける | システム `ApplyJointForce`（フェーズ5-4で使った） | `apply_joint_force.sdf` |
| リンクに力をかける | システム `ApplyLinkWrench` | `apply_link_wrench.sdf` |
| 水の浮力 | システム `Buoyancy` | `buoyancy.sdf` |
| 翼の揚力と抗力 | システム `LiftDrag` | `lift_drag.sdf` |
| 風 | ワールドの `<wind>` とシステム `WindEffects` | `wind.sdf` |
| 接触の検出（何と何が触れたか） | 接触センサとシステム `Contact` | `contact_sensor.sdf` |

例のワールドは、`gz sim apply_link_wrench.sdf` のように、ファイル名だけで開ける（Gazeboが付属のフォルダから探す）。中身を読むときは、上のフォルダのファイルを開く。作成時には、この表の例のワールドが付属のフォルダにあることまでを確かめた（表の機能は、関節の制限・減衰と `ApplyJointForce` のほかは、この教材では動かしていない）。

## 4. URDFとxacroでロボットを書く

### 4-1. URDFとSDFの違い

ROS2の世界では、ロボットの形と関節を、SDFでなくURDFで書くことが多い。URDFは、ROS（ROS1）の時代からの、ロボット1台を記述するXMLの書き方である。

| | SDF | URDF |
|---|---|---|
| 記述できる範囲 | ワールド全体（光・物理の設定・複数のモデル） | ロボット1台だけ |
| リンクのつながり | 自由（輪になったつながりも書ける） | 木の形だけ（親は1つ） |
| 使う道具 | Gazebo | ROS2の道具（`robot_state_publisher`、RViz、MoveIt、ros2_control）と、Gazebo（読み込むときにSDFに変換する） |
| Gazebo特有の設定（システム・摩擦など） | そのまま書ける | `<gazebo>` の中に書く（変換のときにSDFに写される） |

URDFは、ROS2の道具の多くが読める共通の形式なので、ロボットの記述は1つのURDFに書き、Gazeboにも、ROS2のノードにも同じものを渡す、という使い方が一般的である。要素の名前は、SDFとよく似ている（`<link>`・`<joint>`・`<inertial>`・`<collision>`・`<visual>`）。違いは、位置を `<pose>` でなく `<origin xyz="..." rpy="..."/>` の属性で書くこと、関節の位置が子のリンクの座標系の原点になること、などである。

### 4-2. xacroで短く書く

URDFには、変数や繰り返しの仕組みが無い。同じ形の腕を2本書くと、ほぼ同じ数十行が2回並び、長さを変えるときに、何か所も直すことになる。xacro（XML Macros）は、URDFに次の仕組みを足した書き方で、`xacro` コマンドで普通のURDFに展開してから使う。

- `<xacro:property name="arm_len" value="0.5"/>`: 名前を付けた値（変数）。`${arm_len}` で使う。
- `${arm_mass*arm_r*arm_r/2}`: 式。値を計算して埋め込む（`pi` も使える）。
- `<xacro:macro name="arm_link" params="name">`: マクロ（部品のひな形）。`<xacro:arm_link name="link1"/>` と書くと、中身が展開される。

### 4-3. 体験: 2関節のアームを書く

台（高さ1 mの柱）の上に、長さ0.5 mの腕を2本つないだアームを書く。腕は、x軸の向きに水平に伸びた姿勢で現れ、重力で振り下ろされる。

ファイル: `gz_practice/arm.urdf.xacro`

<!-- file: gz_practice/arm.urdf.xacro -->
```xml
<?xml version="1.0"?>
<!-- 台に取り付けた2関節のアーム。腕は水平に伸びた姿勢で現れ、重力で振り下ろされる -->
<robot name="arm" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <xacro:property name="arm_len" value="0.5"/>
  <xacro:property name="arm_r" value="0.03"/>
  <xacro:property name="arm_mass" value="1.0"/>

  <!-- x軸の向きに伸びる円柱の腕（質量・慣性・衝突・見た目をまとめて書く） -->
  <xacro:macro name="arm_link" params="name">
    <link name="${name}">
      <inertial>
        <origin xyz="${arm_len/2} 0 0"/>
        <mass value="${arm_mass}"/>
        <inertia ixx="${arm_mass*arm_r*arm_r/2}" ixy="0" ixz="0"
                 iyy="${arm_mass*(3*arm_r*arm_r + arm_len*arm_len)/12}" iyz="0"
                 izz="${arm_mass*(3*arm_r*arm_r + arm_len*arm_len)/12}"/>
      </inertial>
      <collision>
        <origin xyz="${arm_len/2} 0 0" rpy="0 ${pi/2} 0"/>
        <geometry><cylinder radius="${arm_r}" length="${arm_len}"/></geometry>
      </collision>
      <visual>
        <origin xyz="${arm_len/2} 0 0" rpy="0 ${pi/2} 0"/>
        <geometry><cylinder radius="${arm_r}" length="${arm_len}"/></geometry>
      </visual>
    </link>
  </xacro:macro>

  <link name="world"/>

  <link name="base">
    <inertial>
      <origin xyz="0 0 0.5"/>
      <mass value="5.0"/>
      <inertia ixx="${5.0*(0.1*0.1 + 1.0*1.0)/12}" ixy="0" ixz="0"
               iyy="${5.0*(0.1*0.1 + 1.0*1.0)/12}" iyz="0"
               izz="${5.0*(0.1*0.1 + 0.1*0.1)/12}"/>
    </inertial>
    <collision>
      <origin xyz="0 0 0.5"/>
      <geometry><box size="0.1 0.1 1.0"/></geometry>
    </collision>
    <visual>
      <origin xyz="0 0 0.5"/>
      <geometry><box size="0.1 0.1 1.0"/></geometry>
    </visual>
  </link>

  <joint name="fix_base" type="fixed">
    <parent link="world"/>
    <child link="base"/>
  </joint>

  <xacro:arm_link name="link1"/>
  <xacro:arm_link name="link2"/>

  <joint name="joint1" type="revolute">
    <parent link="base"/>
    <child link="link1"/>
    <origin xyz="0 0.1 1.0"/>
    <axis xyz="0 1 0"/>
    <limit lower="-1.0" upper="1.0" effort="100" velocity="10"/>
    <dynamics damping="0.5"/>
  </joint>

  <joint name="joint2" type="revolute">
    <parent link="link1"/>
    <child link="link2"/>
    <origin xyz="${arm_len} 0.07 0"/>
    <axis xyz="0 1 0"/>
    <limit lower="-2.5" upper="2.5" effort="100" velocity="10"/>
    <dynamics damping="0.5"/>
  </joint>

  <!-- Gazeboで関節の角度を送るシステム（ROS2へはブリッジで中継する） -->
  <gazebo>
    <plugin filename="gz-sim-joint-state-publisher-system"
            name="gz::sim::systems::JointStatePublisher"/>
  </gazebo>
</robot>
```

ファイルの中身を読む。

- **腕のマクロ `arm_link`**: 質量1 kg、半径0.03 m、長さ0.5 mの円柱を、リンクの原点からx軸の向きに伸ばす。円柱は、そのままではz軸の向きに伸びるので、`rpy="0 ${pi/2} 0"`（y軸の周りに90度）で横に倒し、中心を `${arm_len/2}`（0.25 m）だけずらす。慣性モーメントは円柱の式で、軸の周りが $m r^2 / 2$、軸に直交する向きが $m(3r^2 + L^2)/12$ である。このマクロを `link1`・`link2` の2回使う。
- **`world` と `fix_base`**: `world` という名前の、中身の無いリンクに、固定の関節（`type="fixed"`）で台をつなぐと、台はワールドに固定される（Gazeboに読み込むとき、`world` は特別な名前として、ワールドそのものを指す）。これが無いと、台ごと倒れる。
- **関節 `joint1`・`joint2`**: `type="revolute"` は、動ける範囲のある回転の関節。`<origin>` が関節の位置（親のリンクの座標系で）で、子のリンクの原点もここになる。`joint1` は柱の上端、`joint2` は `link1` の先端（x = 0.5 m）にある。y の値（0.1・0.07）は、腕どうしと柱が見た目で重ならないように、横にずらしたもの。回る軸はどちらもy軸（`<axis xyz="0 1 0"/>`）で、角度が正の向きに増えると、腕の先が下がる。
- **`<limit>`**: `joint1` は−1.0〜1.0 rad（約±57度）しか回らない。`effort` と `velocity` は、関節に出せる力と速さの上限（URDFの回転の関節では、書くことが必須）。
- **`<dynamics damping="0.5">`**: 関節の減衰。角速度に比例する抵抗のトルク（N·m·s/rad）で、これが無いと、腕はいつまでも揺れ続ける。
- **`<gazebo>`**: Gazeboにだけ伝えたい設定。ここでは、関節の角度をGazeboのトピックに送るシステム `JointStatePublisher` を足す。`reference` 属性を付けずに書くと、モデル全体に付く。

腕が水平な姿勢（角度0）から放すと、`joint1` は重力で正の向きに回り、上限の1.0 radで止まるはずである。`link2` は、`joint1` が止まった後も回り続け、真下に垂れて落ち着くはずである。`link1` が水平から1.0 rad下がっているので、`link2` が真下（水平から $\pi/2 \approx 1.571$ rad）を向くときの `joint2` の角度は、 $\pi/2 - 1.0 \approx 0.571$ rad になる。

xacroをURDFに展開し、URDFとして正しいかを `check_urdf` で確かめる。`xacro` と `check_urdf` は、それぞれ `ros-jazzy-xacro`・`ros-jazzy-urdfdom` のパッケージに含まれ、フェーズ5-0の1-2節のGazeboの導入で、依存するパッケージとして一緒に入る（作成時の環境で確かめた。見つからない場合は、`sudo apt install` でこの2つを入れる）。

```bash
cd ~/gz_practice

xacro arm.urdf.xacro > arm.urdf

check_urdf arm.urdf
```

**期待する結果**（`check_urdf` の分）:

```text
robot name is: arm
---------- Successfully Parsed XML ---------------
root Link: world has 1 child(ren)
    child(1):  base
        child(1):  link1
            child(1):  link2
```

`Successfully Parsed XML` と、リンクの木の形（`world` の子が `base`、その子が `link1`、その子が `link2`）が出れば、URDFとして正しい。`arm.urdf` を開くと、`${...}` の式が計算された値（例: `link1` の `iyy` は0.02105833333333333）に置き換わり、マクロが2本の腕に展開されているのが分かる。

Gazeboが読み込むときのSDFへの変換は、`gz sdf -p` で見られる。

```bash
gz sdf -p arm.urdf
```

**期待する結果**（`joint1` の部分だけを抜粋）:

```text
    <joint name='joint1' type='revolute'>
      <pose relative_to='base'>0 0.10000000000000001 1 0 0 0</pose>
      <parent>base</parent>
      <child>link1</child>
      <axis>
        <xyz>0 1 0</xyz>
        <limit>
          <lower>-1</lower>
          <upper>1</upper>
          <effort>100</effort>
          <velocity>10</velocity>
        </limit>
        <dynamics>
          <damping>0.5</damping>
          <friction>0</friction>
          <spring_reference>0</spring_reference>
          <spring_stiffness>0</spring_stiffness>
        </dynamics>
      </axis>
    </joint>
```

URDFの `<origin>` が、SDFの `<pose relative_to='base'>` に変わっている。ほかに、`fix_base` は親を `world` とする固定の関節になり、`<gazebo>` の中の `<plugin>` は、モデルの直下に写されている（この抜粋の外）。

### 4-4. 体験: Gazeboに出して、関節の角度をROS2で読む

アームを、Gazeboの空のワールドに出し、関節の角度をROS2のトピックで読む。4つのターミナルを使う。

```bash
# T1
gz sim -r empty.sdf
```

`empty.sdf` は、Gazeboに付属する、地面と光だけのワールド（ワールド名は `empty`）。`-r` を付けると、一時停止せずに、すぐに時間が進み始める。

```bash
# T2
cd ~/gz_practice

ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(xacro arm.urdf.xacro)"
```

`robot_state_publisher` は、パラメータ `robot_description` で受け取ったURDFを、同じ名前のトピック `/robot_description`（型は [`std_msgs/msg/String`](https://github.com/ros2/common_interfaces/blob/jazzy/std_msgs/msg/String.msg)）に送っておくノードである。このトピックは、後から購読したノードにも最後の値が届く設定（QoSの `transient_local`）で送られるので、T4の `create` を後から起動しても受け取れる。本来の役目は、関節の角度（`/joint_states`）を受け取って、各リンクの位置関係（TF）を計算して配ることで、RVizでロボットを表示するときなどに使う。`$(xacro arm.urdf.xacro)` は、xacroを展開した結果の文字列を、その場でパラメータに渡すシェルの書き方である。

```bash
# T3
ros2 run ros_gz_bridge parameter_bridge /world/empty/model/arm/joint_state@sensor_msgs/msg/JointState[gz.msgs.Model --ros-args -r /world/empty/model/arm/joint_state:=/joint_states
```

`JointStatePublisher` が送るGazeboのトピック `/world/<ワールド名>/model/<モデル名>/joint_state` を、ROS2の [`sensor_msgs/msg/JointState`](https://github.com/ros2/common_interfaces/blob/jazzy/sensor_msgs/msg/JointState.msg) に変換して中継する。`[` は、GazeboからROS2への一方向の中継を表す（フェーズ5-0の4-1節）。`-r` で、ROS2の側のトピック名を、ROSの慣習の名前 `/joint_states` に付け替えている。

```bash
# T4
ros2 run ros_gz_sim create -topic robot_description -name arm
```

**期待する結果**（T2〜T4の表示。先頭の数字は時刻で、実行ごとに異なる）:

```text
# T2（robot_state_publisher）
[INFO] [1791773676.514197905] [robot_state_publisher]: Robot initialized
# T3（parameter_bridge）
[INFO] [1791773676.505908554] [ros_gz_bridge]: Creating GZ->ROS Bridge: [/world/empty/model/arm/joint_state (gz.msgs.Model) -> /world/empty/model/arm/joint_state (sensor_msgs/msg/JointState)] (Lazy 0)
# T4（create）
[INFO] [1791773679.361088633] [ros_gz_sim]: Requesting list of world names.
[INFO] [1791773679.565705610] [ros_gz_sim]: Waiting messages on topic [robot_description].
[INFO] [1791773680.242149835] [ros_gz_sim]: Entity creation successful.
```

T1では、地面だけのGazeboの画面が開き、時間が進んでいる。T2とT3は、起動したときの1行だけを表示し、その後は何も出さずに動き続ける。T3の行に出るROS2の側のトピック名は、付け替える前の名前のままだが、実際には `/joint_states` に送られている。

`create` は、トピック `/robot_description` からURDFを受け取り、動いているワールドにモデル `arm` として出して、終わる。`Entity creation successful.` が出れば成功である。T1の画面では、柱の上に水平な腕が現れ、すぐに振り下ろされる。`link1` は斜め下（約57度）で止まり、`link2` は振り子のように何度か揺れて、真下に垂れて落ち着く。

腕が落ち着いたら（10秒ほど待つ）、関節の角度を読む。

```bash
# T4
ros2 topic echo /joint_states --once
```

**期待する結果**（`stamp` はシミュレーションの時刻で、読んだ時刻によって変わる）:

```text
header:
  stamp:
    sec: 15
    nanosec: 0
  frame_id: ''
name:
- fix_base
- joint1
- joint2
position:
- 0.0
- 1.0000000000000195
- 0.5797281298591915
velocity:
- 0.0
- -1.0198786259962844e-13
- -6.5156213757688874e-15
effort:
- 0.0
- 0.0
- 0.0
---
```

**読み方**:

- `name` と `position` は、同じ順番で対応している。固定の関節 `fix_base` も並ぶが、角度は常に0である。
- `joint1` は1.0 radで、`<limit>` の上限で止まっている。
- `joint2` は約0.580 radで、手計算の0.571 radより、約0.009 rad（約0.5度）大きい。`velocity` がほぼ0なので、揺れている途中ではなく、この角度で止まっている。このずれは、刻みごとの計算から来るものである。作成時に、関節の減衰を4倍（2.0）にすると、ずれも約4倍（0.034 rad）になり、刻み幅を半分（0.0005秒）にすると、ずれも半分（0.0045 rad）になった。連続の時間の物理では、減衰は動いているときだけ効くので、止まった位置を変えないはずである。`joint1` を上限に押し付けたまま、刻みごとに計算するうちに、減衰の力が、わずかに残る形になっていると考えられる（DARTの内部の計算までは確かめていない）。シミュレーションの結果は、刻み幅や物理エンジンの設定で少し変わるので、実機と比べるときは、どのくらいずれうるかを、刻み幅を変えて確かめておく。

止めるときは、T3・T2・T1の順に `Ctrl+C` で止める（T4の `create` と `ros2 topic echo --once` は、自分で終わる）。

## 5. ros2_controlで関節を動かす

### 5-1. ros2_controlの仕組み

4-4節では、関節の角度を読むだけで、腕は重力に任せて垂れ下がった。腕を狙った角度に動かすには、関節に指令を送る仕組みが要る。3-4節の表のように、Gazeboのシステム（`ApplyJointForce` 等）で関節に力をかけることもできるが、それはGazeboの中だけの書き方で、実機のロボットに移すときには、指令を送る側を書き直すことになる。

ros2_controlは、この「指令を送る側」と「関節を実際に動かす側」を分けて、間をそろえた形でつなぐ、ROS2の仕組みである。関節を動かす側は、実機ならモータのドライバを読み書きする部品、シミュレーションならGazeboの関節を読み書きする部品に替わるが、指令を送る側（コントローラ）と、さらにその上のノードは、どちらでも同じものを使える。腕の動作計画のMoveItは、計画した動きをros2_controlの軌道のコントローラ（5-4節で使う）に送って関節を動かすのが標準の構成で、移動ロボットでも、車輪をros2_controlのコントローラで動かす構成が多い。

ros2_controlは、次の部品からできている。

| 部品 | 役目 | この節で使うもの |
|---|---|---|
| ハードウェアの部品（hardware component） | 関節の状態を読み、指令を書き込む。実機とシミュレーションで差し替える部分 | `gz_ros2_control/GazeboSimSystem`（Gazeboの関節を読み書きする） |
| インターフェース | 関節ごとに、読める値（状態のインターフェース。角度 `position`・角速度 `velocity` 等）と、書ける値（指令のインターフェース。角度・角速度・トルク `effort` のどれか）を決める | 指令は角度、状態は角度と角速度 |
| コントローラ | 状態を読み、ROS2のトピックやアクションで受けた目標から、指令を計算して書き込む | `joint_state_broadcaster`・`position_controllers/JointGroupPositionController`・`joint_trajectory_controller/JointTrajectoryController` |
| コントローラマネージャ（controller manager） | 決まった周期（`update_rate`）で、ハードウェアの読み → コントローラの計算 → ハードウェアへの書き、を繰り返す。コントローラの読み込みと有効・無効の切り替えも受け付ける | Gazeboの中で動かす（`gz_ros2_control` のシステムが起動する） |

コントローラのうち `joint_state_broadcaster` は、指令を書かず、全部の関節の状態を `/joint_states` に送るだけのものである。4-4節でブリッジが担っていた役目を、ros2_controlの中で果たす。

![ROS2のノード（ros2 topic pub、ros2 action send_goal）が、トピックやアクションで、コントローラに目標を送る。gz simの中では、コントローラマネージャがupdate_rateごとに、コントローラとjoint_state_broadcasterを動かす。コントローラマネージャは、ハードウェアの部品（GazeboSimSystem）と指令・状態をやり取りし、GazeboSimSystemはGazeboの関節に指令を書き、状態を読む。joint_state_broadcasterは、状態を/joint_statesに送り、ros2 topic echoなどで読める](img/interlude_gazebo_ros2_control.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart LR
    N["ROS2のノード<br/>ros2 topic pub<br/>ros2 action send_goal"] -->|"目標"| C
    subgraph GZ["gz sim の中"]
        subgraph CM["コントローラマネージャ（update_rate ごとに繰り返す）"]
            C["コントローラ<br/>位置のコントローラ<br/>軌道のコントローラ"]
            JSB["joint_state_broadcaster<br/>状態を送る"]
        end
        CM <-->|"指令・状態"| HW["ハードウェアの部品<br/>GazeboSimSystem"]
        HW <-->|"書く・読む"| J["Gazeboの関節<br/>物理エンジンが動きを計算"]
    end
    JSB -->|"/joint_states"| E["ros2 topic echo など"]
```

</details>

コントローラの読み込みと有効・無効の切り替えは、図の外から、コントローラマネージャのサービスを呼んで行う（5-3節の `spawner`）。

### 5-2. 体験: アームにros2_controlの記述を足す

4-3節のアームに、ros2_controlの記述を足す。4-3節のファイルは書き換えずに残し、それを読み込む新しいファイルを作る。

ファイル: `gz_practice/arm_control.urdf.xacro`

<!-- file: gz_practice/arm_control.urdf.xacro -->
```xml
<?xml version="1.0"?>
<!-- 4-3節のアームに、ros2_controlで関節を動かすための記述を足す -->
<robot name="arm" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <xacro:include filename="arm.urdf.xacro"/>

  <!-- ハードウェア（ここではGazebo）と、関節ごとの指令・状態のインターフェース -->
  <ros2_control name="GazeboSimSystem" type="system">
    <hardware>
      <plugin>gz_ros2_control/GazeboSimSystem</plugin>
    </hardware>
    <joint name="joint1">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
    <joint name="joint2">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
  </ros2_control>

  <!-- Gazeboの中でコントローラマネージャを動かすシステムと、その設定のファイル -->
  <gazebo>
    <plugin filename="gz_ros2_control-system"
            name="gz_ros2_control::GazeboSimROS2ControlPlugin">
      <parameters>$(env HOME)/gz_practice/arm_controllers.yaml</parameters>
    </plugin>
  </gazebo>
</robot>
```

- **`<xacro:include>`**: ほかのxacroのファイルを読み込み、その `<robot>` の中身を、ここに展開する。パスは、このファイルからの相対パスで書ける。これで、4-3節の台と2本の腕が、そのまま入る（4-3節の `JointStatePublisher` も入るが、この節では使わない）。
- **`<ros2_control>`**: ros2_controlの記述。`<hardware>` の `<plugin>` で、ハードウェアの部品にGazeboの `GazeboSimSystem` を使うと決め、`<joint>` ごとに、指令のインターフェース（`<command_interface>`）と状態のインターフェース（`<state_interface>`）を並べる。ここでは、2つの関節とも、角度を指令し、角度と角速度を読む。
- **`<gazebo>` の `<plugin>`**: Gazeboのシステム `GazeboSimROS2ControlPlugin` を足す。このシステムが、Gazeboの中でコントローラマネージャを起動し、URDFの `<ros2_control>` に従ってハードウェアの部品を作る。`<parameters>` は、コントローラの設定のファイルの場所である。`$(env HOME)` は、xacroが展開するときに、環境変数 `HOME` の値（ホームのフォルダ）に置き換わる。

> **補足（設定のファイルの場所）**: 一般的には、URDFと設定のファイルをROS2のパッケージに入れ、`$(find パッケージ名)/config/....yaml` のように、パッケージの場所から書く。この章ではパッケージを作らないので、ホームのフォルダからの絶対パスにした。練習のフォルダを `~/gz_practice` 以外に置いた場合は、このパスも合わせて直す。

ファイル: `gz_practice/arm_controllers.yaml`

<!-- file: gz_practice/arm_controllers.yaml -->
```yaml
# コントローラマネージャと、使うコントローラの設定
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    arm_position_controller:
      type: position_controllers/JointGroupPositionController

    arm_trajectory_controller:
      type: joint_trajectory_controller/JointTrajectoryController

arm_position_controller:
  ros__parameters:
    joints:
      - joint1
      - joint2

arm_trajectory_controller:
  ros__parameters:
    joints:
      - joint1
      - joint2
    command_interfaces:
      - position
    state_interfaces:
      - position
      - velocity
```

- **`controller_manager`**: コントローラマネージャのパラメータ。`update_rate` は、読む → 計算 → 書くを繰り返す周期で、ここでは毎秒100回。続く3つは、使うコントローラの名前（自分で付ける）と種類（`type`）の対応で、ここに書いただけでは、まだ動かない。
- **`arm_position_controller`**: 位置のコントローラ。トピックで受けた角度を、そのまま関節の指令に書く。`joints` に、動かす関節を並べる。
- **`arm_trajectory_controller`**: 軌道のコントローラ。「何秒後にどの角度」という点の並び（軌道）を受け取り、間をなめらかにつないで、刻みごとの指令を書く。使う指令と状態のインターフェースも書く。

### 5-3. 体験: 位置のコントローラで動かす

4-4節と同じく、4つのターミナルを使う。4-4節のGazeboとノードが動いていたら、先にすべて止めておく（同じ名前のモデル `arm` は、1つのワールドに2つ出せない）。

```bash
# T1
gz sim -r empty.sdf
```

```bash
# T2
cd ~/gz_practice

ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(xacro arm_control.urdf.xacro)"
```

```bash
# T3
ros2 run ros_gz_bridge parameter_bridge /clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock
```

```bash
# T4
ros2 run ros_gz_sim create -topic robot_description -name arm
```

T2は、4-4節と同じで、展開するファイルだけが違う。T3は、関節の角度の代わりに、Gazeboのシミュレーションの時刻（`/clock`）を、ROS2の [`rosgraph_msgs/msg/Clock`](https://github.com/ros2/rcl_interfaces/blob/jazzy/rosgraph_msgs/msg/Clock.msg) に中継する。`GazeboSimROS2ControlPlugin` は、コントローラマネージャとコントローラを、シミュレーションの時刻で動かす設定（`use_sim_time`）で起動するので、この時刻が届かないと、`/joint_states` の時刻（`stamp`）が0のままになる。

**期待する結果**（T1〜T4の表示の抜粋。先頭の数字は時刻で、実行ごとに異なる）:

```text
# T1（gz sim。モデルが出た後に、次のような行が続く）
[INFO] [1791810686.539546435] [gz_ros_control]: Loading controller_manager
[INFO] [1791810686.569926672] [controller_manager]: Subscribing to '/robot_description' topic for robot description.
[INFO] [1791810689.350234158] [controller_manager]: Received robot description from topic.
[INFO] [1791810689.356456470] [gz_ros_control]: The position_proportional_gain has been set to: 0.1
[INFO] [1791810689.356608224] [gz_ros_control]: Loading joint: joint1
[INFO] [1791810689.356819982] [gz_ros_control]: Loading joint: joint2
[INFO] [1791810689.358266788] [controller_manager]: Resource Manager has been successfully initialized. Starting Controller Manager services...
# T2（robot_state_publisher）
[INFO] [1791810682.609140804] [robot_state_publisher]: Robot initialized
# T3（parameter_bridge）
[INFO] [1791810682.569635721] [ros_gz_bridge]: Creating GZ->ROS Bridge: [/clock (gz.msgs.Clock) -> /clock (rosgraph_msgs/msg/Clock)] (Lazy 0)
# T4（create）
[INFO] [1791810684.451354843] [ros_gz_sim]: Requesting list of world names.
[INFO] [1791810684.657046516] [ros_gz_sim]: Waiting messages on topic [robot_description].
[INFO] [1791810686.347894716] [ros_gz_sim]: Entity creation successful.
```

T4の `Waiting messages on topic [robot_description].` の行は、2回以上出ることもある。T1には、モデルが出た後に、`GazeboSimROS2ControlPlugin` が起動したコントローラマネージャのログが続く。`Received robot description from topic.` は、コントローラマネージャも、`create` と同じトピック `/robot_description` からURDFを受け取ったことを示す。`Loading joint:` の行で、URDFの `<ros2_control>` に書いた2つの関節が読み込まれ、`Starting Controller Manager services...` で準備ができる（ここに載せた行のほかに、`WARN` の行もいくつか出るが、この体験では問題にならない）。

T1の画面では、腕は4-4節と違い、水平に伸びたまま止まっている。`GazeboSimROS2ControlPlugin` は、どのコントローラも指令を書いていない関節を、その場の角度に保つ（`hold_joints` の設定で、既定で有効）。作成時には、`create` の5秒後に、2つの関節の角度が0のままであることを、Gazeboのトピックで確かめた。

コントローラを読み込んで有効にする。

```bash
# T4
ros2 run controller_manager spawner joint_state_broadcaster arm_position_controller
```

**期待する結果**（先頭の数字は時刻で、実行ごとに異なる。`waiting for service` の行は出ないこともある）:

```text
[INFO] [1791810692.489886635] [spawner_joint_state_broadcaster]: waiting for service /controller_manager/list_controllers to become available...
[INFO] [1791810693.281591640] [spawner_joint_state_broadcaster]: Loaded joint_state_broadcaster
[INFO] [1791810693.331165490] [spawner_joint_state_broadcaster]: Configured and activated joint_state_broadcaster
[INFO] [1791810693.361231133] [spawner_joint_state_broadcaster]: Loaded arm_position_controller
[INFO] [1791810693.390987546] [spawner_joint_state_broadcaster]: Configured and activated arm_position_controller
```

`spawner`（スポナー）は、コントローラマネージャのサービスを呼んで、名前を並べたコントローラを順に読み込み（`Loaded`）、準備して有効にする（`Configured and activated`）小さなプログラムで、終わると自分で終了する。ノードの名前は、最初に並べたコントローラの名前から付く。2つとも `Configured and activated` が出れば成功である。この時点では、位置のコントローラにまだ目標が届いていないので、腕は水平のままである。

位置のコントローラに、目標の角度を送る。位置のコントローラは、トピック `/<コントローラの名前>/commands` で、関節の角度を [`std_msgs/msg/Float64MultiArray`](https://github.com/ros2/common_interfaces/blob/jazzy/std_msgs/msg/Float64MultiArray.msg)（数の並び）として受け取る。並びの順は、設定のファイルの `joints` の順（`joint1`、`joint2`）である。

```bash
# T4
ros2 topic pub --once /arm_position_controller/commands std_msgs/msg/Float64MultiArray "{data: [0.5, -1.0]}"
```

**期待する結果**:

```text
Waiting for at least 1 matching subscription(s)...
publisher: beginning loop
publishing #1: std_msgs.msg.Float64MultiArray(layout=std_msgs.msg.MultiArrayLayout(dim=[], data_offset=0), data=[0.5, -1.0])
```

1回だけ送って終わる（`Waiting for ...` の行は、出ないことや、2回出ることもある）。T1の画面では、`link1` が水平から0.5 rad（約29度）下がり、`link2` は `link1` に対して1.0 rad（約57度）上に折れ、すぐに止まる。

角度を読む。

```bash
# T4
ros2 topic echo /joint_states --once
```

**期待する結果**（`stamp` はシミュレーションの時刻で、読んだ時刻によって変わる）:

```text
header:
  stamp:
    sec: 19
    nanosec: 307000000
  frame_id: base_link
name:
- joint1
- joint2
position:
- 0.5000000000030748
- -0.9999999999948956
velocity:
- 5.102862576933376e-14
- -5.505231687186196e-14
effort:
- .nan
- .nan
---
```

**読み方**:

- 今度の `/joint_states` は、ブリッジでなく `joint_state_broadcaster` が送っている。並ぶのは、`<ros2_control>` に書いた2つの関節だけで、固定の関節 `fix_base` は入らない。`effort` が `.nan`（値が無い）なのは、状態のインターフェースにトルクを書いていないからである。`frame_id` の `base_link` は、`joint_state_broadcaster` の既定の値で、この体験では使わない。
- 角度は、送った目標（0.5、−1.0）に、小数第10位まで一致している。4-4節では、重力と減衰の計算から止まる角度が決まり、手計算と小さくずれた。それと違い、ここでは重力に逆らって、目標の角度にぴったり止まっている。

ぴったり止まるのは、`GazeboSimSystem` の角度の指令が、関節に力をかけるのでなく、関節の角速度を直接決めているからである。公式の説明によると、刻みごとに「角度の差 × `position_proportional_gain`（既定0.1）× `update_rate`」の角速度を関節に与える。差が縮むほど遅くなり、目標に近づいていく動きで、時定数（差が約37%に縮むまでの時間）は $1/(0.1 \times 100) = 0.1$ 秒になる。角速度を直接決めるので、腕の重さや重力、関節の減衰は、止まる角度に効かない。

> **補足（実機との違い）**: 実機のモータは、目標の角度にすぐには向かえず、重さや重力に応じたトルクを出しながら近づく。シミュレーションでも、指令のインターフェースをトルク（`effort`）にし、コントローラの側にPIDの係数を書けば、重さや重力を含めた動きになる（作成時には確かめていない）。角度の指令は、動きが単純で扱いやすいので、この体験のように、まず関節を思いどおりに動かしてみるときや、物理の細部より上の層（動作計画など）を試すときに向く。

### 5-4. 体験: 軌道のコントローラに切り替える

位置のコントローラは、目標を受けるとすぐにそちらへ向かうので、大きく離れた角度を送ると、腕が急に動く。軌道のコントローラは、「2秒後にこの角度、4秒後にこの角度」という点の並びを受け取り、間の角度を補間して、刻みごとに少しずつ指令を書く。

2つのコントローラは、どちらも `joint1` と `joint2` の角度の指令のインターフェースを使う。1つの指令のインターフェースは、1つのコントローラしか使えない（ros2_controlは、これを「claim（確保）」と呼ぶ）ので、位置のコントローラを有効にしたまま、軌道のコントローラを有効にすることはできない。位置のコントローラを止めて外してから、軌道のコントローラを読み込む。

```bash
# T4
ros2 run controller_manager unspawner arm_position_controller

ros2 run controller_manager spawner arm_trajectory_controller
```

**期待する結果**（先頭の数字は時刻で、実行ごとに異なる）:

```text
$ ros2 run controller_manager unspawner arm_position_controller
[INFO] [1791810410.340617469] [unspawner_arm_position_controller]: waiting for service /controller_manager/switch_controller to become available...
[INFO] [1791810412.608193834] [unspawner_arm_position_controller]: Deactivated controller
[INFO] [1791810412.629564983] [unspawner_arm_position_controller]: Unloaded controller
$ ros2 run controller_manager spawner arm_trajectory_controller
[INFO] [1791810413.541017559] [spawner_arm_trajectory_controller]: waiting for service /controller_manager/list_controllers to become available...
[INFO] [1791810415.328701942] [spawner_arm_trajectory_controller]: Loaded arm_trajectory_controller
[INFO] [1791810415.368663057] [spawner_arm_trajectory_controller]: Configured and activated arm_trajectory_controller
```

`unspawner` は、`spawner` の逆で、コントローラを止めて（`Deactivated`）外す（`Unloaded`）。外した後は、どのコントローラも指令を書かないので、腕はその場の角度に保たれる（5-3節の `hold_joints`）。

順番を逆にして、位置のコントローラを外す前に軌道のコントローラを読み込むと、`spawner` は `Failed to activate controller : arm_trajectory_controller` と表示して失敗し、T1には `Unable to activate controller 'arm_trajectory_controller' since the command interface 'joint1/position' is currently claimed by another controller.` と出る（作成時に確かめた）。

軌道のコントローラは、アクション `/<コントローラの名前>/follow_joint_trajectory`（型は `control_msgs/action/FollowJointTrajectory`）で軌道を受け取る。アクションは、時間のかかる処理を頼み、終わったら結果を受け取る通信の形である（フェーズ3-5（[`docs/phase3_5_actions.md`](phase3_5_actions.md)）で扱った。任意の章なので、読んでいなければ、「終わるまで待って結果が返るサービス」と考えればよい）。次の軌道は、2秒後に（−0.5、1.0）、4秒後に（0、0）を通る。

```bash
# T4
ros2 action send_goal /arm_trajectory_controller/follow_joint_trajectory control_msgs/action/FollowJointTrajectory "{trajectory: {joint_names: [joint1, joint2], points: [{positions: [-0.5, 1.0], time_from_start: {sec: 2}}, {positions: [0.0, 0.0], time_from_start: {sec: 4}}]}}"
```

**期待する結果**（末尾の抜粋。前に、送った目標の全体が表示される。`ID` は実行ごとに異なる）:

```text
Goal accepted with ID: 84c1d5f0cb784c618c9abacf88cc1b5a

Result:
    error_code: 0
error_string: Goal successfully reached!

Goal finished with status: SUCCEEDED
```

`Goal accepted` で軌道が受け付けられ、約4秒後に `SUCCEEDED`（成功）と、`error_code: 0`・`Goal successfully reached!` が返れば成功である。T1の画面では、腕が5-3節の姿勢から2秒かけて、`link1` が水平より上（−0.5 rad）、`link2` が下に折れた姿勢へ動き、さらに2秒かけて、水平に伸びた姿勢へ戻る。

作成時に、動いている間の `/joint_states` を記録し、腕が動き始めてから0.5秒ごとに抜き出すと、次のようになった（角度の単位はrad。小数第4位までに丸めた）。

| 経過（秒） | 0 | 0.5 | 1.0 | 1.5 | 2.0 | 2.5 | 3.0 | 3.5 | 4.0 | 4.5 |
|---|---|---|---|---|---|---|---|---|---|---|
| `joint1` | 0.4995 | 0.2903 | 0.0405 | −0.2095 | −0.4588 | −0.3949 | −0.2702 | −0.1452 | −0.0205 | −0.0001 |
| `joint2` | −0.9990 | −0.5806 | −0.0810 | 0.4190 | 0.9177 | 0.7899 | 0.5405 | 0.2905 | 0.0409 | 0.0002 |

**読み方**:

- 0〜2秒の間、`joint1` は0.5秒ごとに約0.25 radずつ、一定の速さ（0.5 rad/s）で変わっている。2〜4秒の間も、別の一定の速さ（`joint1` は0.25 rad/s）で変わる。この軌道では、点ごとに角度だけを書き、角速度を書いていないので、軌道のコントローラは、点と点の間を直線でつないだ。直線でつなぐと、点のところで速さが急に変わるので、公式の説明は勧めていない。点ごとに角速度も書くと3次の曲線で、さらに角加速度も書くと5次の曲線で、なめらかにつなぐ。
- 2秒の時点の角度（−0.4588）は、軌道の点の−0.5に届いていない。差の約0.04 radは、0.5 rad/sで動く目標を、5-3節の時定数0.1秒の遅れで追いかけた分（約0.05 rad）に近い。軌道のコントローラが書く指令は軌道の上にあり、`GazeboSimSystem` がそれを0.1秒ほど遅れて追う。4秒の時点も同じ理由で0に届かず、その後0.5秒ほどで0に落ち着く。

> **補足（`ros2 control` のコマンド）**: コントローラの一覧や状態は、`ros2 control list_controllers` などのコマンドで見られるが、このコマンドは、この章で入れたパッケージには含まれず、別のパッケージ `ros-jazzy-ros2controlcli` にある（作成時の環境で確かめた。この章では使わない）。

止めるときは、T3・T2・T1の順に `Ctrl+C` で止める。

## 6. この章のまとめ

- Gazeboのワールドは、SDFで、ワールド → モデル → リンク・関節の入れ子に書く。リンクは、慣性（質量と慣性モーメント）・衝突の形・見た目の3つを持ち、物理の計算に使うのは慣性と衝突の形である。
- 物理エンジン（既定はDART）は、刻み幅（既定0.001秒）ごとに、まず速度を、次に位置を更新する。落下の位置は、この計算の式と小数第6位まで一致した。接触では、物どうしが数ミリ重なることを許し、少しずつ押し戻す。
- 摩擦係数と反発係数は、衝突の形の `<surface>` に書く。DARTでは、2つの面の摩擦係数は小さいほう、反発係数は積が使われる。坂で滑る距離と、跳ね上がる高さは、手計算と合った。
- ロボット1台は、ROS2の道具と共通のURDFで書き、xacroで短く書いて展開する。Gazeboは、読み込むときにSDFに変換する。Gazebo特有の設定は `<gazebo>` に書く。
- `create` でURDFをワールドに出し、`JointStatePublisher` とブリッジで、関節の角度をROS2の `/joint_states` として読めた。関節の制限は正確に効き、止まった角度には、刻み幅に比例する小さなずれがあった。
- ros2_controlは、コントローラ（指令を計算する側）とハードウェアの部品（関節を読み書きする側）を、インターフェースでつなぐ。URDFの `<ros2_control>` に関節ごとのインターフェースを書き、Gazeboでは `GazeboSimROS2ControlPlugin` がコントローラマネージャを動かす。コントローラは `spawner` で読み込んで有効にし、1つの指令のインターフェースは1つのコントローラだけが使える。
- `GazeboSimSystem` の角度の指令は、関節の角速度を直接決めるので、重力に関係なく目標の角度にぴったり止まり、時定数0.1秒（`position_proportional_gain` 0.1、`update_rate` 100 Hz）の遅れで目標を追う。軌道のコントローラは、点の間を補間した指令を書き、腕はそれを0.1秒ほど遅れて追った。

## 7. つまずきやすい点

| 症状 | 確認すること |
|---|---|
| `gz sdf -k` がエラーを出す | 要素の綴りと入れ子。`<collision>` の中に `<geometry>` があるか。数の並び（`<pose>` は6つ、`<size>` は箱なら3つ）が合っているか |
| ▶を押しても何も落ちない | ワールドに `gz-sim-physics-system` の `<plugin>` があるか（2-2節）。モデルに `<static>true</static>` を付けていないか |
| `gz service` が `Service call timed out` になる | ワールド名（`/world/<名前>/control` の `<名前>`）が、SDFの `<world name="...">` と同じか。T1でGazeboが開いているか |
| `gz model` で読んだ位置が、表の値より手前の位置になる | `multi_step` は現実の時間の速さで進むので、進み終わる前に読んでいる。画面の動きが止まってから読む |
| 跳ねるはずの球が跳ねない | 相手（地面）の反発係数が0のままでないか。DARTでは2つの積が使われる（3-3節） |
| `create` が `Waiting messages on topic [robot_description]` で止まる | T2の `robot_state_publisher` が動いているか。T2のxacroの展開でエラーが出ていないか |
| アームが台ごと倒れる | URDFに `world` のリンクと、台へつなぐ固定の関節があるか（4-3節） |
| `/joint_states` に何も届かない | URDFの `<gazebo>` に `JointStatePublisher` があるか。T3のトピック名のワールド名（`empty`）とモデル名（`arm`。`create` の `-name`）が合っているか（4-4節）。5節では、`spawner` で `joint_state_broadcaster` を有効にしたか |
| `spawner` が `waiting for service /controller_manager/list_controllers` のまま進まない | コントローラマネージャが起動していない。T1に `Loading controller_manager` が出ているか。出ていれば、`Starting Controller Manager services...` まで進んでいるか（T2の `robot_state_publisher` が動いていないと、URDFを待ち続ける）。展開したURDFの `<parameters>` のパスに、設定のファイルがあるか |
| `spawner` が `Failed to activate controller` で失敗する | ほかのコントローラが、同じ関節の指令のインターフェースを使っていないか（5-4節）。失敗したコントローラは、読み込まれたまま（`Loaded` の後の有効化だけが失敗した状態で）残るので、T1〜T4をすべて止めてやり直し、先に `unspawner` で外してから読み込むのが確実である |
| T1に `No clock received` の警告が続き、`/joint_states` の `stamp` が0のまま | T3の `/clock` のブリッジが動いているか（5-3節） |
| 5節で、腕が4-4節のように垂れ下がる | T2で展開しているのが `arm_control.urdf.xacro` か（`arm.urdf.xacro` には `<ros2_control>` が無い） |

## 8. 次へ

この章は任意の補足なので、本筋はフェーズ6-1（[`docs/phase6_1_record.md`](phase6_1_record.md)）へ進む。フェーズ7-3（[`docs/phase7_3_gazebo_env.md`](phase7_3_gazebo_env.md)）では、フェーズ5-4の車両の車輪の関節に、この章の4-3節と同じ考え方の減衰（SDFでは、関節の `<axis>` の `<dynamics>` の中の `<damping>` の要素）を足して使う。

## 9. 公式ドキュメント・参考資料

確認状況（2026-10-12）: Gazeboの「SDF worlds」「Building your own robot」「Spawn URDF」「Spawn a Gazebo model from ROS 2」のページは、この章の作成時に本文を確認した。要素の既定値は、導入したSDFormat（14.9.0）の仕様の表示（`gz sdf -d`）で、`create` のオプションは `ros2 run ros_gz_sim create --help` で確認した。URDFとxacroのROS 2のチュートリアルは、原稿（`ros2/ros2_documentation` のjazzyブランチ）が実在することを確かめた。摩擦と反発の係数の組み合わせ方と、4-4節の角度のずれは、作成時の実験で確かめたもので、DARTのソースは読んでいない。5節のros2_controlの記述と設定は、`gz_ros2_control` のjazzyの版の説明とデモの設定のファイル、ros2_controlの公式の文書（jazzy）を確認し、作成時に導入した版（`gz_ros2_control` 1.2.20、`ros2_controllers` 4.42.1、`controller_manager` 4.48.1）で動かして確かめた。角度の指令の動き（`position_proportional_gain`）と軌道の補間の仕方は、公式の説明と、5-4節の記録が合うことを確かめた。

### 公式

- [SDF worlds — Gazebo Harmonic](https://gazebosim.org/docs/harmonic/sdf_worlds/)（ワールドのファイルの書き方。物理の設定・システム・光・モデルの読み込み）
- [Building your own robot — Gazebo Harmonic](https://gazebosim.org/docs/harmonic/building_robot/)（SDFでモデル・リンク・慣性・衝突と見た目・関節を書くチュートリアル）
- [Spawn URDF — Gazebo Harmonic](https://gazebosim.org/docs/harmonic/spawn_urdf/)（URDFをSDFに変換してワールドに出す仕組み。Gazeboのサービスで出す方法）
- [Spawn a Gazebo model from ROS 2 — Gazebo Harmonic](https://gazebosim.org/docs/harmonic/ros2_spawn_model/)（ROS 2からモデルを出す方法。Jazzyのlaunchファイル `gz_spawn_model.launch.py` も紹介している）
- [SDFormat Specification](https://sdformat.org/spec)（SDFの要素の一覧と既定値）
- [gz::sim::systems::Physics — Gazebo Sim 8 APIリファレンス](https://gazebosim.org/api/sim/8/classgz_1_1sim_1_1systems_1_1Physics.html)（物理のシステム）
- [gz::sim::systems::JointStatePublisher — Gazebo Sim 8 APIリファレンス](https://gazebosim.org/api/sim/8/classgz_1_1sim_1_1systems_1_1JointStatePublisher.html)（関節の状態を送るシステム。トピック名の決まり）
- [Building a visual robot model from scratch — ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/Building-a-Visual-Robot-Model-with-URDF-from-Scratch.html)（URDFの書き方のチュートリアル）
- [Using Xacro to clean up your code — ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/Using-Xacro-to-Clean-Up-a-URDF-File.html)（xacroのプロパティ・式・マクロ）
- [ros/xacro（GitHub）](https://github.com/ros/xacro)（xacroのソースと説明）
- [Getting Started — ros2_control: Jazzy](https://control.ros.org/jazzy/doc/getting_started/getting_started.html)（ros2_controlの部品と、コントローラマネージャ・インターフェースの考え方）
- [gz_ros2_control — ros2_control: Jazzy](https://control.ros.org/jazzy/doc/gz_ros2_control/doc/index.html)（URDFの `<ros2_control>` と `GazeboSimROS2ControlPlugin` の書き方。`hold_joints`・`position_proportional_gain` の説明）
- [ros-controls/gz_ros2_control（GitHub、jazzyブランチ）](https://github.com/ros-controls/gz_ros2_control/tree/jazzy)（ソースとデモ。デモのURDFと設定のファイルが手本になる）
- [joint_state_broadcaster — ros2_control: Jazzy](https://control.ros.org/jazzy/doc/ros2_controllers/joint_state_broadcaster/doc/userdoc.html)・[position_controllers](https://control.ros.org/jazzy/doc/ros2_controllers/position_controllers/doc/userdoc.html)・[joint_trajectory_controller](https://control.ros.org/jazzy/doc/ros2_controllers/joint_trajectory_controller/doc/userdoc.html)（この章で使った3つのコントローラ。軌道の補間の仕方は、同じ場所の「Trajectory Representation」のページ）

> 出典: ワールドとロボットのファイルと、コントローラの設定のファイルは、上記の公式の文書とSDFの仕様の要素の名前と書き方にならって独自に書いたもので、転載ではない。`gz sdf`・`xacro`・`check_urdf`・ノード・コントローラ・`ros2` のコマンドの表示は、導入したツールの出力を載せたもの（SDFormat・Gazebo Sim・ros_gz・gz_ros2_control・ros2_control・ros2_controllers・ros2cli はApache License 2.0、xacro・urdfdom・robot_state_publisher はBSD License）。文章は独自に書いたもの。
