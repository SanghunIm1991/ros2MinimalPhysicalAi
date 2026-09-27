# Tips集: 手順書の本文に入れなかった小さな補足

手順書（フェーズ）の本文に入れると流れが途切れる、分量の少ない補足をまとめた資料。どの項目も独立しているので、関係する手順書を読んでいるときや、疑問を持ったときに、その項目だけを読めばよい。

- 対象: ROS2 Jazzy（Ubuntu 24.04）。版に依存する記述は、この資料の作成時に `/opt/ros/jazzy` とUbuntuのパッケージに入っていたもので確かめた
- 所要目安: 1項目あたり5〜15分（読み物）

> **実行環境が無くても読めるように**: コマンドを載せた箇所には、その直後に「期待する結果」として表示の例と読み方を書いている。ノードを起動した後の表示（1-4節）は、ROS2の仕様から想定したもの。3-1節の警告の文面は、使い捨ての環境で実際に表示させたもの。5-3節の `ros2 param` の表示は、手順書の作成時にturtlesimを起動して実際に確かめたもの（Tabキーでの補完の表示は確かめていない）。8-4節の表示は、この資料の作成時にデモのtalkerを起動して実際に確かめたもの。

> **出どころ**: 内容は、`/opt/ros/jazzy` のソース（`rclpy`・`rclcpp`）、`ros2 pkg create` が作る雛形、Ubuntuのパッケージ（`colcon-core`・`setuptools`）を読んで確かめ、自分の言葉で書いた。ソースやドキュメントの転載ではない。4節のDDSの仕組み（ディスカバリ・HEARTBEAT・ACKNACK・既定の送り方）は、DDSとFast DDSの一般的な仕組みから書いたもので、この資料の作成時に通信を観察して確かめてはいない。既定の実装が `rmw_fastrtps_cpp`（Fast DDS 2.14系）であることは、`/opt/ros/jazzy` で確かめた。4-2節の警告の文面は、フェーズ3-2bに載せたもの（コードとROS2の仕様から想定した表示）を引いた。5節は、`/opt/ros/jazzy` の `ros2param`・`ros2run`・`ros2cli` のソース（補完の仕組み）とturtlesimのプログラムを読み、turtlesimを起動して `ros2 param` の表示を確かめた。5-4節の `generate_parameter_library` は、Jazzy向けのaptパッケージがあることだけを確かめ、機能の説明はこのライブラリについての一般的な情報から書いた（この環境には導入しておらず、動かしていない）。6節・7節は、フェーズ0の本文から移したもので、ROS2・DDSの公式ページの記載（URLはフェーズ0（[`docs/phase0_overview.md`](phase0_overview.md)）の9節）に基づいて書いた。6節のRMWの導入状況（ROS2と一緒に入るもの、aptにあるパッケージ、依存関係に製品本体が含まれるか）は、この資料の作成時に `/opt/ros/jazzy` とaptのパッケージ情報で確かめた。8節の名前の組み立て方はROS2の名前の規則から書き、8-4節の表示（C++版・Python版のデモのtalkerで、YAMLのノード名3通り）とトピック名の付き方は、この資料の作成時に実際に起動して確かめた。8-2節のプライベート名と、8-3節の `ros2 topic pub` に相対名を渡した場合の送り先は、動かして確かめてはいない。

## 項目の一覧

| 節 | 項目 | 関係する手順書 |
|---|---|---|
| 1 | シミュレーション時刻（`use_sim_time`） | フェーズ1（パラメータの一覧）、フェーズ3-1（時刻の取り方）、フェーズ5-0（Gazebo）、フェーズ5-4（Gazeboの物理と閉ループ） |
| 2 | `CMakeLists.txt` の読み方（モダンCMakeの要点） | フェーズ2の2-3節以降、C++のサンプルすべて |
| 3 | ビルド中の `SetuptoolsDeprecationWarning` の意味 | フェーズ2の2-4節 |
| 4 | トピック通信の裏側（ディスカバリと届いたことの確認） | フェーズ3-2b（QoS） |
| 5 | パラメータを利用者に知らせる方法 | フェーズ3-3（パラメータ）、フェーズ4（launch） |
| 6 | ROS2の通信の層を詳しく（DDS・RMW・ディスカバリ・QoS） | フェーズ0の2節 |
| 7 | ROS1との違い | フェーズ0の1節 |
| 8 | 名前空間（namespace）が効く範囲 | フェーズ3-2a（絶対名のトピック）、フェーズ4の4-6節（名前空間・remap） |

## 1. シミュレーション時刻（`use_sim_time`）

> **読める時期**: フェーズ1（[`docs/phase1_cli_turtlesim.md`](phase1_cli_turtlesim.md)）やフェーズ3-1（[`docs/phase3_1_pubsub.md`](phase3_1_pubsub.md)）の時点では、1-1節の最初の段落（2つの時計）、1-4節（切り替え方）、1-5節（気をつける点）だけを読めばよい（1-4節の例はフェーズ3-1の `talker` を使うので、試すのはフェーズ3-1の後）。1-1節の2つ目の段落・1-2節・1-3節は、Gazebo（フェーズ5-0）やPI制御（フェーズ5-2）を前提にしているので、フェーズ5-0（[`docs/phase5_0_gazebo.md`](phase5_0_gazebo.md)）を済ませた後に読むと分かりやすい。

### 1-1. 時計が2つある

フェーズ1の `ros2 param list` や、フェーズ3-3の `ros2 param list` には、自分で宣言していない `use_sim_time` というパラメータが並んでいた。これは「ノードの時計を、現実の時刻にするか、シミュレーションの時刻にするか」を切り替えるスイッチで、すべてのノードが自動で持っている（`Node` が作られるときに用意される。参考資料 [`docs/reference_node_class.md`](reference_node_class.md) の2節）。

