# Tips集: 手順書の本文に入れなかった小さな補足

手順書（フェーズ）の本文に入れると流れが途切れる、分量の少ない補足をまとめた資料。どの項目も独立しているので、関係する手順書を読んでいるときや、疑問を持ったときに、その項目だけを読めばよい。

- 対象: ROS2 Jazzy（Ubuntu 24.04）。版に依存する記述は、この資料の作成時に `/opt/ros/jazzy` とUbuntuのパッケージに入っていたもので確かめた
- 所要目安: 1項目あたり5〜15分（読み物）

> **実行環境が無くても読めるように**: コマンドを載せた箇所には、その直後に「期待する結果」として表示の例と読み方を書いている。ノードを起動した後の表示（1-4節）は、ROS2の仕様から想定したもの。3-1節の警告の文面は、使い捨ての環境で実際に表示させたもの。

> **出どころ**: 内容は、`/opt/ros/jazzy` のソース（`rclpy`・`rclcpp`）、`ros2 pkg create` が作る雛形、Ubuntuのパッケージ（`colcon-core`・`setuptools`）を読んで確かめ、自分の言葉で書いた。ソースやドキュメントの転載ではない。

## 項目の一覧

| 節 | 項目 | 関係する手順書 |
|---|---|---|
| 1 | シミュレーション時刻（`use_sim_time`） | フェーズ1（パラメータの一覧）、フェーズ3-1（時刻の取り方）、フェーズ5-0（Gazebo） |
| 2 | `CMakeLists.txt` の読み方（モダンCMakeの要点） | フェーズ2の2-3節以降、C++のサンプルすべて |
| 3 | ビルド中の `SetuptoolsDeprecationWarning` の意味 | フェーズ2の2-4節 |

## 1. シミュレーション時刻（`use_sim_time`）

### 1-1. 時計が2つある

フェーズ1の `ros2 param list` や、フェーズ3-3の `ros2 param list` には、自分で宣言していない `use_sim_time` というパラメータが並んでいた。これは「ノードの時計を、現実の時刻にするか、シミュレーションの時刻にするか」を切り替えるスイッチで、すべてのノードが自動で持っている（`Node` が作られるときに用意される。参考資料 [`docs/reference_node_class.md`](reference_node_class.md) の2節）。

- **現実の時刻**（`use_sim_time` が `false`。既定）: PCの時計そのもの。1秒は現実の1秒。
- **シミュレーションの時刻**（`use_sim_time` が `true`）: シミュレータが「今、シミュレーションの中では何秒か」を知らせてくる時刻。シミュレータを一時停止すれば止まり、PCが重ければ現実よりゆっくり進む。

フェーズ5-0の2-1節では、Gazeboが一時停止の状態で始まること、右下のRTF（シミュレーションの時間が現実の何倍の速さで進んでいるか）が100%を下回ることがあることを見た。このとき、シミュレーションの中の車両は「シミュレーションの時刻」で動いている。フェーズ5-0の4節で見たオドメトリの `header.stamp` も、Gazeboを起動してからのシミュレーションの時刻だった。

### 1-2. なぜ合わせる必要があるのか

制御のノードが現実の時刻で動き、車両がシミュレーションの時刻で動いていると、2つの時計がずれる。たとえばRTFが50%のとき、制御のノードが「1秒たったので速度を上げる」と判断しても、シミュレーションの中ではまだ0.5秒しかたっていない。PI制御の積分のように「経過時間 × 値」を足していく計算では、このずれがそのまま誤差になる。一時停止中も制御のノードの時計だけが進み、再生したとたんに大きな指令を出す、ということも起こる。

`use_sim_time` を `true` にすると、ノードの時計がシミュレーションの時刻に合うので、PCの速さに関係なく同じ結果になる。記録したデータ（rosbag）を再生して、ノードを当時と同じ時刻で動かすときにも同じ仕組みを使う。

