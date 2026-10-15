# チートシート: この教材で使うROS2のコマンドの早見表

この教材の手順書に出てくるコマンドを、目的ごとに1か所に集めた早見表。「あのコマンドの書き方は何だったか」を思い出したいときに引く。例には、この教材で作るパッケージ（`learn_py` など）・実行ファイル・トピックの名前をそのまま使っている。自分のパッケージで使うときは、名前を置き換える。

- 対象: ROS2 Jazzy（Ubuntu 24.04）と、Gazebo Harmonic
- 並び: おおよそ手順書で出てくる順（ターミナルの準備 → ビルド → 起動 → 調べる → 記録 → Gazebo）。最後の14節に、この教材の独自の名前の一覧を置いた

> **この資料の位置づけ**: 一覧のための資料なので、コマンドの表示（期待する結果）と、その読み方は載せていない。それらは、各行の「出てくる手順書」の節に書いてある。例は、手順書のコマンドをそのまま写した。ただし、手順書が書き方だけを示しているもの（`ros2 topic type /トピック名`）と、手順書ではわざと誤った値を渡しているもの（`target_generator` の配列のパラメータ）は、この教材の正しい値を入れた。その旨は、各行に書き添えた。初めて使うコマンドは、手順書の側で、表示と読み方を見てから使う。オプションの意味は、手元の `--help`（例: `ros2 topic echo --help`）で確かめた。

## 0. 表の見方

- 「コマンド（この教材の例）」の列は、そのまま打てる形で書いた。`<...>` は、自分で決める値（パッケージ名など）を表す。
- 「出てくる手順書」の列は、そのコマンドを最初に、または主に使う手順書。任意の章（間章・C++版など）だけで使うものは、そう書き添えた。
- 長く動き続けるコマンド（ノードの起動・`echo`・`hz` など）は、Ctrl+Cで止める。

## 1. ターミナルを開いたら（環境の準備）

ROS2のコマンドは、**ターミナルごとに**環境を読み込んでから使う。環境は、ターミナルを閉じると消える。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| ROS2本体を読み込む（`~/.bashrc` に書いておく） | `source /opt/ros/jazzy/setup.bash` | [環境構築](setup_wsl2_ros2.md)の4節 |
| 通信の範囲を自分のPCの中に限る（`~/.bashrc` に書いておく） | `export ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST` | [フェーズ1](phase1_cli_turtlesim.md)の1-3節 |
| 上の設定が効いているかを確かめる | `echo $ROS_AUTOMATIC_DISCOVERY_RANGE` | [フェーズ1](phase1_cli_turtlesim.md)の1-3節 |
| 自分でビルドしたパッケージを読み込む（ビルドした後、新しいターミナルごと） | `cd ~/ros2_ws` の後に `source install/setup.bash` | [フェーズ2](phase2_packages.md)の2-5節 |
| 強化学習の仮想環境を有効にする（フェーズ7） | `source ~/rl_venv/bin/activate` | [強化学習の環境構築](setup_rl_sb3.md)の2節 |
| 仮想環境から抜ける | `deactivate` | [強化学習の環境構築](setup_rl_sb3.md) |

`source install/setup.bash` は、**ビルドした後に開いたターミナル**でも、**ビルドする前から開いていたターミナル**でも必要になる。新しいパッケージや実行ファイルを足したのに `ros2 run` が見つけられないときは、まずこれを疑う（13節）。また、`colcon build` は、強化学習の仮想環境を無効にしたターミナルで行う（[強化学習の環境構築](setup_rl_sb3.md)の5節。有効にしたままビルドすると、ノードの起動用のスクリプトが仮想環境のPythonを指すことがある）。

## 2. パッケージを作って、ビルドする

