# フェーズ2 手順書: ワークスペースとパッケージ作成・ビルド

`docs/learning_plan.md` フェーズ2（idea_origin.md ステップ1の1-0）に対応する。`ament_python` と `ament_cmake` の空パッケージを1つずつ作り、ビルドと実行の流れの違いを確認する。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy（`colcon`、`gcc`、`cmake` は導入済み）
- 所要目安: 1コマ
- 言語: **Python（`ament_python`）とC++（`ament_cmake`）の両方**
- 前提: フェーズ1（`docs/phase1_cli_turtlesim.md`）で `ros2 run <パッケージ> <実行ファイル>` の書式に触れていること

> **この手順書の位置づけ**: コマンドはROS2 Jazzyの実機の雛形（`ros2 pkg create` のテンプレート）を読んで確認しているが、公式チュートリアルの本文とは照合できていない（docs.ros.orgがボット対策で取得不可）。出力が違えば実機を優先し、差分を貼ってほしい。文章・構成は自分の言葉で書いた。公式ドキュメント（CC BY 4.0）の出典は末尾に記載する。

## 0. 学習目標と完了条件

1. ワークスペース（`ws/`）を作り、`ament_python` と `ament_cmake` のパッケージを1つずつ作れる。
2. `colcon build` → `source install/setup.bash` → `ros2 run` の流れを説明できる。
3. 2種類のパッケージで「必要なファイル」と「実行ファイルが登録される仕組み」の違いを説明できる。
4. `--symlink-install` の有無で、Pythonの修正が再ビルドなしで反映されるかどうかが変わることを体験する。

## 1. 全体像

### 1-1. ワークスペースの構成