### 1-3. 仕組み: `/clock` トピック

シミュレーションの時刻は、`/clock` というトピック（型は `rosgraph_msgs/msg/Clock`）で配られる。`use_sim_time` が `true` のノードは、このトピックを自動で購読し、届いた時刻を自分の時計の現在時刻にする。rclpyのソース（`rclpy/time_source.py`）では、この購読のQoSは `best_effort`・depth 1になっている（最新の時刻だけが大事なので、取りこぼしても再送しない）。

Gazeboの場合、時刻はまずGazeboの側の `/clock` に出る（フェーズ5-0の2-1節の `gz topic -l` の一覧にある）。ROS2のノードに届けるには、ほかのトピックと同じくブリッジが必要である。

```text
/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock
```

`parameter_bridge` の引数にこの1本を足すと、Gazeboの時刻がROS2の `/clock` に中継される。区切りの `[` は「Gazebo→ROS2の片方向」の意味（フェーズ5-0の3-1節）。

フェーズ5-0で使ったデモ（`diff_drive.launch.py`）は、3-1節の `--print` の表示のとおり、`cmd_vel` と `odometry` の4本しか中継しておらず、`/clock` は中継していない。したがって、フェーズ5-0の課題6（10秒走ったら止まるノード）の `self.get_clock().now()` は、現実の時刻で測っている。

### 1-4. 切り替え方

実行時に、ほかのパラメータと同じ書き方で渡す（フェーズ3-3の5-2節の `-p`）。

```bash
ros2 run learn_py talker --ros-args -p use_sim_time:=true
```

launchファイルでは、ノードに渡すパラメータに `{'use_sim_time': True}` を加える（フェーズ4のパラメータの渡し方と同じ）。起動中のノードの設定は `ros2 param get` で確かめられる。

```bash
ros2 param get /talker use_sim_time
```

**期待する結果**（書式はフェーズ1の `ros2 param get` と同じ）:

```text
Boolean value is: True
```

`True` なら、そのノードの時計はシミュレーションの時刻を使っている。

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

C++のサンプルを足すたびに書いた `add_executable`・`ament_target_dependencies`・`install(TARGETS ...)` の3点セットは、この表の同じ行の繰り返しである。

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
- **依存は3か所。** C++では、`package.xml` の `<depend>`、`find_package`、`ament_target_dependencies` の3か所に依存を書く（フェーズ3-2aの「Python版とC++版の違いのまとめ」）。`find_package` は「探して使える状態にする」、`ament_target_dependencies` は「どのターゲットがそれを使うか」で、役割が違う。
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

## 4. 参考資料

確認状況（2026-09-25）: この資料の内容は、WSLに導入済みのROS2 Jazzy（`/opt/ros/jazzy`）のソースと、Ubuntuのパッケージのソースを読んで確かめた。下記のURLは、この資料の作成時に実在をWeb検索で確かめたが、本文は読み直していない（4つ目の記事は、3-1節の警告文の中に示されているもの）。

- [Using the ros2 param command-line tool — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Using-ros2-param.html)（`use_sim_time` などのパラメータの扱い）
- [ament_cmake user documentation — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Ament-CMake-Documentation.html)（`ament_target_dependencies` などの `ament_` の関数）
- [CMake Documentation — cmake-buildsystem(7)](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html)（ターゲットと `PUBLIC`/`PRIVATE`/`INTERFACE` の考え方）
- [Why you shouldn't invoke setup.py directly（Paul Ganssle）](https://blog.ganssle.io/articles/2021/10/setup-py-deprecated.html)（setuptoolsの警告文に出てくる解説記事）

> 出典: 文章は自分の言葉で書いたもので、上記のドキュメントやソースの転載ではない。`CMakeLists.txt` の各行と警告の文面は、ROS2の雛形（Apache 2.0）とsetuptoolsの出力を、説明に必要な範囲で引用した。