**ワークスペース**は、自作のパッケージを集めてビルドする作業用のフォルダ（この教材では `~/ros2_ws`）。パッケージは `~/ros2_ws/src` の下に作り、ビルドは `~/ros2_ws` で行う。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| Pythonのパッケージを作る（`~/ros2_ws/src` で） | `ros2 pkg create --build-type ament_python --license Apache-2.0 --maintainer-name learner --maintainer-email noreply@example.com --dependencies rclpy std_msgs --node-name hello learn_py` | [フェーズ2](phase2_packages.md)の2-2節 |
| C++のパッケージを作る（同上） | `ros2 pkg create --build-type ament_cmake --license Apache-2.0 --maintainer-name learner --maintainer-email noreply@example.com --dependencies rclcpp std_msgs --node-name hello learn_cpp` | [フェーズ2](phase2_packages.md)の2-3節 |
| launchファイル・設定ファイル置き場のパッケージを作る | `ros2 pkg create --build-type ament_cmake --license Apache-2.0 --maintainer-name learner --maintainer-email noreply@example.com learn_bringup` | [フェーズ4](phase4_launch.md)の3-1節 |
| すべてのパッケージをビルドする（`~/ros2_ws` で） | `colcon build --symlink-install` | [フェーズ2](phase2_packages.md)の2-4節 |
| 1つだけビルドする | `colcon build --symlink-install --packages-select learn_py` | [フェーズ3-1](phase3_1_pubsub.md)の3-3節 |
| 2つ以上を選んでビルドする | `colcon build --symlink-install --packages-select learn_py learn_bringup` | [フェーズ5-3](phase5_3_launch.md)の3-4節 |
| ビルドの結果を全部消して、やり直す | `rm -rf build install log`（`~/ros2_ws` で。消してから `colcon build`） | [フェーズ2](phase2_packages.md)の2-7節の課題4 |
| 1つのパッケージのビルドの結果だけを消す | `rm -rf build/learn_cpp install/learn_cpp` | [フェーズ3-1](phase3_1_pubsub.md)のつまずきやすい点 |
| 入っているパッケージを探す | `ros2 pkg list \| grep learn` | [フェーズ2](phase2_packages.md)の2-5節 |
| パッケージの実行ファイルの一覧を見る | `ros2 pkg executables learn_py` | [フェーズ2](phase2_packages.md)の2-5節 |
| パッケージが入っている場所を見る | `ros2 pkg prefix learn_py` | [フェーズ2](phase2_packages.md)の2-5節 |

- `--symlink-install` は、Pythonのソースやlaunchファイルを、`install/` に写さずリンクで置くオプション。Pythonのソースを直したときに、ビルドし直さなくても変更が効く（[フェーズ2](phase2_packages.md)の2-7節）。ただし、新しいファイル（ソース・launchファイル・YAMLなど）を足したとき、実行ファイルを足したとき（`setup.py` の `entry_points` を直したとき）、C++のソースを直したときは、ビルドし直す（[フェーズ4](phase4_launch.md)の3-3節）。
- `ros2 pkg create` の `--maintainer-name`・`--maintainer-email` は、必ず付ける。省くと、Gitの設定などから実際のメールアドレスが `package.xml` に入るおそれがある（[フェーズ2](phase2_packages.md)の2-2節）。値は、手順書と同じダミーの `learner`・`noreply@example.com` にしてある。
- `rm -rf` は、消したものを戻せない。`~/ros2_ws` にいることを確かめてから打つ（`build`・`install`・`log` は、ビルドすれば作り直されるので、消してよい）。

## 3. ノードを起動する（`ros2 run`）

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| ノードを起動する | `ros2 run learn_py talker` | [フェーズ3-1](phase3_1_pubsub.md)の3-3節 |
| ROS2に付いてくるノードを起動する | `ros2 run turtlesim turtlesim_node` | [フェーズ1](phase1_cli_turtlesim.md)の3-1節 |
| パラメータを渡して起動する | `ros2 run learn_py param_talker --ros-args -p message:="from cli" -p period:=0.5` | [フェーズ3-3](phase3_3_parameters.md)の5-2節 |
| パラメータをYAMLのファイルで渡して起動する | `ros2 run learn_py param_talker --ros-args --params-file ~/ros2_ws/config/param_talker.yaml` | [フェーズ3-3](phase3_3_parameters.md)の5-4節 |
| 配列のパラメータを渡す（引用符で囲む。値は既定値と同じもの） | `ros2 run learn_py target_generator --ros-args -p "step_times:=[10.0, 50.0, 80.0]" -p "step_values:=[10.0, 5.0, 0.0]"` | [フェーズ5-3](phase5_3_launch.md)の3-4節（手順書では、わざと誤った値を渡して、エラーになることを確かめている） |
| ノードの名前を変えて起動する | `ros2 run turtlesim turtlesim_node --ros-args --remap __node:=my_turtle` | [フェーズ1](phase1_cli_turtlesim.md)の3-3節 |
| 名前空間を付けて、YAMLのファイルも渡して起動する | `ros2 run demo_nodes_cpp talker --ros-args -r __ns:=/demo --params-file talker.yaml` | [Tips集](tips.md)の8-4節 |
| シミュレーション時刻で動かす | `ros2 run learn_py talker --ros-args -p use_sim_time:=true` | [Tips集](tips.md)の1-4節 |
| ROS2のノードでないPythonのモジュールを動かす | `python3 -m learn_py.closed_loop_sim` | [フェーズ5-2](phase5_2_pi.md)の4-2節 |

