# フェーズ5-0 手順書: Gazeboを導入し、用意されたロボットをROS2から動かす

`docs/learning_plan.md` フェーズ5の冒頭（idea_origin.md ステップ5）に対応する。3D物理シミュレータのGazebo（Harmonic）を導入し、公式のデモ集 `ros_gz_sim_demos` に入っている2輪の車両を、ROS2のトピックで走らせる。フェーズ5-1以降で自作する「車両」の速度制御を、物理シミュレータの車両で先に体験しておく位置づけ。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy
- 前提: フェーズ1（`ros2 topic pub` でTwistを送れる）とフェーズ3-1（Publisherを書ける）。launchファイルは起動するだけで、書き方（フェーズ4）は知らなくてよい
- 所要目安: 1〜2コマ
- 言語: 導入と観察は言語非依存。自作ノードはPython

> **作成中**: この手順書は1節（導入）まで書いてある。2節以降は、導入後の環境でlaunchファイルやトピック名を確かめてから書く（下の「0. 学習目標と完了条件」と構成案は予定）。

> **実行環境が無くても読めるように**: 実行する手順の直後には「期待する結果」として、表示される内容の例とその読み方を載せている。出どころ（実際に確かめた表示か、仕様から想定した表示か）は各所に書く。

## 0. 学習目標と完了条件（予定）

1. Gazebo（Harmonic）とROS2の連携パッケージ `ros_gz` を導入し、ROS2のバージョンとGazeboのバージョンの対応を説明できる。
2. `ros_gz_sim_demos` の2輪車のデモを起動し、`ros2 topic pub` で車両を走らせられる。
3. Gazebo側のトピックとROS2側のトピックを、`ros_gz_bridge` が橋渡ししていることを説明できる。
4. フェーズ3-1のPublisherを応用した自作ノードで車両を走らせ、オドメトリ（車両が推定した位置・速度）を購読して観察できる。

構成案（2節以降）:

- 2. Gazebo単体でワールドを開く（ROS2を使わずに、シミュレータの画面と操作に慣れる）
- 3. デモを起動し、ROS2のコマンドで車両を走らせる
- 4. 仕組み: `ros_gz_bridge` によるトピックの橋渡し（図）
- 5. 自作ノードで走らせ、オドメトリを観察する（Python）
- 6. フェーズ5-1以降とのつながり（自作の疑似プラントとGazeboの車両の比べ方）

## 1. Gazeboの導入

### 1-1. どのGazeboを入れるか

Gazeboには、旧来の「Gazebo Classic」と、その後継の新しい「Gazebo」（以前はIgnitionと呼ばれていた系統）がある。Gazebo Classicはサポートが終わる系統のため、この教材では新しいGazeboを使う。

新しいGazeboには「Harmonic」「Ionic」などの名前の付いた版がある。ROS2の版ごとに推奨の組み合わせが決まっていて、**ROS2 JazzyにはGazebo Harmonic**が推奨されている。Jazzyからは、GazeboがROS2のaptリポジトリから「vendorパッケージ」（ROS2の配布物としてGazeboのライブラリを同梱したもの）として配布されるので、Gazebo本家のaptリポジトリを別に追加しなくてよい。

入れるものは次の2つ。

| パッケージ | 中身 |
|---|---|
| `ros-jazzy-ros-gz` | Gazebo Harmonic本体（vendorパッケージ経由）と、ROS2と連携するための `ros_gz_sim`（起動用）・`ros_gz_bridge`（トピックの橋渡し）など |
| `ros-jazzy-ros-gz-sim-demos` | 公式のデモ集。この手順書で動かす2輪車のデモが入っている |

### 1-2. インストール

```bash
sudo apt update
sudo apt install ros-jazzy-ros-gz ros-jazzy-ros-gz-sim-demos
```

依存するパッケージ（RViz2・rqt_plot・xacro など）もまとめて入るので、ダウンロードとインストールにしばらくかかる。

> **PCの負荷について**: Gazeboは3D描画と物理計算を同時に行うため、ここまでのturtlesimやrqtよりはるかに重い。WSL2に割り当てたメモリが少ない環境では、動かす間はブラウザなど他のアプリを閉じておくとよい。

**期待する結果**（仕様から想定した表示。パッケージの数・容量は環境によって異なる）:

```text
...
The following NEW packages will be installed:
  ros-jazzy-gz-sim-vendor ros-jazzy-ros-gz ros-jazzy-ros-gz-bridge ros-jazzy-ros-gz-sim ros-jazzy-ros-gz-sim-demos ...
...
Do you want to continue? [Y/n]
```

`Y` で続け、最後にエラー（`E:` で始まる行）が出ずにプロンプトへ戻れば成功。

### 1-3. 入ったことを確かめる

インストール直後のターミナルでは、新しく入ったパッケージの設定が読み込まれていない。**新しいターミナルを開いてから**次を実行する（`~/.bashrc` で `/opt/ros/jazzy/setup.bash` が読み込まれる）。

```bash
ros2 pkg list | grep ros_gz
gz sim --version
```

**期待する結果**（仕様から想定した表示。版の細かな数字は異なる）:

```text
ros_gz
ros_gz_bridge
ros_gz_image
ros_gz_interfaces
ros_gz_sim
ros_gz_sim_demos
Gazebo Sim, version 8.x.x
```

- 1つめのコマンドで `ros_gz_bridge`・`ros_gz_sim`・`ros_gz_sim_demos` が並べば、ROS2側の連携パッケージが入っている。
- 2つめで `version 8.` と出れば、Gazebo Harmonic（Gazebo Simの8系）が使える。`gz: command not found` になる場合は、新しいターミナルで試しているかを確かめる。

## 公式ドキュメント・参考資料

確認状況（2026-09-24）: Gazebo公式のページはこの手順書の作成時に本文を確認した。

### 公式

- [Installing Gazebo with ROS — Gazebo Harmonic](https://gazebosim.org/docs/harmonic/ros_installation/)（ROS2の版とGazeboの版の対応、Jazzyでの導入方法）
- [gazebosim/ros_gz — ros_gz_sim_demos（GitHub、jazzyブランチ）](https://github.com/gazebosim/ros_gz/tree/jazzy/ros_gz_sim_demos)（デモの一覧と起動方法）

### 日本語

- [ROS 2初心者向けレベル2 – TF | URDF | RViz | Gazebo：Gazebo｜Hafnium](https://note.com/hafnium/n/n29e7e879da18)
- [gazeboとROS2を連携してロボットを操作する #Gazebo - Qiita](https://qiita.com/N622/items/9d89f77d85d9da0af29e)

> 記事は個人による非公式の解説で、版によって異なる場合がある。公式ドキュメントと食い違う場合は公式を優先する。

> 出典: 導入のコマンドは、上記のGazebo公式ドキュメントの手順と同等のもの。文章は独自に書いたもので、逐語の転載ではない。