- **現実の時刻**（`use_sim_time` が `false`。既定）: PCの時計そのもの。1秒は現実の1秒。
- **シミュレーションの時刻**（`use_sim_time` が `true`）: シミュレータが「今、シミュレーションの中では何秒か」を知らせてくる時刻。シミュレータを一時停止すれば止まり、PCが重ければ現実よりゆっくり進む。

フェーズ5-0の2-1節では、Gazeboが一時停止の状態で始まること、右下のRTF（シミュレーションの時間が現実の何倍の速さで進んでいるか）が100%を下回ることがあることを見た。このとき、シミュレーションの中の車両は「シミュレーションの時刻」で動いている。フェーズ5-0の3-4節（オドメトリを観察する）で見た `header.stamp` も、Gazeboを起動してからのシミュレーションの時刻だった。

### 1-2. なぜ合わせる必要があるのか

制御のノードが現実の時刻で動き、車両がシミュレーションの時刻で動いていると、2つの時計がずれる。たとえばRTFが50%のとき、制御のノードが「1秒たったので速度を上げる」と判断しても、シミュレーションの中ではまだ0.5秒しかたっていない。PI制御の積分のように「経過時間 × 値」を足していく計算では、このずれがそのまま誤差になる。一時停止中も制御のノードの時計だけが進み、再生したとたんに大きな指令を出す、ということも起こる。

`use_sim_time` を `true` にすると、ノードの時計がシミュレーションの時刻に合うので、PCの速さに関係なく同じ結果になる。記録したデータ（rosbag）を再生して、ノードを当時と同じ時刻で動かすときにも同じ仕組みを使う。

### 1-3. 仕組み: `/clock` トピック