`--ros-args` の後ろに書いたものは、プログラムではなく、ROS2に向けた指定になる。`-p 名前:=値` がパラメータ、`-r 古い名前:=新しい名前`（`--remap` と同じ）が名前の付け替え（リマップ）である。`__node`・`__ns` は、ノードの名前と名前空間を表す特別な名前。

## 4. 動いているものを調べる（ノードとトピック）

**トピック**は、ノードの間で一方向に流れ続けるデータの通り道。ノードを起動した状態で、別のターミナルから調べる。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| ノードの一覧 | `ros2 node list` | [フェーズ1](phase1_cli_turtlesim.md)の3-3節 |
| ノードが持つトピック・サービス・アクションを見る | `ros2 node info /turtlesim` | [フェーズ1](phase1_cli_turtlesim.md)の3-3節 |
| トピックの一覧（型も） | `ros2 topic list -t` | [フェーズ1](phase1_cli_turtlesim.md)の3-4節 |
| トピックの型と、送る側・受ける側の数 | `ros2 topic info /turtle1/cmd_vel` | [フェーズ1](phase1_cli_turtlesim.md)の3-4節 |
| 送る側・受ける側の詳細（QoSも） | `ros2 topic info /chatter -v` | [フェーズ3-1](phase3_1_pubsub.md)の5節、[フェーズ3-2b](phase3_2b_qos.md)の5-1節 |
| トピックの型だけ | `ros2 topic type /chatter` | [参考資料: 標準のデータ型とインターフェース](reference_interfaces.md)の4節（書き方だけを示している。`/chatter` はフェーズ3-1のトピック） |
| 流れている中身を見続ける | `ros2 topic echo /sine` | [フェーズ3-1](phase3_1_pubsub.md)の5節 |
| 1回だけ見る | `ros2 topic echo --once /model/vehicle_green/odometry` | [フェーズ5-0](phase5_0_gazebo.md)の3-4節 |
| 中身の一部の項目だけ見る | `ros2 topic echo --once /model/vehicle_green/odometry --field twist.twist.linear.x` | [フェーズ5-1](phase5_1_plant.md)の6-2節 |
| 送られてくる周期を測る | `ros2 topic hz /turtle1/pose` | [フェーズ1](phase1_cli_turtlesim.md)の3-4節 |
| 手で1回だけ送る | `ros2 topic pub --once /plant/pedal std_msgs/msg/Float64 "{data: 0.5}"` | [フェーズ5-1](phase5_1_plant.md)の6-1節 |
| 手で送り続ける（`-r` は1秒あたりの回数） | `ros2 topic pub -r 1 /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.8}}"` | [フェーズ1](phase1_cli_turtlesim.md)の3-4節 |
| すべての項目を既定値（0）で送る（止める） | `ros2 topic pub --once /model/vehicle_green/cmd_vel geometry_msgs/msg/Twist "{}"` | [フェーズ5-0](phase5_0_gazebo.md)の3-3節 |

`pub` に渡す中身は、YAMLの書き方（`{項目: 値}`）で書き、全体を `"` で囲む。書かなかった項目は既定値（数なら0、文字列なら空、真偽値なら `false`）になる。中身の項目の名前は、7節の `ros2 interface show` で確かめる。

## 5. サービスとアクション