![ワークスペースの構成。src/だけが自分で書くもの（Git管理）で、build/・install/・log/は生成物](img/phase2_ws_tree.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart TB
    WS["ws/ （ワークスペース）"]
    WS --> SRC["src/ ← 自分で書くもの（Git管理）"]
    WS --> BLD["build/ ← 中間生成物（Git管理外）"]
    WS --> INS["install/ ← 実行に使う成果物（Git管理外）"]
    WS --> LOG["log/ ← ビルドログ（Git管理外）"]
    SRC --> PY["learn_py/ （ament_python）"]
    SRC --> CPP["learn_cpp/ （ament_cmake）"]
```

</details>

`build/`・`install/`・`log/` は `.gitignore` に登録済み（リポジトリ直下の `.gitignore`）。`src/` だけがGit管理の対象になる。

### 1-2. ビルドと実行の流れ

![ソースからビルド、環境変数への登録、ノード起動までの流れ](img/phase2_build_flow.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart LR
    A["src/ のソース"] -- "colcon build" --> B["build/ （中間）"]
    B --> C["install/ （成果物）"]
    C -- "source install/setup.bash" --> D["環境変数に登録<br/>（AMENT_PREFIX_PATH等）"]
    D -- "ros2 run パッケージ 実行ファイル" --> E["ノードが起動"]
```

</details>

`source install/setup.bash` を忘れると、`ros2 run` が「パッケージが見つからない」と言う。ビルドし直した後だけでなく、**新しいターミナルを開くたびに必要**（`~/.bashrc` には `/opt/ros/jazzy` のsourceだけが入っているため）。

### 1-3. 2種類のパッケージの違い

| 観点 | `ament_python` | `ament_cmake`（C++） |
|---|---|---|
| ビルド設定 | `setup.py` と `setup.cfg` | `CMakeLists.txt` |
| 依存の宣言 | `package.xml` の `<depend>` （加えて `setup.py` の `install_requires`） | `package.xml` の `<depend>` と `CMakeLists.txt` の `find_package` |
| ソースの置き場所 | `パッケージ名/`（Pythonモジュール） | `src/`、`include/` |
| 実行ファイルの登録 | `setup.py` の `entry_points` の `console_scripts` | `CMakeLists.txt` の `add_executable` と `install(TARGETS ...)` |
| コンパイル | なし（ソースをそのまま配置） | あり（`g++` 経由で機械語にする） |
| 修正後の再ビルド | `--symlink-install` なら**不要** | **必須** |

### 1-4. 用語の整理: 実行ファイル・ノード・launchファイル

この手順書で言う「実行ファイル」は、`ros2 run <パッケージ> <実行ファイル>` の第2引数に書ける名前のこと。`setup.py` の `entry_points`（Python）や `CMakeLists.txt` の `add_executable`（C++）で**ビルド時に登録**され、`ros2 pkg executables` で一覧できる、パッケージング上の単位。実体はPythonの短いラッパースクリプトか、コンパイル済みのELFバイナリで、1個 = `ros2 run` すると1個のプロセスが立ち上がる。

これは「ノード」「launchファイル」とは別の軸の概念で、次のように整理できる。

| 概念 | 何の単位か | どこで見える・使うか |
|---|---|---|
| 実行ファイル | ビルド・パッケージングの単位（プロセスとして起動できるもの） | `ros2 pkg executables`、`ros2 run <pkg> <実行ファイル>` |
| ノード | 実行時（ランタイム）の単位。`rclpy.Node` / `rclcpp::Node` のインスタンス | `ros2 node list`、トピック/サービス/パラメータの持ち主 |
| launchファイル | 複数の実行ファイル（＝複数のノード）をまとめて起動する設定の単位 | `ros2 launch <pkg> <ファイル>`（フェーズ4で扱う） |

この手順書のように「1つの実行ファイルの `main()` が1つのノードを作ってspinする」のが最小構成では最も単純で典型的な形だが、**実行ファイルとノードは厳密には1対1ではない**。1つの実行ファイル（1プロセス）が複数のノードを作って同時にspinすることもできるし、逆にノード単体を `ros2 run` で直接起動する方法はない（必ず「それを起動する実行ファイル」を経由する）。launchファイルはさらに1段上の層で、`ros2 pkg executables` には出てこず、`ros2 run` の対象にもならない（`ros2 launch` 専用のファイル）。フェーズ4（`docs/phase4_launch.md`）で、launchファイルが複数の実行ファイル＝ノードをまとめて起動する様子を実際に書いて確認する。

## 2. 手順

以降、コマンドは `~/work/ros2MinimalPhysicalAi`（本リポジトリのclone先）を起点に書く。場所が違う場合は読み替える。

### 2-1. ワークスペースを作る

```bash
cd ~/work/ros2MinimalPhysicalAi
mkdir -p ws/src
cd ws/src
```

`ws/src` に置いたものがパッケージとして扱われる。

### 2-2. Pythonパッケージを作る

```bash
ros2 pkg create --build-type ament_python \
  --license Apache-2.0 \
  --maintainer-name learner --maintainer-email noreply@example.com \
  --dependencies rclpy std_msgs \
  --node-name hello \
  learn_py
```

- `--dependencies`: `package.xml` に `<depend>` として書かれる（後の手順で `std_msgs` を使うため今のうちに入れる）。
- `--node-name hello`: 動作確認用の最小ノード `hello` の雛形が作られる。
- **メンテナ名・メールは必ず明示する**。省略すると、コマンドがGitの設定などから自動で補う場合があり、実メールアドレスがファイルに入る恐れがある。本リポジトリはPublic化を前提にしているため、`learner` / `noreply@example.com` のようなダミーを使う。

作られたものを確認する:

```bash
find learn_py -type f | sort
cat learn_py/package.xml
cat learn_py/setup.py
```

`package.xml` に `<depend>rclpy</depend>` と `<depend>std_msgs</depend>` の2行があれば、`--dependencies` が効いている。`<test_depend>` の行しか無い場合は指定が抜けているので、この2行を `<license>` の行の後ろに手で足す（フェーズ3-1の3-2でも触れる）。

### 2-3. C++パッケージを作る

```bash
ros2 pkg create --build-type ament_cmake \
  --license Apache-2.0 \
  --maintainer-name learner --maintainer-email noreply@example.com \
  --dependencies rclcpp std_msgs \
  --node-name hello \
  learn_cpp

find learn_cpp -type f | sort
cat learn_cpp/package.xml
cat learn_cpp/CMakeLists.txt
```

> 補足: `--node-name` は生成する実行ファイルの**名前**を指定するだけのオプション（`ros2 pkg create --help` でも `name of the empty executable` としか説明されておらず、選べる「種類」の列挙はない）。ノードの言語・雛形の中身を決めているのは `--build-type` の方（`ament_python` → `rclpy`のPython最小ノード、`ament_cmake`/`cmake` → `rclcpp`のC++最小ノード、`ament_cargo` → Rust）。2-2と2-3で同じ `--node-name hello` を指定しているのは、両方とも「動作確認用の最小ノード」という同じ役割を、`--build-type` 違いのテンプレートで作っているため。

> 課題1: 2つのパッケージの `package.xml` を見比べる。`<buildtool_depend>` の違い（`ament_python` / `ament_cmake`）と、`<export><build_type>` の違いを確認する。

### 2-4. ビルドする

```bash
cd ~/work/ros2MinimalPhysicalAi/ws
colcon build --symlink-install
```

- **必ずワークスペースの直下（`ws/`）で実行する**。`src/` の中で実行すると `build/` などが意図しない場所にできる。
- 初回のビルド時間は環境による（この手順書の検証環境では約10秒だった）。応答が遅い・止まる場合のみ、メモリ不足を疑って `colcon build --symlink-install --parallel-workers 2` のように並列数を絞る（検証環境ではこの絞り込みは不要だった。WSLのメモリ実測は `setup_wsl2_ros2.md` 参照）。
- 成功すると `Summary: 2 packages finished` のように表示される。
- Pythonパッケージのビルド中に `SetuptoolsDeprecationWarning`（非推奨の警告）が出ることがある。ビルドが成功していれば、この段階では無視してよい。

確認:

```bash
ls
# build  install  log  src
```

### 2-5. 環境に登録して実行する

```bash
source install/setup.bash
ros2 run learn_py hello
ros2 run learn_cpp hello
```

期待される表示（雛形の内容。文言が違っていても、実行できていればよい）:

```
Hi from learn_py.
hello world learn_cpp package
```

> 2026-09-22時点でユーザーが実機で確認済み: `learn_py`・`learn_cpp`とも期待どおりの表示だった。

`source` の効果を確認する:

```bash
ros2 pkg list | grep learn
ros2 pkg prefix learn_py
ros2 pkg executables learn_py
ros2 pkg executables learn_cpp
```

> 課題2: 新しいターミナルを開き、`source` しないまま `ros2 run learn_py hello` を試す。エラーになることを確認してから、`source install/setup.bash` をして再実行する。

#### `ros2 pkg` サブコマンドの補足

ここまでで使った4つのサブコマンド（`ros2 pkg <sub> --help` で確認済み、Jazzy時点）の役割とオプション:

| サブコマンド | 役割 | 主なオプション |
|---|---|---|
| `create <package_name>` | 新規パッケージの雛形を作る（2-2/2-3で使用） | `--build-type {cmake,ament_cmake,ament_cargo,ament_python}`（言語・ビルド系）、`--node-name`（雛形ノードの実行ファイル名。上記の補足参照）、`--library-name`（雛形ライブラリ名、C++のみ）、`--dependencies`（`package.xml`の`<depend>`）、`--license`、`--maintainer-name`/`--maintainer-email`、`--destination-directory`（作成先ディレクトリ）、`--package-format {2,3}`（`package.xml`のスキーマ版） |
| `list` | `source`済みの環境から見えているパッケージ名を一覧表示する | オプションなし（`-h`のみ）。今回のように `grep` で絞り込む使い方が一般的 |
| `prefix <package_name>` | そのパッケージのインストール先prefix（`install/<pkg>` 等）を表示する | `--share`: launchファイルやconfig等の共有リソースが入る `share/<pkg>` を表示する |
| `executables [package_name]` | そのパッケージが提供する実行ファイル名（`ros2 run` の第2引数に使える名前）を一覧表示する | `package_name`省略で全パッケージ分。`--full-path`: インストール先の絶対パスも表示する |

`list`・`prefix`・`executables`はいずれも読み取り専用（副作用なし）で、`ros2 run` する前の確認・デバッグに使う。

### 2-6. `install/` の中身を見る

実行ファイルがどこにできているか見て、2種類の違いを実感する。

```bash
ls -l install/learn_py/lib/learn_py/
ls -l install/learn_cpp/lib/learn_cpp/
cat install/learn_py/lib/learn_py/hello | head -20
file install/learn_cpp/lib/learn_cpp/hello
```

- Python版の `hello` は、Pythonの `main` を呼び出すだけの**短いスクリプト**（`setup.py` の `entry_points` から生成される）。
- C++版の `hello` は**コンパイル済みのバイナリ**（`file` コマンドが `ELF ... executable` と表示する）。

> 課題3: `--symlink-install` を付けた場合、`install/learn_py/lib/python3.12/site-packages/learn_py` がシンボリックリンクになっていることを `ls -l` で確認する。

### 2-7. 修正の反映を体験する（`--symlink-install` の効果）

**Python（再ビルド不要）**: `learn_py/learn_py/hello.py` の `print` の文言を書き換え、**ビルドせずに**実行する。

```bash
# 書き換え後
ros2 run learn_py hello     # 文言が変わっている
```

**C++（再ビルド必須）**: `learn_cpp/src/hello.cpp` の文言を書き換え、ビルドせずに実行する。

```bash
ros2 run learn_cpp hello    # まだ古い文言
cd ~/work/ros2MinimalPhysicalAi/ws
colcon build --symlink-install --packages-select learn_cpp
ros2 run learn_cpp hello    # 新しい文言になる
```

- `--packages-select <名前>`: 指定したパッケージだけをビルドする。C++は時間がかかるので、普段はこれを使うと速い。
- 新しい実行ファイルを追加する場合（`setup.py` の `entry_points` や `CMakeLists.txt` の変更）は、`--symlink-install` でも**再ビルドが必要**。

> 補足: なぜC++の再ビルドにも `--symlink-install` を付けるか
>
> C++のコンパイル済み実行ファイルは `src/` に実体が無い（ソースからコンパイルして初めて生成される）ため、`--symlink-install` を付けても「再ビルド不要」にはならない（`install/learn_cpp/lib/learn_cpp/hello` は `src/` ではなく `build/learn_cpp/hello` へのシンボリックリンクになるだけで、`build/` 側の中身はどのみち再コンパイルが要る）。この1コマンド単体では、付けても付けなくても動作は変わらない。
>
> それでも付ける理由は、**ワークスペース全体のインストール方式を一貫させるため**。`--packages-select` で対象を絞っている間は他パッケージに影響しないが、`--packages-select` を付け忘れて`colcon build`（対象なし＝ワークスペース全体）を実行すると、既にシンボリックリンク化されていた `learn_py` 側が実体コピーに巻き戻り、「Pythonは編集がビルド無しで反映される」状態が静かに壊れる（実機での検証で確認済み）。毎回同じ形のコマンドを使い回すことで、この事故を防いでいる。なお、C++パッケージでも `package.xml` やCMake生成物（`install/learn_cpp/share/learn_cpp/` 配下の一部）はシンボリックリンクになるため、実行ファイル以外では`--symlink-install`に意味がある。

> 課題4: `--symlink-install` を付けずに `colcon build` して、Pythonのソースを書き換えても反映されないことを確認する。確認後は `rm -rf build install log` で消して、`--symlink-install` 付きでビルドし直す（`ws/` の中だけを消すこと）。

### 2-8. Git管理の確認

```bash
cd ~/work/ros2MinimalPhysicalAi
git status --short
```

- **`ws/src` を一度もコミットしていない時点**（本フェーズで初めて実行する場合）: `?? ws/` の1行だけが出る。これは `ws/build`・`ws/install`・`ws/log` が `.gitignore` で除外されているからではなく、gitの既定動作（追跡ファイルが1つも無いディレクトリは中身を展開せず1行にまとめる）による。`ws/src/` だけが個別に出るわけではない。中身を個別に確認したい場合は `git status --short -uall` を使うと、`ws/src/...` 配下のファイルだけが列挙され、`ws/build`・`ws/install`・`ws/log` は（`.gitignore` どおり）出てこないことを確認できる（フェーズ3-3〜4の途中では、一時的に `ws/config/...` も出る。フェーズ4で移すのでコミットしない）。
- **`ws/src` を一度コミットした後**: 変更が無ければ何も出ない（クリーン）。新しいファイルを `ws/src` 配下に追加した場合は、そのファイルのパス（例: `?? ws/src/learn_py/learn_py/new_node.py`）だけが個別に出る。

あわせて、Public化前提のため、次を確認する:

```bash
grep -n 'maintainer' ws/src/learn_py/package.xml ws/src/learn_cpp/package.xml
grep -n 'maintainer' ws/src/learn_py/setup.py
```

実メールアドレスが入っていないこと（`noreply@example.com` であること）。コミットするかどうかはClaudeに依頼する（コミットは規約に従いClaudeが行う）。

## 3. Python版とC++版の違いのまとめ

本フェーズで確認したPython（`ament_python`）とC++（`ament_cmake`）の違いを整理する（1-3節の表・2-6節・2-7節の内容の総括）。

| 観点 | Python（`ament_python`） | C++（`ament_cmake`） |
|---|---|---|
| 作成コマンドの差 | `--build-type ament_python` | `--build-type ament_cmake` |
| 実行ファイルの登録場所 | `setup.py` の `entry_points` の `console_scripts` | `CMakeLists.txt` の `add_executable` と `install(TARGETS ...)` |
| `install/` 内の実行ファイルの正体 | Pythonスクリプト（ソースを呼び出す短いラッパー） | コンパイル済みELFバイナリ |
| ソース修正後に必要な操作 | `--symlink-install` なら不要（即反映） | 必須（`g++` での再コンパイルが要る。2-7節） |

コンパイルという工程が挟まる分、C++は「編集→即実行」のPythonに比べて反復（トライ＆エラー）のサイクルが長くなる。一方でコンパイル時に型やシグネチャの誤りを検出できる点はC++の利点（実行時まで気づかないPythonとの違い）。

## 4. つまずきやすい点

| 症状 | 確認すること |
|---|---|
| `Package 'learn_py' not found` | `source install/setup.bash` をしたか。別ターミナルで `ws/` の `install` を読んだか |
| `colcon: command not found` | `sudo apt install python3-colcon-common-extensions`（導入はユーザーが行う）。`colcon --help` で確認 |
| ビルド中に応答が遅い・止まる | メモリ不足の可能性。`--parallel-workers 1` か `2` に絞る |
| `No executable found`（`ros2 run`） | `entry_points` / `install(TARGETS ...)` の記述、ビルド後の `source` |
| Pythonの修正が反映されない | `--symlink-install` を付けてビルドしたか。実行ファイルを**追加**した場合は再ビルドが必要 |
| 警告が大量に出る | 失敗（`Failed`）かどうかだけ確認する。`Summary` の行が判断基準 |

## 5. 次のフェーズへ

フェーズ3（ノードの基本）では、この2つのパッケージ（`learn_py`、`learn_cpp`）に、Publisher/Subscriber・パラメータ・サービス・アクションのノードを**同じ仕様で**追加していく。

## 6. 公式ドキュメント・参考資料

確認状況（2026-09-20）: 下記は `docs/idea_origin.md` に掲載済みのURLで、今回は再確認していない（docs.ros.orgは本文取得がボット対策で拒否される）。コマンドとテンプレートの内容は、WSLに導入済みのROS2 Jazzy（`/opt/ros/jazzy`）の雛形を直接読んで確認した。

### 公式

- [Beginner: Client libraries — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries.html)（ワークスペース作成・パッケージ作成・colconの各チュートリアルはここから辿る）
- [colcon documentation](https://colcon.readthedocs.io/)
- [ament_cmake user documentation — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Ament-CMake-Documentation.html)
- [ament_cmake_python user documentation — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Ament-CMake-Python-Documentation.html)（純Pythonのパッケージは `ament_python` を使う旨の記載あり）

### 日本語

- [ROS 2のワークスペース：colconとパッケージ](https://gbiggs.github.io/rosjp_ros2_intro/workspaces_and_colcon.html)（colcon解説）

> 記事は個人による非公式の解説で、版によって異なる場合がある。公式ドキュメントと食い違う場合は公式を優先する。