シミュレーションの時刻は、`/clock` というトピック（型は [`rosgraph_msgs/msg/Clock`](https://github.com/ros2/rcl_interfaces/blob/jazzy/rosgraph_msgs/msg/Clock.msg)）で配られる。`use_sim_time` が `true` のノードは、このトピックを自動で購読し、届いた時刻を自分の時計の現在時刻にする。rclpyのソース（`rclpy/time_source.py`）では、この購読のQoSは `best_effort`・depth 1になっている（最新の時刻だけが大事なので、取りこぼしても再送しない）。

Gazeboの場合、時刻はまずGazeboの側の `/clock` に出る（フェーズ5-0の2-1節の `gz topic -l` の一覧にある）。ROS2のノードに届けるには、ほかのトピックと同じくブリッジが必要である。

```text
/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock
```

`parameter_bridge` の引数にこの1本を足すと、Gazeboの時刻がROS2の `/clock` に中継される。区切りの `[` は「Gazebo→ROS2の片方向」の意味（フェーズ5-0の3-1節）。

フェーズ5-0で使ったデモ（`diff_drive.launch.py`）は、3-1節の `--print` の表示のとおり、`cmd_vel` と `odometry` の4本しか中継しておらず、`/clock` は中継していない。したがって、フェーズ5-0の課題6（10秒走ったら止まるノード）の `self.get_clock().now()` は、現実の時刻で測っている。フェーズ5-3のlaunchも、このデモを取り込むだけなので同じである（制御のループは、Gazeboと関係なく自作のノードだけで閉じているので、困らない）。Gazeboの物理をプラントにするフェーズ5-4（[`docs/phase5_4_gazebo_plant.md`](phase5_4_gazebo_plant.md)）の5節で、自分のlaunchのブリッジにこの1本を足し、計算するノードに `use_sim_time` を渡す。

### 1-4. 切り替え方

実行時に、ほかのパラメータと同じ書き方で渡す（フェーズ3-3の5-2節の `-p`）。launchファイルでは、ノードに渡すパラメータに `{'use_sim_time': True}` を加える（フェーズ4のパラメータの渡し方と同じ）。

次の例は、フェーズ3-1の `talker` をシミュレーションの時刻で動かし、別のターミナルで設定を確かめるものである。`--ros-args -p 名前:=値` は、起動するときにパラメータの値を指定する書き方で、ここでは `use_sim_time` を `true` にして起動する（詳しくはフェーズ3-3（[`docs/phase3_3_parameters.md`](phase3_3_parameters.md)）の5-2節（起動時に指定する）で扱う）。シミュレータは動かしていないので、`/clock` はどこからも届かない。

```bash
# T1
ros2 run learn_py talker --ros-args -p use_sim_time:=true

# T2
ros2 param get /talker use_sim_time
```

**期待する結果**（書式はフェーズ1の `ros2 param get` と同じ）: T1には、`publish:` のログが1行も出ない。

```text
# T2
$ ros2 param get /talker use_sim_time
Boolean value is: True
```

T2の `True` は、そのノードの時計がシミュレーションの時刻を使っていることを示す。T1にログが出ないのは、壊れたのではない。`/clock` が届かないので時刻が0のまま進まず、ノードの時計で数えるタイマー（Pythonの `create_timer`）が一度も発火しないためである（1-5節の表の「`/clock` が届くまで時刻は0」「タイマーも時計に従う」）。確かめたら、T1を `Ctrl+C` で止める。

### 1-5. 気をつける点

| 気をつける点 | 内容 |
|---|---|
| `/clock` が届くまで時刻は0 | `use_sim_time` を `true` にしたのに `/clock` が来ていないと、`now()` は0を返し続ける（rclpyの `TimeSource` は、最後に受け取った時刻を覚えておく変数を0で始める）。時刻が0のままなら、ブリッジに `/clock` を足したか、シミュレータが一時停止していないかを確かめる |
| タイマーも時計に従う | Pythonの `create_timer` と、C++の `create_timer` は、ノードの時計で数える。シミュレーションの時刻を使うと、一時停止中はタイマーも止まる。C++の `create_wall_timer`（フェーズ3-1のC++版で使った）は、名前のとおり常に現実の時刻で数える |
| ノードごとに揃える | 一部のノードだけ `true` にすると、ノードによって時刻の基準が違うメッセージが混ざる。シミュレーションに関わるノードは全部揃える。launchでまとめて起動すると揃えやすい |

## 2. `CMakeLists.txt` の読み方（モダンCMakeの要点）

### 2-1. 「モダンCMake」とは

CMakeは、C++のビルドの手順を書くための道具である。長く使われてきた道具なので、書き方には古い流儀と新しい流儀がある。おおよそCMake 3.0（2014年）ごろから広まった新しい流儀を「モダンCMake」と呼ぶ。考え方の中心は次の1点である。

**設定は「ターゲット」ごとに付ける。** ターゲットとは、`add_executable`（実行ファイル）や `add_library`（ライブラリ）で作る、ビルドの成果物1つ1つのことである。古い流儀では、`include_directories(...)` や `add_definitions(...)` のように、そのフォルダ（`CMakeLists.txt`）の**全部の**成果物にまとめて効く命令で設定していた。モダンCMakeでは、`target_` で始まる命令で「どのターゲットに、何が必要か」を1つずつ書く。成果物が増えても、互いの設定が混ざらない。

フェーズ2の `ros2 pkg create` が作った雛形も、この流儀で書かれている。C++のコード自体（C++17）とは別の話で、C++を書いた経験があっても、CMakeを使ったことがなければ馴染みのない書き方である。

### 2-2. 雛形を1行ずつ読む

フェーズ2の2-3節で作った `learn_cpp/CMakeLists.txt`（`ros2 pkg create --build-type ament_cmake` の雛形。使い捨ての環境で作って確かめた）の主な行を、上から順に表にした。テストの部分（`if(BUILD_TESTING)` の中）は省いている。

| 行 | 意味 |
|---|---|
| `cmake_minimum_required(VERSION 3.8)` | このファイルを読むのに必要なCMakeの最低の版。版によって使える命令が違うので、最初に宣言する |
| `project(learn_cpp)` | プロジェクト名を決める。以降、`${PROJECT_NAME}` でこの名前を参照できる |
| `if(CMAKE_COMPILER_IS_GNUCXX OR ...)` と `add_compile_options(-Wall -Wextra -Wpedantic)` | コンパイラがGCCかClangなら、警告を多めに出すオプションを付ける。`add_compile_options` はターゲットごとでなく全体に効く古い流儀の命令だが、「このパッケージは全部、警告を多めに」という方針なので全体に掛けている |
| `find_package(rclcpp REQUIRED)` | 外部のパッケージ（ここでは `rclcpp`）を探して、使えるようにする。`REQUIRED` は「見つからなければ、ここでエラーにする」。探す先は、`source` した環境（`/opt/ros/jazzy` など）のパッケージが置いている設定ファイル |
| `add_executable(hello src/hello.cpp)` | `hello` という名前のターゲット（実行ファイル）を、`src/hello.cpp` から作る。以降の `target_...` の命令は、この名前で対象を指す |
| `target_include_directories(hello PUBLIC ...)` | `hello` をビルドするとき、ヘッダを探す場所を足す（2-3節で詳しく） |
| `target_compile_features(hello PUBLIC c_std_99 cxx_std_17)` | `hello` には、C++17（とC99）に対応したコンパイルが必要、と宣言する。古い流儀では `-std=c++17` のようなオプションを直接書いたが、この書き方ならCMakeがコンパイラに合ったオプションを選ぶ |
| `ament_target_dependencies(hello "rclcpp" "std_msgs")` | `hello` が `rclcpp` と `std_msgs` を使う、と宣言する。`ament_cmake` が用意している関数で、CMakeの標準の命令ではない。中では、それぞれのパッケージのヘッダの場所とライブラリを、`hello` に結び付けている（CMake標準の `target_include_directories` と `target_link_libraries` をまとめて呼ぶのに近い） |
| `install(TARGETS hello DESTINATION lib/${PROJECT_NAME})` | ビルドした `hello` を、`install/learn_cpp/lib/learn_cpp/` へ置く。`ros2 run` はこの場所から実行ファイルを探す（フェーズ2の2-6節） |
| `ament_package()` | ROS2のパッケージとして必要な情報（索引への登録など）を書き出す。**ファイルの最後**に置く決まり |

フェーズ3-1以降でC++のサンプルを足すたびに書く `add_executable`・`ament_target_dependencies`・`install(TARGETS ...)` の3点セットは、この表の同じ行の繰り返しである。

### 2-3. `PUBLIC` / `PRIVATE` と `$<...>`

`target_` の命令には、`PUBLIC`・`PRIVATE`・`INTERFACE` のどれかを書く。これは「その設定を、このターゲットを**使う側**にも伝えるか」の指定である。

| キーワード | このターゲット自身のビルドに使う | このターゲットを使う側にも伝える |
|---|---|---|
| `PRIVATE` | 使う | 伝えない |
| `PUBLIC` | 使う | 伝える |
| `INTERFACE` | 使わない | 伝える |

違いが効いてくるのは、ライブラリを作って別のターゲットやパッケージから使わせるとき（間章のコンポーネントのような場合）である。実行ファイルはほかから使われないので、どれを書いても結果はほぼ同じになる。フェーズ3-2bで共通部品のヘッダを読むときに `PRIVATE` を書いたのは、「このターゲットの中だけで使う」ことをはっきりさせるためである。

雛形の `target_include_directories` にある `$<BUILD_INTERFACE:...>` と `$<INSTALL_INTERFACE:...>` は、**ジェネレータ式**と呼ばれる書き方である。`$<条件:値>` の形で、「条件に合うときだけ値を使う」という意味になる。ここでは、ビルドするときはソースの `include/` を、インストールした後は `install/` の中の `include/learn_cpp` を、ヘッダの探し先にしている。ヘッダをライブラリとして公開する場合のための書き方で、実行ファイルがパッケージの中のヘッダを読むだけなら、フェーズ3-2bのように短く書いてよい（フェーズ3-2bの該当箇所の補足にも同じ説明がある）。

### 2-4. 読むときのコツ

- **ターゲット名を追う。** `add_executable(名前 ...)` で生まれた名前が、`target_...(名前 ...)`・`ament_target_dependencies(名前 ...)`・`install(TARGETS 名前 ...)` に何度も出てくる。同じ名前の行を拾えば、1つの成果物に必要な設定がそろう。
- **依存は3か所。** C++では、`package.xml` の `<depend>`、`find_package`、`ament_target_dependencies` の3か所に依存を書く（フェーズ3-2aの「Python版とC++版の違いのまとめ」でも整理する）。`find_package` は「探して使える状態にする」、`ament_target_dependencies` は「どのターゲットがそれを使うか」で、役割が違う。
- **`ament_` で始まるものはROS2の追加分。** CMakeの標準の命令と見分けがつけば、CMakeの一般的な解説と、ROS2の解説のどちらを調べればよいかが分かる。

## 3. ビルド中の `SetuptoolsDeprecationWarning` の意味

### 3-1. 警告の意味

フェーズ2の2-4節では、Pythonパッケージのビルド中に `SetuptoolsDeprecationWarning` が出ることがあり、ビルドが成功していれば無視してよい、とだけ書いた。この警告の意味は次のとおりである。

- **setuptools**: Pythonのパッケージをビルド・インストールするための道具。`ament_python` のパッケージにある `setup.py` は、setuptoolsの書き方である。
- **DeprecationWarning（非推奨の警告）**: 「今は動くが、将来の版で使えなくなる予定の使い方をしている」という予告。エラーではないので、ビルドはそのまま続く。

何が非推奨なのかというと、**`setup.py` を直接実行してインストールする使い方**である。Pythonのパッケージの作り方は、`pyproject.toml` という設定ファイルと、`pip` などの標準化された道具を使う方式へ移ってきている。一方、colconは `ament_python` のパッケージをビルドするとき、裏で `python3 setup.py ...` を直接実行している。`--symlink-install` を付けたときは `setup.py develop`、付けないときは `setup.py install` の形である（colconのログの `command.log` に、実行したコマンドが残る）。setuptoolsから見ると「もう勧めていない使い方」なので、警告を出す。

使い捨ての環境で、colconを通さずに `setup.py install` を直接実行すると、次の警告が出た（Ubuntu 24.04のsetuptools 68.1.2。前後の行は省略）。

```text
SetuptoolsDeprecationWarning: setup.py install is deprecated.
!!

        ********************************************************************************
        Please avoid running ``setup.py`` directly.
        Instead, use pypa/build, pypa/installer or other
        standards-based tools.
```

「`setup.py` を直接実行しないで、標準化された道具を使ってほしい」という内容である。`setup.py develop` のときは、同じ趣旨の `EasyInstallDeprecationWarning`（`easy_install` という古いインストールの仕組みを使っている、という警告）も出た。

### 3-2. 無視してよい理由と、ふだんは表示されない理由

この使い方を変えるのはcolconとROS2の側の仕事で、パッケージを書く側が直せるものではない。ROS2 Jazzyの組み合わせでは、この使い方でビルドが正しくできる。だから、ビルドが成功していれば無視してよい。

実際には、colcon自身がこの2つの警告を表示しないようにしている。colconは `python3` を `-W ignore:setup.py install is deprecated -W ignore:easy_install command is deprecated` という引数付きで実行している（`-W ignore:文言` は、その文言で始まる警告を表示しないというPythonの指定）。Ubuntu 24.04のパッケージ（colcon-core 0.21.3・setuptools 68.1.2）の組み合わせで、フェーズ2の2つのパッケージを `colcon build --symlink-install` したところ、警告は1行も出なかった。

それでも警告を見かけることがあるのは、主に次のような場合である。

- **setuptoolsの版が違う**: `pip` で新しいsetuptoolsを入れていると、警告の文言が変わり、colconの指定（文言の先頭の一致で判定する）に当たらなくなることがある。ROS2のPythonパッケージはシステムのPython（aptで入れたもの）に合わせて作られているので、システム側へ `pip` でsetuptoolsを入れ直すことは避ける。
- **別の非推奨を踏んでいる**: たとえば `setup.cfg` に `script-dir` のようなハイフン区切りの名前を書くと、「アンダースコア区切り（`script_dir`）を使ってほしい」という別の `SetuptoolsDeprecationWarning` が出る（setuptools 68.1.2のソースで確認）。古いROS2の版向けに書かれたパッケージや記事をそのまま使うと出ることがある。こちらはパッケージを書く側で直せるので、警告の文面に出てくる名前に書き換える。

どちらの場合も、判断の基準はフェーズ2の4節の表と同じで、`Summary` の行で失敗（`Failed`）が無いかを確かめる。警告の文面を読み、自分のパッケージのファイル名や項目名が出てくるなら直せる警告、そうでなければ道具の側の話、と見分けるとよい。

## 4. トピック通信の裏側: 相手をどう見つけ、届いたことをどう確かめるか

### 4-1. 疑問: 配信側は受信側のことを知っているのか

ネットワークでブロードキャストをするとき、送る側はふつう受け手の事情を考えない。では、ROS2のPublisherも、Subscriberのことを知らずに送りっぱなしにしているのだろうか。

答えは「コードの上では知らないが、下の層は知っている」である。`publish()` を呼ぶプログラムは、相手が何人いるかを意識しない。一方で、その下でトピック通信を担うDDS（Data Distribution Service。ROS2 Jazzyの既定の実装はFast DDSで、ROS2からは `rmw_fastrtps_cpp` として使われる）は、送る前に相手を探し、トピック名・型・QoSを確かめ合ってから送っている。

### 4-2. 送る前に相手を見つける（ディスカバリ）

DDSは、データを送る前に次の2段階で相手を探す。

1. **参加者を見つける**: 各プロセスが「ここにROS2の参加者がいる」と、定期的に知らせ合う。既定の設定では、この知らせをマルチキャストで送る。ブロードキャストに近い送り方をするのは、この段階だけである。
2. **送受信口を見つける**: 見つけた相手と、「どのトピックを、どの型で、どのQoSで、送る側か受ける側か」を交換する。

2段階目で、トピック名と型が同じでも、QoSが条件を満たさなければ、つながらない（マッチしない）。条件は「購読側が求める水準を、配信側が満たしているか」である。配信側が `reliable`、購読側が `best_effort` ならつながる。逆に配信側が `best_effort`、購読側が `reliable` なら、つながらない。

フェーズ3-2b（[`docs/phase3_2b_qos.md`](phase3_2b_qos.md)）の5-1節の④（talkerが `best_effort`、listenerが `reliable`）で出る次の警告は、この段階で出ている。`New publisher discovered`（新しい配信側を見つけた）という書き出しのとおり、相手を見つけたうえで、QoSが合わないので「やり取りしない」と判断している。

```text
[WARN] [...] [qos_listener]: New publisher discovered on topic '/qos_test', offering incompatible QoS. No messages will be received from it. Last incompatible policy: RELIABILITY
```

`ros2 topic info -v` で、相手のノード名やQoSが表示されるのも、この段階で交換した情報を見せているからである。

なお、どこまで相手を探しに行くかは、環境変数 `ROS_AUTOMATIC_DISCOVERY_RANGE` で絞れる（`LOCALHOST` にすると同じPCの中だけ）。

### 4-3. 送るときと、届いたことの確かめ方

マッチした後のデータは、全員に一斉に送るのではなく、マッチした購読側それぞれの宛先へ送る。Fast DDSの既定では、別のPCへはユニキャストのUDPで、同じPCの中では共有メモリで送る。届いたことを確かめるかどうかは、QoSの `reliability` で変わる。

- **`best_effort`**: 送りっぱなしで、届いたかどうかを確かめない。途中で失われたデータは、そのまま失われる。「受け手の事情を考えない」送り方に近いのは、こちらである。
- **`reliable`**: 配信側が「ここまで送った」という知らせ（HEARTBEAT）を出し、購読側が「ここまで受け取った、これが欠けている」（ACKNACK）と返す。配信側は、欠けた分を送り直す。TCPに似た仕組みだが、DDSの通信規約（RTPS）がUDPの上で自前で行っている。送り直せるのは、配信側が手元に残している分（QoSの `depth` の件数）だけである。

`durability` の `transient_local` も、相手を把握しているからこそできる。配信側が送ったデータを手元に残しておき、後から参加した購読側を見つけた時点で、残しておいた分を渡す。渡すのは、購読側も `transient_local` を求めている場合だけである。フェーズ3-2bの5-2節では、⑤（talker・listenerとも `transient_local`）で過去の分が届き、⑥（listenerが `volatile`）ではつながるが過去の分は届かない。⑦（talkerが `volatile`、listenerが `transient_local`）は、この文書の4-2節で説明したQoSの突き合わせで、つながらない。

### 4-4. まとめ

| 段階 | 相手を把握しているか | 届いたことの確認 |
|---|---|---|
| ディスカバリ | 把握する（トピック名・型・QoSを交換する） | ―（参加者の知らせは、既定ではマルチキャスト） |
| 送信（`best_effort`） | 把握している（宛先を知っている） | しない（送りっぱなし） |
| 送信（`reliable`） | 把握している | する（HEARTBEAT・ACKNACKで、欠けた分を送り直す） |

## 5. パラメータを利用者に知らせる方法

### 5-1. 困りごと: 起動するまで一覧が分からない

作ったノードを人に渡すとき、「どんなパラメータがあり、どんな値を入れてよいか」をどう伝えるか。ソースコードを読んでもらうか、実際に動かして試してもらうのでは不便である。`ros2 run パッケージ 実行ファイル --ros-args -p 名前:=値` と打つときに、Tabキーでパラメータ名の候補が出ることもない。

これは、ROS2のパラメータが、ノードが起動して `declare_parameter` を呼んだ時点で初めて決まる仕組みだからである。実行ファイルを起動しないまま、そのパラメータの一覧を取り出す標準の方法は無い。そこで実務では、「起動中のノードに問い合わせる方法」と「起動しなくても分かるように、別の形で知らせる方法」を組み合わせる。

### 5-2. 起動中のノードに問い合わせる

ノードが動いていれば、`ros2 param list /ノード名` で一覧が、`ros2 param describe /ノード名 パラメータ名` で型・説明・値の制約が分かる（フェーズ3-3（[`docs/phase3_3_parameters.md`](phase3_3_parameters.md)）の5-3節）。

`describe` に説明や値の範囲を出すには、ノードの側で、`declare_parameter` の引数に `ParameterDescriptor`（パラメータの説明書き）を渡しておく。主に設定できるのは次の項目で、範囲を付けておくと、範囲の外の値を設定しようとしたときに自動で拒否される。

| 項目 | 中身 |
|---|---|
| `description` | 説明文 |
| `read_only` | 起動した後は変えられないようにするか |
| `integer_range` / `floating_point_range` | 整数・小数の値の範囲（最小・最大・刻み） |
| `additional_constraints` | 範囲では表せない制約を、文章で書く |

自分のノードに付ける練習は、フェーズ3-3の5-4節の末尾にある課題6（発展。`ParameterDescriptor` に説明文と `FloatingPointRange` を付ける）で行える。

起動中のノードに対しては、Tabキーでの補完も効く。`ros2 param get /turtlesim ` や `ros2 param describe /turtlesim ` のようにノード名まで打ってTabキーを押すと、そのノードに問い合わせて、パラメータ名の候補を出す。`ros2` の補完は、`source /opt/ros/jazzy/setup.bash` で自動的に有効になる（`ros2cli` パッケージが、Pythonの補完ライブラリ `argcomplete` を登録する）。一方、`ros2 run` は、実行ファイル名より後ろの引数（`--ros-args -p ...`）を補完しない。補完しようにも、起動する前のノードからはパラメータの一覧を取れないからである（5-1節）。

### 5-3. turtlesimで確かめる

turtlesimは、背景色のパラメータに説明文と値の範囲（0〜255）を付けている。フェーズ1で使った `ros2 param get`・`set` に加えて、`describe` で説明と範囲を読み、範囲の外の値を入れてみる。

```bash
# T1
ros2 run turtlesim turtlesim_node

# T2
ros2 param list /turtlesim

ros2 param describe /turtlesim background_r

ros2 param describe /turtlesim holonomic

ros2 param set /turtlesim background_r 300

ros2 param set /turtlesim background_r 150
```

**期待する結果**（T2の分）:

```text
$ ros2 param list /turtlesim
  background_b
  background_g
  background_r
  holonomic
  qos_overrides./parameter_events.publisher.depth
  qos_overrides./parameter_events.publisher.durability
  qos_overrides./parameter_events.publisher.history
  qos_overrides./parameter_events.publisher.reliability
  start_type_description_service
  use_sim_time

$ ros2 param describe /turtlesim background_r
Parameter name: background_r
  Type: integer
  Description: Red channel of the background color
  Constraints:
    Min value: 0
    Max value: 255
    Step: 1

$ ros2 param describe /turtlesim holonomic
Parameter name: holonomic
  Type: boolean
  Description: If true, then turtles will be holonomic
  Constraints:

$ ros2 param set /turtlesim background_r 300
Setting parameter failed: Parameter {background_r} doesn't comply with integer range.

$ ros2 param set /turtlesim background_r 150
Set parameter successful
```

- `background_r` の `Description` が説明文、`Constraints` の下の `Min value`・`Max value`・`Step` が値の範囲で、どちらも `ParameterDescriptor` に書かれた内容である。
- `holonomic` は説明文だけを付けていて、範囲は付けていないので、`Constraints:` の下が空になる。
- 範囲の外の300は `Setting parameter failed` で拒否され、背景色は変わらない。範囲の中の150は `Set parameter successful` になり、turtlesimのウィンドウの背景が紫がかった色に変わる。
- `ros2 param describe /turtlesim ` まで打ってTabキーを2回押すと、`background_b`・`background_g` などの候補が並ぶ（T1のturtlesimを止めると、候補は出なくなる）。

### 5-4. 起動しなくても分かるようにする

利用者が起動する前に知りたい情報は、次のような形で知らせる。最初の項目はlaunchファイル（複数のノードをまとめて起動する設定のファイル）を使うので、フェーズ4（[`docs/phase4_launch.md`](phase4_launch.md)）の後に読むと分かりやすい。

- **launchの引数として見せる**: 利用者に変えてほしい値を、launchファイルの `DeclareLaunchArgument` に説明と既定値を付けて宣言する。`ros2 launch パッケージ ファイル --show-args` で、起動せずに名前・説明・既定値の一覧を表示できる（フェーズ4（[`docs/phase4_launch.md`](phase4_launch.md)）の4-1節）。配布するときは、ノードを直接 `ros2 run` してもらうのではなく、launchファイルを入口にするのが一般的な形である。
- **既定値のYAMLファイルを同梱する**: すべてのパラメータと既定値を並べたYAMLを、パッケージの `config/` に置き、コメントで説明や範囲を書いておく。利用者は、これを写して書き換える（YAMLでの指定のしかたは、フェーズ3-3の5-4節）。インストールしておけば、`ros2 pkg prefix パッケージ` で示される場所の `share/パッケージ/config/` で見つけられる。
- **定義ファイルからコードと説明書を作る**: 外部のライブラリ `generate_parameter_library`（PickNik社のOSS。Jazzy向けのaptパッケージは `ros-jazzy-generate-parameter-library`）を使うと、パラメータの名前・型・既定値・説明・制約をYAMLに1回書くだけで、宣言と値の検査をするC++/Pythonのコードと、説明書（Markdown等）を生成できる。コードと説明書の内容がずれないのが利点で、ros2_controlなどが採用している。

### 5-5. 使い分けの目安

| 方法 | 起動しなくても分かるか | 向いている場面 |
|---|---|---|
| `ParameterDescriptor`＋`ros2 param describe` | 分からない（起動中のノードに問い合わせる） | どのノードでも、まず付けておく。範囲の検査も兼ねる |
| launchの引数＋`--show-args` | 分かる | 利用者に変えてほしい値が、少数に絞れるとき |
| 既定値のYAMLの同梱 | 分かる（ファイルを読む） | パラメータが多く、まとめて書き換えてもらうとき |
| `generate_parameter_library` | 分かる（生成した説明書を読む） | パラメータが多く、説明書との食い違いを防ぎたいとき |

小さなパッケージなら、上の3つ（`ParameterDescriptor`、launchの引数、YAML）で足りる。

## 6. ROS2の通信の層を詳しく

フェーズ0（[`docs/phase0_overview.md`](phase0_overview.md)）の2節では、ROS2の通信が層を積み重ねた構造（プロトコルスタック）になっていることだけを説明した。ここでは、図に出てきた各層と、関係する仕組みの名前を補う。

- **DDS（Data Distribution Service）**: OMG（Object Management Group）という標準化団体が定めた、分散型のpublish/subscribe通信の規格。ROS2は、このDDS（正確には、その通信規約のDDS/RTPS）を標準の通信層として使っている。
- **RMW（ROS Middleware Interface）**: ROS2のAPIと、DDSの実装製品をつなぐ層。この層のおかげで、ROS2のコードを変えずにDDSの実装を差し替えられる。ROS2と一緒に入るのは、既定のFast DDS向けのRMW（`rmw_fastrtps_cpp`）だけである。ROS2のaptのリポジトリには、Cyclone DDS・RTI Connext・GurumDDS向けのRMWのパッケージもあり、使う実装のパッケージを別に導入してから、環境変数 `RMW_IMPLEMENTATION` で切り替える。RTI ConnextとGurumDDSは企業の商用製品で、使う条件（ライセンス）は提供元が定めている。特にConnextは、RMWのパッケージを入れただけでは動かず、製品本体も別に導入する必要がある。
- **ディスカバリ（自動発見）**: 各ノードは起動すると、マルチキャストでお互いを見つけ合う。同じネットワークにある別のシステムと混ざらないよう、`ROS_DOMAIN_ID`（ドメインID）の番号で通信の範囲を分ける。見つけた相手と何を確かめ合っているかは、この資料の4節で説明した。
- **QoS（Quality of Service）**: トピックごとに、信頼性（reliability: 送りっぱなしか、確実に届けるか）、持続性（durability: 後から参加した購読側に過去のデータを渡すか）、履歴（history: 直近の何件を手元に残すか）等を決められる。配信側と購読側のQoSが噛み合わないと、つながらない（購読側が求める水準を、配信側が満たしているかで決まる）。実際に試すのはフェーズ3-2b（[`docs/phase3_2b_qos.md`](phase3_2b_qos.md)）。

## 7. ROS1との違い

ROS2は、初代のROS（ROS1）を作り直したものである。古い記事やサンプルにはROS1向けのものが多いので、読み分けるために違いの要点を知っておくと役に立つ。

ROS1では、`roscore`（マスター）という中央のプロセスがノードの名前の取りまとめを担い、通信にはROS独自の規約（TCPROS・XMLRPC）を使っていた。マスターが止まると新しい接続が作れなくなり、複数台のロボットの構成・セキュリティ・リアルタイム制御には、別の工夫が必要だった。

ROS2は、この反省から、通信の層に業界標準のDDS（この資料の6節）を採用し、**中央の管理者がいない分散型の構成**に作り直された。リアルタイム制御・組み込み機器・複数台のロボット・セキュリティを、最初から考えに入れた設計になっている。

ROS1の最後の版（Noetic）のサポートは2025年5月に終了しており、今の新規開発はROS2が前提である。記事やサンプルが `roscore`・`rosrun`・`catkin_make` を使っていたら、ROS1向けと見分けられる（ROS2では `ros2 run`・`colcon build` を使う）。

## 8. 名前空間（namespace）が効く範囲

### 8-1. 疑問: 「名前空間を付けても変わらない」とは、どこの話か

フェーズ3-2a（[`docs/phase3_2a_turtlesim.md`](phase3_2a_turtlesim.md)）の4節では、`'/turtle1/cmd_vel'` のように先頭に `/` を付けたトピック名は「ノードに名前空間を付けて起動しても、組み立てられるトピック名が変わらない」と書いた。これは、ノードの中でトピック名を組み立てるときの話である。

名前空間は、**ノードに付く属性**である。ノードがPublisherやSubscriberを作るとき、渡されたトピック名が相対名（先頭に `/` が無い名前）なら、ノードの名前空間を前に付けて、完全な名前にする。絶対名（先頭が `/`）なら、そのまま使う。この組み立ては、ノードの中（ROS2のクライアントライブラリ）で済んでしまう。

トピック通信の側が扱うのは、組み立て終わった完全な名前だけである。`/demo/chatter` は「`demo` という区画の中の `chatter`」ではなく、単に `/demo/chatter` という1つの長い名前として扱われる。だから、名前空間は通信を仕切る壁ではない。名前空間が違うノード同士でも、完全な名前が同じならつながる。通信の範囲そのものを分けたい場合は、名前空間ではなく `ROS_DOMAIN_ID`（この資料の6節）を使う。

### 8-2. 名前の3つの書き方

ノードの名前空間が `/demo`、ノード名が `talker` のとき、トピック名は次のように組み立てられる。

| 書き方 | 例 | 完全な名前 |
|---|---|---|
| 絶対名 | `/chatter` | `/chatter`（名前空間は付かない） |
| 相対名 | `chatter` | `/demo/chatter` |
| プライベート名 | `~/chatter` | `/demo/talker/chatter`（ノードの完全な名前の下） |

名前空間は、起動するときに外から付けられる。`ros2 run` なら `--ros-args -r __ns:=/demo`、launchなら `PushRosNamespace`（フェーズ4（[`docs/phase4_launch.md`](phase4_launch.md)）の4-6節）である。コードの側は、相対名で書いておけば、付けられた名前空間に従う。

### 8-3. 名前空間を意識しなければならない場面

**同じノードを複数動かすとき。** ロボットを2台動かすなら、それぞれのノードを `/robot1`・`/robot2` の名前空間で起動し、`/robot1/cmd_vel`・`/robot2/cmd_vel` に分けるのがふつうのやり方である。turtlesimの `/turtle1/cmd_vel` と `/turtle2/cmd_vel`（フェーズ1（[`docs/phase1_cli_turtlesim.md`](phase1_cli_turtlesim.md)）の3-5節で、`/spawn` で2匹目を出したときに増えるトピック）も、同じ発想で分かれている。このとき、コードの中のトピック名が絶対名だと、名前空間を付けても全員が同じトピックに集まり、指令が混ざる。複数動かす前提のノードは、相対名で書く。

**名前空間の違う相手とつなぐとき。** 3-2aの `turtle_circle` は、相手のturtlesimが待っている `/turtle1/cmd_vel` に確実に届けるため、絶対名で書いている。ところが、こう書いたノードは、名前空間を付けるだけでは送り先が変わらないので、ロボットごとに名前空間で分ける使い方ができない。別の亀やロボットに向け直すには、remap（トピック名の付け替え。フェーズ4の4-6節）を使う（remapは絶対名にも効く）。複数台で使い回す前提なら、相対名で書いておくと、名前空間を付けるだけで分けられる。どちらにしても、つながるかどうかは、両側の完全な名前が一致するかで決まる。

**コマンドから触るとき。** `ros2 topic echo`・`ros2 param get` などは、完全な名前で指定する（`ros2 param get /demo/talker use_sim_time` のように）。`ros2 topic pub chatter ...` のように相対名で書くと、コマンドの側は名前空間の外（`/`）にいるので `/chatter` に送ってしまい、`/demo/chatter` を待つノードには届かない。`ros2 topic list` で完全な名前を確かめてから指定するとよい。

**パラメータのYAMLファイルを渡すとき。** この段落と8-4節は、パラメータをYAMLファイルで渡す方法（フェーズ3-3（[`docs/phase3_3_parameters.md`](phase3_3_parameters.md)）の5-4節）を読んだ後のほうが分かりやすい。YAMLの先頭に書くノード名は、名前空間を含めた完全な名前と照らし合わされる。`talker:` と書いたファイルは、名前空間の無い `/talker` にしか効かない。名前空間を付けて起動したノードには、`/demo/talker:` のように名前空間まで書くか、すべてのノードに当てはまる `/**:` を使う。この違いはエラーにならず、値が黙って既定値のままになるので気づきにくい。

### 8-4. 確かめる: YAMLのノード名と名前空間

ROS2に付属するデモのtalker（`demo_nodes_cpp`）を、名前空間 `/demo` で起動し、`use_sim_time: true` を書いたYAMLを渡す。ノード名の書き方だけが違う、次の3つのファイルを用意する（置き場所はどこでもよいが、T1はそのディレクトリで実行する。別の場所から実行する場合は、`--params-file` に絶対パスで渡す）。

```yaml
# talker.yaml
talker:
  ros__parameters:
    use_sim_time: true
```

```yaml
# demo_talker.yaml
/demo/talker:
  ros__parameters:
    use_sim_time: true
```

```yaml
# all.yaml
/**:
  ros__parameters:
    use_sim_time: true
```

T1で1つめのファイルを渡して起動し、T2で確かめる。確かめたら、T1を `Ctrl+C` で止め、残りのファイルでも同じことを繰り返す。

```bash
# T1
ros2 run demo_nodes_cpp talker --ros-args -r __ns:=/demo --params-file talker.yaml

# T2
ros2 topic list

ros2 param get /demo/talker use_sim_time
```

**期待する結果**（T2。`talker.yaml` を渡した場合。`ros2 topic list` は抜粋）:

```text
$ ros2 topic list
/demo/chatter
$ ros2 param get /demo/talker use_sim_time
Boolean value is: False
```

トピックが `/demo/chatter` になっていれば、名前空間が付いている。`use_sim_time` は `False` のままで、`talker:` と書いたYAMLの値が効いていない。`demo_talker.yaml` か `all.yaml` を渡した場合は、同じコマンドで `Boolean value is: True` になる。Python版のデモ（`ros2 run demo_nodes_py talker ...`）でも、結果は同じである。

## 9. 参考資料

確認状況（2026-09-25）: この資料の内容は、WSLに導入済みのROS2 Jazzy（`/opt/ros/jazzy`）のソースと、Ubuntuのパッケージのソースを読んで確かめた。下記のURLは、この資料の作成時に実在をWeb検索で確かめたが、本文は読み直していない（4つ目の記事は、3-1節で引用した警告文の続き（引用では省略した部分）に示されているもの）。

- [Using the ros2 param command-line tool — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Using-ros2-param.html)（`use_sim_time` などのパラメータの扱い）
- [ament_cmake user documentation — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Ament-CMake-Documentation.html)（`ament_target_dependencies` などの `ament_` の関数）
- [CMake Documentation — cmake-buildsystem(7)](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html)（ターゲットと `PUBLIC`/`PRIVATE`/`INTERFACE` の考え方）
- [Why you shouldn't invoke setup.py directly（Paul Ganssle）](https://blog.ganssle.io/articles/2021/10/setup-py-deprecated.html)（setuptoolsの警告文に出てくる解説記事）

> 出典: 文章は自分の言葉で書いたもので、上記のドキュメントやソースの転載ではない。`CMakeLists.txt` の各行と警告の文面は、ROS2の雛形（Apache 2.0）とsetuptoolsの出力を、説明に必要な範囲で引用した。