**サービス**は、1回の要求に1回の応答を返す通信。**アクション**は、時間のかかる処理を頼み、途中経過（フィードバック）を受け取りながら結果を待つ通信。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| サービスの一覧（型も） | `ros2 service list -t` | [フェーズ1](phase1_cli_turtlesim.md)の3-5節 |
| サービスの型 | `ros2 service type /clear` | [フェーズ1](phase1_cli_turtlesim.md)の3-5節 |
| サービスを呼ぶ（要求が空の型） | `ros2 service call /reset_counter std_srvs/srv/Trigger` | [フェーズ3-4](phase3_4_services.md)の4-4節（任意） |
| サービスを呼ぶ（要求に値がある型） | `ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 10, b: 20}"` | [フェーズ3-4](phase3_4_services.md)の6-1節（任意） |
| アクションの一覧（型も） | `ros2 action list -t` | [フェーズ1](phase1_cli_turtlesim.md)の3-7節 |
| アクションのサーバとクライアントを見る | `ros2 action info /fibonacci` | [フェーズ3-5](phase3_5_actions.md)の6-1節（任意） |
| ゴールを送り、途中経過も表示する | `ros2 action send_goal /fibonacci example_interfaces/action/Fibonacci "{order: 5}" --feedback` | [フェーズ3-5](phase3_5_actions.md)の6-1節（任意） |

## 6. パラメータ

**パラメータ**は、ノードが外から変えられるようにしている設定値。ノードを起動した状態で、別のターミナルから扱う。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| 全ノードのパラメータの一覧 | `ros2 param list` | [フェーズ1](phase1_cli_turtlesim.md)の3-6節 |
| 1つのノードのパラメータの一覧 | `ros2 param list /param_talker` | [フェーズ3-3](phase3_3_parameters.md)の5-3節 |
| 値を読む | `ros2 param get /pi_controller kp` | [フェーズ5-2](phase5_2_pi.md)の6-1節 |
| 値を変える（動いている間に） | `ros2 param set /vehicle_plant tau_accel 3.0` | [フェーズ5-1](phase5_1_plant.md)の6-3節 |
| 型・説明・範囲を見る | `ros2 param describe /param_talker period` | [フェーズ3-3](phase3_3_parameters.md)の5-3節 |
| 今の値をすべてYAMLの形で表示する | `ros2 param dump /pi_controller` | [フェーズ5-2](phase5_2_pi.md)の6-3節 |
| YAMLのファイルの値をまとめて設定する | `ros2 param load /param_talker ~/ros2_ws/config/param_talker.yaml` | [フェーズ3-3](phase3_3_parameters.md)の5-4節 |

`ros2 param dump` の表示をファイルに保存すれば、そのまま `--params-file`（3節）や `ros2 param load` に渡せるYAMLになる。

## 7. インターフェース（型）を調べる

**インターフェース**は、トピック・サービス・アクションでやり取りするデータの型（`std_msgs/msg/Float64` など）。名前は「パッケージ/種類/型名」の形で、種類は `msg`（メッセージ）・`srv`（サービス）・`action`（アクション）。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| 型の中身（項目の名前と型）を見る | `ros2 interface show geometry_msgs/msg/Twist` | [フェーズ1](phase1_cli_turtlesim.md)の3-4節 |
| コメントを除いて短く見る | `ros2 interface show nav_msgs/msg/Odometry --no-comments` | [参考資料: 標準のデータ型とインターフェース](reference_interfaces.md)の2-4節 |
| `pub` に貼れる形のひな形を出す | `ros2 interface proto geometry_msgs/msg/Twist` | [参考資料: 標準のデータ型とインターフェース](reference_interfaces.md)の4節 |
| パッケージに入っている型の一覧 | `ros2 interface package std_srvs` | [参考資料: 標準のデータ型とインターフェース](reference_interfaces.md)の4節 |

## 8. launch（まとめて起動する）

**launchファイル**は、複数のノードと、その設定をまとめて起動するためのファイル。この教材では、`learn_bringup` パッケージの `launch/` に置いている。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| launchファイルで起動する | `ros2 launch learn_bringup pubsub.launch.py` | [フェーズ4](phase4_launch.md)の4-1節 |
| 引数を渡して起動する | `ros2 launch learn_bringup vehicle_sim.launch.py gazebo:=false` | [フェーズ5-3](phase5_3_launch.md)の4-1節 |
| 受け付ける引数と既定の値を見る（起動しない） | `ros2 launch learn_bringup vehicle_sim.launch.py --show-args` | [フェーズ4](phase4_launch.md)の4-1節、[フェーズ5-3](phase5_3_launch.md)の3-4節 |
| 何が起動されるかを表示する（起動しない） | `ros2 launch learn_bringup pubsub.launch.py --print` | [フェーズ4](phase4_launch.md)の4-1節 |
| XML・YAMLの形式のlaunchファイルの中身を表示する（起動しない） | `ros2 launch learn_bringup pubsub.launch.xml talker_lang:=cpp --print` | [フェーズ4](phase4_launch.md)の4-3節 |
| ファイルのパスを引数で渡す（`~` でなく `$HOME` を使う） | `ros2 launch learn_bringup record.launch.py bag:=$HOME/ros2_ws/bags/sim_default` | [フェーズ6-1](phase6_1_record.md)の4-5節 |

引数の書き方は `名前:=値` で、`ros2 run` と違って `--ros-args` は要らない。`--show-args` と `--print` は、ノードを起動せずに中身を確かめられるので、launchファイルを書いた直後の点検に向く。

## 9. 記録と再生（`ros2 bag`）

**bag**は、トピックに流れたデータを時刻ごと記録したもの（フォルダ1つ）。記録したデータは、後から何度でも同じように流し直せる。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| 選んだトピックを記録する（`-o` は保存するフォルダの名前） | `ros2 bag record -o manual --topics /target_velocity /plant/velocity /plant/pedal` | [フェーズ6-1](phase6_1_record.md)の2-1節 |
| 中身（時間・トピック・件数）を見る | `ros2 bag info manual` | [フェーズ6-1](phase6_1_record.md)の2-2節 |
| 再生する | `ros2 bag play manual` | [フェーズ6-1](phase6_1_record.md)の2-3節 |
| 5倍の速さで、止まった状態から再生する（スペースキーで開始） | `ros2 bag play sim_default -r 5 -p` | [フェーズ6-3](phase6_3_metrics.md)の6-1節 |
| 2つ目の記録を、5秒の位置から、トピックとノードの名前を付け替えて再生する（2つの記録を重ねるとき） | `ros2 bag play gz_default --start-offset 5 --remap /target_velocity:=/gz/target_velocity /plant/velocity:=/gz/plant/velocity /plant/pedal:=/gz/plant/pedal __node:=player_gz` | [フェーズ6-1](phase6_1_record.md)の6節 |
| launchで、決めた時間だけ記録する（この教材のlaunch） | `ros2 launch learn_bringup record.launch.py bag:=$HOME/ros2_ws/bags/sim_default` | [フェーズ6-1](phase6_1_record.md)の4-5節 |

`ros2 bag record` は、`-o` に書いたフォルダが既にあると記録を始めない。同じ名前で取り直すときは、古いフォルダを消すか、別の名前にする。

## 10. 可視化（rqt）

**rqt**は、ROS2の様子をウィンドウで見るための道具の集まり。どれもウィンドウを開くので、WSLでは、WSLg（WSLのGUIの仕組み）でウィンドウが出る。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| ノードとトピックのつながりを図で見る | `rqt_graph` | [フェーズ1](phase1_cli_turtlesim.md)の3-8節、[フェーズ3-1](phase3_1_pubsub.md)の5節 |
| ログを一覧で見る | `ros2 run rqt_console rqt_console` | [フェーズ1](phase1_cli_turtlesim.md)の3-8節 |
| 数値の変化をグラフで見る（項目は `/トピック/項目` で指定） | `ros2 run rqt_plot rqt_plot /target_velocity/data /plant/velocity/data /plant/pedal/data` | [フェーズ5-3](phase5_3_launch.md)の4-3節、[フェーズ6-1](phase6_1_record.md)の3-1節 |
| rqtの道具を1つの窓から選ぶ | `rqt` | [フェーズ1](phase1_cli_turtlesim.md)の1-2節 |

`rqt_console` と `rqt_plot` は、単独のコマンドとしては用意されていない（`/opt/ros/jazzy/bin` に、`rqt`・`rqt_graph` はあるが、この2つは無い）ので、`ros2 run` を通して起動する。

## 11. Gazebo（`gz` と、ROS2とのつなぎ）

`gz` で始まるコマンドは、ROS2ではなくGazeboのコマンドで、Gazeboの中のトピック（ROS2のトピックとは別もの）を扱う。ROS2の側から扱うには、`ros_gz_bridge` でつなぐ。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| Gazeboの版を見る | `gz sim --version` | [フェーズ5-0](phase5_0_gazebo.md)の1-3節 |
| ワールドを開く（一時停止の状態で始まる） | `gz sim diff_drive.sdf` | [フェーズ5-0](phase5_0_gazebo.md)の2-1節 |
| ワールドを開いて、すぐに動かし始める | `gz sim -r empty.sdf` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の4-4節（任意） |
| Gazeboのトピックの一覧 | `gz topic -l` | [フェーズ5-0](phase5_0_gazebo.md)の2-1節 |
| Gazeboのトピックに送る | `gz topic -t /model/vehicle_blue/cmd_vel -m gz.msgs.Twist -p "linear: {x: 0.5}, angular: {z: 0.3}"` | [フェーズ5-0](phase5_0_gazebo.md)の2-1節 |
| モデルの位置と姿勢を見る | `gz model -m box -p` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の2-2節（任意） |
| 物理を決めた歩数だけ進めて止める | `gz service -s /world/drop/control --reqtype gz.msgs.WorldControl --reptype gz.msgs.Boolean --timeout 5000 --req 'multi_step: 300, pause: true'` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の2-2節（任意） |
| SDFのファイルに誤りが無いかを確かめる | `gz sdf -k ~/ros2_ws/src/learn_bringup/worlds/vehicle_force.sdf` | [フェーズ5-4](phase5_4_gazebo_plant.md)の2-2節 |
| URDFをSDFに直した結果を表示する | `gz sdf -p arm.urdf` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の4-3節（任意） |
| Gazeboの車をROS2から動かすデモを起動する | `ros2 launch ros_gz_sim_demos diff_drive.launch.py rviz:=false` | [フェーズ5-0](phase5_0_gazebo.md)の3-1節 |
| GazeboのトピックをROS2へつなぐ | `ros2 run ros_gz_bridge parameter_bridge /clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の5-3節（任意） |
| ROS2のサービスで、物理を決めた歩数だけ進める | `ros2 service call /world/vehicle_rl/control ros_gz_interfaces/srv/ControlWorld "{world_control: {pause: true, multi_step: 1000}}"` | [フェーズ7-3](phase7_3_gazebo_env.md)の3-3節 |
| この教材のGazeboの一式を起動する | `ros2 launch learn_bringup gazebo_plant.launch.py` | [フェーズ5-4](phase5_4_gazebo_plant.md)の6-3節 |

`parameter_bridge` の引数は「トピック名@ROS2の型[Gazeboの型」の形で、間の記号が向きを表す。`[` はGazeboからROS2へ、`]` はROS2からGazeboへ、`@` は両方向である。

## 12. 任意の章で使うコマンド（ros2_control・コンポーネント）

最初のゴール（フェーズ5）にも最終ゴール（フェーズ7-5）にも要らない、任意の章でだけ使うコマンド。

| やりたいこと | コマンド（この教材の例） | 出てくる手順書 |
|---|---|---|
| xacroのファイルからロボットの説明を配る | `ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(xacro arm.urdf.xacro)"` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の4-4節 |
| 配られたロボットを、動いているGazeboに出す | `ros2 run ros_gz_sim create -topic robot_description -name arm` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の4-4節 |
| ros2_controlのコントローラを動かし始める | `ros2 run controller_manager spawner joint_state_broadcaster arm_position_controller` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の5-3節 |
| コントローラを止める | `ros2 run controller_manager unspawner arm_position_controller` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の5-4節 |
| 関節の目標の位置を送る | `ros2 topic pub --once /arm_position_controller/commands std_msgs/msg/Float64MultiArray "{data: [0.5, -1.0]}"` | [Gazeboの物理の間章](interlude_gazebo_physics.md)の5-3節 |
| コンポーネントを入れる入れ物（コンテナ）を起動する | `ros2 run rclcpp_components component_container` | [コンポーネントの間章](interlude_components.md)の7-1節 |
| 入れられるコンポーネントの一覧 | `ros2 component types` | [コンポーネントの間章](interlude_components.md)の6節 |
| コンテナにコンポーネントを入れる | `ros2 component load /ComponentManager learn_components learn_components::Talker` | [コンポーネントの間章](interlude_components.md)の7-1節 |
| 入っているコンポーネントの一覧 | `ros2 component list` | [コンポーネントの間章](interlude_components.md)の7-1節 |
| コンポーネントを取り出す（番号は `list` の表示） | `ros2 component unload /ComponentManager 1` | [コンポーネントの間章](interlude_components.md)の7-1節 |

## 13. 困ったときに打つコマンド

| 症状 | まず確かめること（コマンド） | 詳しい説明 |
|---|---|---|
| `Package 'learn_py' not found` | そのターミナルで `source install/setup.bash`（`~/ros2_ws` で）をしたか | [フェーズ2](phase2_packages.md)のつまずきやすい点 |
| `No executable found` | `ros2 pkg executables learn_py` に名前が出るか。出なければ、`setup.py` の `entry_points`（C++は `CMakeLists.txt` の `install(TARGETS ...)`）を足して、ビルドし直し、`source` し直す | [フェーズ3-1](phase3_1_pubsub.md)のつまずきやすい点、[Tips集](tips.md)の10節 |
| 名前を変えたのに、前の名前の実行ファイルも動く | `colcon` は `install/` の古いファイルを消さない。`rm -rf build/learn_cpp install/learn_cpp` の後にビルドし直す | [フェーズ3-1](phase3_1_pubsub.md)のつまずきやすい点 |
| `ros2 node list` が空、または一部のノードが見えない | すべてのターミナルで `echo $ROS_AUTOMATIC_DISCOVERY_RANGE` が同じ値か | [フェーズ1](phase1_cli_turtlesim.md)の1-3節 |
| トピックはあるのに、受け取れていない | `ros2 topic info <トピック名> -v` で、送る側と受ける側のQoSと、型が合っているか | [フェーズ3-2b](phase3_2b_qos.md)、[Tips集](tips.md)の4-2節 |
| パラメータの値が思ったものになっていない | `ros2 param get <ノード名> <パラメータ名>` で今の値を見る。YAMLのノード名と名前空間が合っているか | [フェーズ3-3](phase3_3_parameters.md)、[Tips集](tips.md)の8-4節 |

## 14. この教材の独自の名前の一覧

この教材で作るパッケージ・実行ファイル・launchファイルと、主なトピックの一覧。どの手順書で作ったものかを思い出すときに使う。

### 14-1. パッケージ

| パッケージ | 中身 | 作る手順書 |
|---|---|---|
| `learn_py` | Pythonのノードと、強化学習でも使う計算の部品（車両のモデル・PI制御・評価の指標） | [フェーズ2](phase2_packages.md)の2-2節 |
| `learn_cpp` | C++のノード（多くは任意のC++版） | [フェーズ2](phase2_packages.md)の2-3節 |
| `learn_bringup` | launchファイル・パラメータのYAML・Gazeboのワールド | [フェーズ4](phase4_launch.md)の3節 |
| `learn_interfaces` | 自作のメッセージの型（任意） | [参考資料: 標準のデータ型とインターフェース](reference_interfaces.md)の6節 |
| `learn_components` | コンポーネントの形のC++のノード（任意） | [コンポーネントの間章](interlude_components.md) |

### 14-2. `learn_py` の実行ファイル

フェーズ2〜3-5の実行ファイル（`hello` から `fibonacci_client` まで）は、`learn_cpp` にも、同じ名前のC++版がある（多くは任意）。フェーズ5-0以降のノードは、Python版だけである。

| 実行ファイル | 役割 | 作る手順書 |
|---|---|---|
| `hello` | 動作確認用の最小の実行ファイル | [フェーズ2](phase2_packages.md) |
| `talker`・`listener` | 文字列を送る・受けるノード | [フェーズ3-1](phase3_1_pubsub.md) |
| `sine_pub`・`sine_sub` | 正弦波の値を送る・受けるノード | [フェーズ3-1](phase3_1_pubsub.md) |
| `turtle_circle` | turtlesimの亀を円に走らせるノード | [フェーズ3-2a](phase3_2a_turtlesim.md) |
| `qos_talker`・`qos_listener` | QoSの組み合わせを試すノード | [フェーズ3-2b](phase3_2b_qos.md) |
| `param_talker` | パラメータで文言と周期を変えられるノード | [フェーズ3-3](phase3_3_parameters.md) |
| `add_server`・`add_client`・`counter_node` | サービスのサーバ・クライアントと、数を送りながら、サービスで0に戻せるノード | [フェーズ3-4](phase3_4_services.md) |
| `fibonacci_server`・`fibonacci_client` | アクションのサーバ・クライアント | [フェーズ3-5](phase3_5_actions.md) |
| `gz_drive` | Gazeboの車に速度の指令を送り、オドメトリの速度をログに出すノード | [フェーズ5-0](phase5_0_gazebo.md) |
| `vehicle_plant`・`gz_display` | 車両の疑似プラントと、その速度をGazeboの車に映すノード | [フェーズ5-1](phase5_1_plant.md) |
| `pi_controller` | PI制御で速度を目標に合わせるノード | [フェーズ5-2](phase5_2_pi.md) |
| `target_generator` | 目標の速度を、決めた時刻に切り替えて送るノード | [フェーズ5-3](phase5_3_launch.md) |
| `gz_plant` | Gazeboの物理で動かす車をプラントにするノード（launchから起動） | [フェーズ5-4](phase5_4_gazebo_plant.md) |
| `goal_monitor` | 目標に届いたかを見張るノード | [フェーズ6-2](phase6_2_goal.md) |
| `metrics` | 記録を再生して、評価の指標を計算するノード | [フェーズ6-3](phase6_3_metrics.md) |
| `nn_controller` | 学習したニューラルネットワークで制御するノード（launchから起動） | [フェーズ7-5](phase7_5_nn_node.md) |

### 14-3. `learn_bringup` のlaunchファイル

| launchファイル | 起動するもの | 作る手順書 |
|---|---|---|
| `pubsub.launch.py`（`.xml`・`.yaml` も） | `talker` と `listener` | [フェーズ4](phase4_launch.md) |
| `param.launch.py`・`turtle.launch.py`・`compose.launch.py` | `param_talker`（YAMLと引数を渡す例）、turtlesimと `turtle_circle`、ほかのlaunchファイルを名前空間を付けて読み込む例 | [フェーズ4](phase4_launch.md) |
| `components.launch.py` | コンテナとコンポーネント（任意） | [コンポーネントの間章](interlude_components.md) |
| `vehicle_sim.launch.py` | 疑似プラント・PI制御・目標・（Gazeboの表示） | [フェーズ5-3](phase5_3_launch.md) |
| `gazebo_plant.launch.py` | Gazeboの物理で動かす車と、PI制御の一式 | [フェーズ5-4](phase5_4_gazebo_plant.md) |
| `record.launch.py` | 上の一式に、決めた時間の記録を足したもの | [フェーズ6-1](phase6_1_record.md) |
| `record_goal.launch.py` | 記録に、目標に届いたかの見張りを足したもの | [フェーズ6-2](phase6_2_goal.md) |
| `gazebo_rl.launch.py` | 強化学習のためのGazeboの環境 | [フェーズ7-3](phase7_3_gazebo_env.md) |
| `gazebo_nn.launch.py` | 学習したニューラルネットワークで制御する一式 | [フェーズ7-5](phase7_5_nn_node.md) |

### 14-4. 主なトピック

| トピック | 型 | 流れるもの |
|---|---|---|
| `/chatter` | `std_msgs/msg/String` | `talker` の文字列（フェーズ3-1） |
| `/turtle1/cmd_vel` | `geometry_msgs/msg/Twist` | turtlesimの亀への速度の指令（フェーズ1・3-2a） |
| `/target_velocity` | `std_msgs/msg/Float64` | 目標の速度（フェーズ5-2以降） |
| `/plant/pedal` | `std_msgs/msg/Float64` | プラントへのペダルの量（フェーズ5-1以降） |
| `/plant/velocity` | `std_msgs/msg/Float64` | プラントの速度（フェーズ5-1以降） |
| `/model/vehicle_green/odometry` | `nav_msgs/msg/Odometry` | Gazeboの車の位置と速度（フェーズ5-0以降） |
| `/model/vehicle_green/cmd_vel` | `geometry_msgs/msg/Twist` | Gazeboの車への速度の指令（フェーズ5-0以降） |
| `/clock` | `rosgraph_msgs/msg/Clock` | シミュレーション時刻（フェーズ5-4以降。[Tips集](tips.md)の1節） |

## 15. 参考資料

- [ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/)（コマンドの使い方は、Tutorialsの「Beginner: CLI tools」にまとまっている）
- 各コマンドの `--help`（例: `ros2 bag play --help`・`gz sim -h`）。オプションの一覧は、手元の版のものが確実である

> 出典: この資料は、この教材の手順書に出てくるコマンドを集めて並べ直したもので、コマンドとオプションの意味は手元の `--help` で確かめた。コマンドの例は、ROS 2の公式ドキュメント（[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)）のチュートリアルの例と同等のものを含む。11節のGazeboのコマンド（`gz`）は、フェーズ5-0とGazeboの物理の間章（Gazeboの公式の文書と、手元のGazeboで確かめたもの）から集めた。文章は自分の言葉で書いたもので、公式ドキュメントの転載ではない。
