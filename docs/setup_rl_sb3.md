# 環境構築 手順書: 強化学習のライブラリ（Stable-Baselines3）

フェーズ7（強化学習）で使うライブラリを、ROS2とは別の仮想環境（venv）に導入する。使うのは、強化学習のアルゴリズムを集めたライブラリ **Stable-Baselines3**（以下、SB3）と、その追加の部品集 **sb3-contrib**、学習させる環境の共通の形を決めるライブラリ **Gymnasium**、ニューラルネットワークの計算に使う **PyTorch**（CPU版）である。

- 想定環境: [環境構築（WSL2 + Ubuntu 24.04 + ROS2 Jazzy）](setup_wsl2_ros2.md)を済ませたUbuntu 24.04（Python 3.12）
- 所要目安: 30分程度（ダウンロードの時間を含む。回線の速さで前後する）
- 必要なディスクの空き: 2GB程度（導入後の仮想環境の大きさは、4節の期待する結果）

> **この手順書の位置づけ（暫定）**: フェーズ7は作成中で、構成を大きく改める可能性がある。導入するライブラリと版も、フェーズ7の改定に合わせて見直すことがある。

> **実行環境が無くても読めるように**: 確認のコマンドの直後には「期待する結果」として、表示される内容の例とその読み方を載せている。**期待する結果は、筆者の環境（2026-10-02に導入）で実際に表示されたもの**である。ただし、画面の記録が残っていなかった分（3節のダウンロードの行のファイル名と、2節のpipの更新・4節の導入の `Successfully installed` の行）は、仮想環境に導入されたものから書き起こしている（ダウンロードの行は、導入されたPyTorchの記録にある版と対応する環境の名前から。`Successfully installed` の行は、導入された版の一覧からの抜粋）。版の数字・所要時間・ディスクの大きさは、導入した時期と環境によって異なる。7節（2026-10-06に追加）の表示の出どころは、次のとおり。`import rclpy` の失敗は、筆者の仮想環境で実際に実行した表示。`pip install pyyaml==6.0.3` の `Successfully installed` の行は、PyTorch等を入れていない使い捨ての仮想環境に入れたときの表示で、その仮想環境の `site-packages` を `PYTHONPATH` に足して、筆者の仮想環境のPythonで `rclpy: OK` と表示されることを確かめた（PyTorchを入れていない仮想環境では、`launch-ros ... requires setuptools` という `ERROR` の行も出たが、3節で入れるPyTorchが `setuptools` を一緒に入れるので、手順書のとおりの仮想環境では出ない見込み）。

## 目的・ゴール

ROS2のシステムのPython（aptで入ったもの）を汚さずに、強化学習のライブラリを使える仮想環境を作る。

**完了の判定基準**（すべて満たしたら完了）

1. 仮想環境 `~/rl_venv` を有効にすると、プロンプトの先頭に `(rl_venv)` が付く（2節）
2. PyTorchがCPU版で入り、`torch.cuda.is_available()` が `False` を返す（3節）
3. SB3・sb3-contrib・Gymnasiumの版が表示され、`Pendulum-v1` の環境を作れる（4節）
4. （フェーズ7-3の前に）仮想環境から、ROS2の `rclpy` を読み込める（7節）

## 注意

- 導入の作業は、**自分の手で**行う。コマンドの意味が分からないまま貼り付けて実行しない。
- **`sudo pip install` や、仮想環境を有効にしないままの `pip install` はしない。** ROS2のPythonのパッケージは、aptで入れたシステムのPythonに依存している。システム側に別の版のライブラリを入れると、ROS2のノードが動かなくなることがある。Ubuntu 24.04では、システムのPythonへの `pip install` は既定で拒否される（`externally-managed-environment` のエラー）。
- 仮想環境は、ワークスペース（`~/ros2_ws`）の**外**に作る。ワークスペースの中に置くと、`colcon build` が仮想環境の中のパッケージまで探しに行き、ビルドが遅くなったり失敗したりする。
- 導入するものはすべて、公式の配布元（Ubuntuのapt、Python Package Index（PyPI）、PyTorchの公式の配布元）から取得する。

### なぜ仮想環境を使うのか

仮想環境（venv）は、Pythonのパッケージを入れる場所を、フォルダ1つに閉じ込める仕組みである。有効にしている間だけ、`python` と `pip` がそのフォルダの中を向く。強化学習のライブラリは、numpyなどの共通のライブラリに新しい版を求めることがあり、ROS2が使う版とぶつかりうる。仮想環境に分けておけば、ぶつからないうえ、要らなくなったらフォルダごと消して元に戻せる（6節）。

## 手順

### 1. 仮想環境を作る部品を入れる

Ubuntu 24.04の既定の状態では、Pythonの仮想環境を作る部品（`venv` の `ensurepip`）が入っていないことがある。aptで入れる。

```bash
sudo apt update

sudo apt install python3-venv
```

**期待する結果**（`sudo apt install` の分。抜粋）:

```text
The following NEW packages will be installed:
  python3-pip-whl python3-setuptools-whl python3-venv python3.12-venv
...
Setting up python3.12-venv (3.12.3-...) ...
Setting up python3-venv (3.12.3-...) ...
```

`python3-venv` と、それが依存する `python3.12-venv` が入れば成功である。すでに入っている場合は `python3-venv is already the newest version` と表示される。版の数字は時期によって違う。

### 2. 仮想環境を作って有効にする

ホームディレクトリの下に、仮想環境 `rl_venv` を作る。

```bash
python3 -m venv ~/rl_venv

source ~/rl_venv/bin/activate

python --version

which python pip
```

**期待する結果**:

```text
$ python --version
Python 3.12.3
$ which python pip
/home/<ユーザー名>/rl_venv/bin/python
/home/<ユーザー名>/rl_venv/bin/pip
```

プロンプトの先頭に `(rl_venv)` が付き、`python` と `pip` が `~/rl_venv/bin` の下を指していれば、仮想環境が有効になっている。Pythonの版の細かい数字（`3.12.3`）は、Ubuntuの更新の状況で違うことがある。

仮想環境は、**ターミナルごとに**有効にする。新しいターミナルを開いたら、`source ~/rl_venv/bin/activate` を実行してから使う。やめるときは `deactivate` を実行する（プロンプトの `(rl_venv)` が消える）。

続けて、仮想環境の中の `pip` を新しくしておく。

```bash
python -m pip install --upgrade pip
```

**期待する結果**（抜粋。版は時期によって違う）:

```text
Successfully installed pip-26.2.1
```

### 3. PyTorch（CPU版）を入れる

**SB3より先に、PyTorchのCPU版を入れる。** 先にSB3を入れると、依存関係として、PyPIからGPU（CUDA）向けのPyTorchが入る。こちらはGPUの計算用のライブラリを含むため数GBあり、GPUが無いPCでは使わない部分がほとんどになる。

PyTorchの公式の案内（[Get Started](https://pytorch.org/get-started/locally/)）で「Linux・Pip・Python・CPU」を選んだときと同じく、配布元にCPU版を指定して入れる（この教材ではtorchvisionを使わないので、torchだけを入れる）。

```bash
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

**期待する結果**（抜粋。版は時期によって違う）:

```text
Looking in indexes: https://download.pytorch.org/whl/cpu
Collecting torch
  Downloading https://download.pytorch.org/whl/cpu/torch-2.14.1%2Bcpu-cp312-cp312-manylinux_2_28_x86_64.whl (... MB)
...
Successfully installed MarkupSafe-3.0.3 filelock-3.32.3 fsspec-2026.7.0 jinja2-3.1.6 mpmath-1.3.0 networkx-3.6.1 setuptools-78.1.0 sympy-1.14.0 torch-2.14.1+cpu typing-extensions-4.16.0
```

ファイル名に `+cpu` と `cp312`（Python 3.12向け）が入っていれば、CPU版が選ばれている。

> **`launch-ros ... requires pyyaml, which is not installed.` と出る場合**: 末尾の `Successfully installed` の直前に、次のような `ERROR` が出ることがある（4節の導入でも出ることがある）。
>
> ```text
> ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
> launch-ros 0.26.12 requires pyyaml, which is not installed.
> ```
>
> これは導入の失敗ではなく、無視してよい。`~/.bashrc` でROS2を読み込んでいると、環境変数 `PYTHONPATH` にROS2のPythonのパッケージの場所（`/opt/ros/jazzy/lib/python3.12/site-packages`）が入り、仮想環境のpipにもROS2の `launch_ros` が見える。`launch_ros` が必要とするpyyamlは、aptでシステムのPythonの側（`python3-yaml`）に入っているが、仮想環境からはシステムの側のパッケージが見えない。そのためpipは「pyyamlが無い」と報告する。仮想環境の中ではROS2のlaunchを使わないので、学習には影響しない。ROS2のコマンドはシステムのPythonで動くので、こちらも影響を受けない。フェーズ7-0〜7-2では、このまま進めてよい。フェーズ7-3からは、仮想環境の中でROS2の `rclpy` を使うので、7節でpyyamlを入れる（入れた後は、この表示は出なくなる）。版の数字（`0.26.12`）は、ROS2の更新の状況で違うことがある。

入ったことを確かめる。

```bash
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
```

**期待する結果**（版は時期によって違う）:

```text
2.14.1+cpu False
```

版の末尾に `+cpu` が付き、`torch.cuda.is_available()` が `False`（GPUを使わない）になっていれば成功である。

> **GPUのあるPCで使う場合**: NVIDIAのGPUを積んだPCでは、PyTorchの公式の案内で、そのPCのCUDAの版に合う配布元を選んで入れる。フェーズ7の7-0〜7-5の規模では、CPU版で足りる。GPUが効いてくるのは、二足歩行のような大きな学習（7-6の予定）からである。

### 4. SB3・sb3-contrib・Gymnasiumを入れる

SB3の公式の導入の案内（[Installation](https://stable-baselines3.readthedocs.io/en/master/guide/install.html)）は、次の2通りを示している（抜粋）。

```bash
pip install 'stable-baselines3[extra]'
```

```bash
pip install stable-baselines3
```

前者は、学習の記録の表示（TensorBoard）・画像の処理（OpenCV）・Atariのゲームの環境（`ale-py`）などの追加の部品もまとめて入れる。この教材ではAtariのゲームを使わないので、後者の最小の形を基本にし、要る部品だけを足す。

- **`gymnasium[classic-control]`**: 振り子（Pendulum）などの古典的な制御の環境を、画面に表示するための部品（`pygame-ce`。ゲームを作るライブラリpygameの、コミュニティによる派生版）を含めて入れる。Gymnasiumの公式の入門（[Basic Usage](https://gymnasium.farama.org/introduction/basic_usage/)）の最初の例も、この形で入れるよう案内している。
- **`sb3-contrib`**: 記憶を持つポリシー（`RecurrentPPO`）など、SB3の追加のアルゴリズム。フェーズ7の後半で使う予定。

この手順書を作った時点（2026-09-29）の最新版に固定して入れる（2026-10-02に、この版で導入して確かめた。版を固定すると、手順書と同じ結果を再現しやすい。新しい版を使う場合は、`==` 以降を外す）。

```bash
pip install stable-baselines3==2.9.0 sb3-contrib==2.9.0 "gymnasium[classic-control]==1.3.0"
```

**期待する結果**（抜粋。依存するライブラリの版は時期によって違う）:

```text
Successfully installed cloudpickle-3.1.2 ... gymnasium-1.3.0 numpy-2.5.3 pygame-ce-2.5.8 sb3-contrib-2.9.0 stable-baselines3-2.9.0
```

末尾の `Successfully installed` の行に、`stable-baselines3-2.9.0`・`sb3-contrib-2.9.0`・`gymnasium-1.3.0` があれば成功である。**`torch` がもう一度ダウンロードされていないこと**も確かめる（3節で入れたCPU版がそのまま使われる）。

版を確かめ、振り子の環境を1つ作ってみる。

```bash
python -c "import stable_baselines3, sb3_contrib, gymnasium; print(stable_baselines3.__version__, sb3_contrib.__version__, gymnasium.__version__)"

python -c "import gymnasium as gym; env = gym.make('Pendulum-v1'); print(env.observation_space); print(env.action_space)"

du -sh ~/rl_venv
```

**期待する結果**（版と `du` の大きさは時期によって違う）:

```text
$ python -c "import stable_baselines3, ..."
2.9.0 2.9.0 1.3.0
$ python -c "import gymnasium as gym; ..."
Box([-1. -1. -8.], [1. 1. 8.], (3,), float32)
Box(-2.0, 2.0, (1,), float32)
$ du -sh ~/rl_venv
1010M	/home/<ユーザー名>/rl_venv
```

- 1行目は、SB3・sb3-contrib・Gymnasiumの版である。
- 2行目の `Box([-1. -1. -8.], [1. 1. 8.], (3,), float32)` は、振り子の環境が返す観測（3つの数）の範囲、3行目の `Box(-2.0, 2.0, (1,), float32)` は、エージェントが与える行動（トルク1つ）の範囲である。中身はフェーズ7-0で読む。
- 仮想環境の大きさは、1GB前後になる（ほとんどがPyTorch）。

### 5. ROS2と一緒に使うときの注意

- **`colcon build` は、仮想環境を無効にしたターミナルで行う。** 仮想環境を有効にしたままビルドすると、ノードの起動用のスクリプトが仮想環境のPythonを指すことがあり、ROS2のノードの動きが環境によって変わる。
- **学習のスクリプトは、仮想環境を有効にしたターミナルで動かす。** `~/.bashrc` でROS2を読み込んでいても、仮想環境の中から強化学習のライブラリを使える（ROS2のPythonのパッケージのパスには、SB3が使うライブラリと名前がぶつかるものは無い）。ただし、環境変数 `PYTHONPATH` に入っている場所（ROS2の `/opt/ros/jazzy/...` や、`source` したワークスペースの `install/`）は、仮想環境のパッケージより先に探される。ワークスペースで作ったパッケージが、SB3などと同じ名前にならないようにする。
- 学習した制御器をROS2のノードで動かすとき（フェーズ7-5）は、ノード側で仮想環境を要らなくする方法（学習した重みをnumpyだけで計算する）を扱う予定である。

### 6. 作り直すとき・消すとき

仮想環境はフォルダ1つなので、消せば導入前の状態に戻る（1節の `python3-venv` は残る）。版を入れ替えたいときや、導入に失敗したときは、消して2節から作り直す。

```bash
deactivate

rm -rf ~/rl_venv
```

**期待する結果**: 何も表示されない。`deactivate` はプロンプトの `(rl_venv)` を消す。`rm -rf` は確認なしに消すので、実行する前に消す対象（`~/rl_venv`）を見直す。ROS2やワークスペースには影響しない。

### 7. ROS2のPythonのライブラリを使う準備（フェーズ7-3の前に）

フェーズ7-3からは、学習の環境の中で、ROS2のPythonのライブラリ `rclpy` を使い、Gazeboとトピックやサービスでやりとりする。ROS2を読み込んだターミナル（`source /opt/ros/jazzy/setup.bash` 済み）では、`rclpy` そのものは、環境変数 `PYTHONPATH` に入ったROS2の場所（`/opt/ros/jazzy/lib/python3.12/site-packages`）から見つかる。ところが、`rclpy` が使うライブラリのうち、YAMLの読み書きをする **PyYAML**（`import yaml`）は、aptでシステムのPythonの側に入っていて、仮想環境の中からは見えない（3節の注意書きと同じ理由）。仮想環境の中にあるのは、自分で入れたものだけだからである。

まず、仮想環境を有効にしたターミナルで、`rclpy` を読み込めないことを確かめる。

```bash
python -c "import rclpy"
```

**期待する結果**（抜粋。途中の行は省いた）:

```text
Traceback (most recent call last):
  ...
  File "/opt/ros/jazzy/lib/python3.12/site-packages/rclpy/parameter.py", line 27, in <module>
    import yaml
ModuleNotFoundError: No module named 'yaml'
```

`rclpy` は見つかっているが（`/opt/ros/jazzy/...` のファイルが読まれている）、その中の `import yaml` で止まっている。足りないのは `yaml` だけなので、PyYAMLを仮想環境に入れる。4節と同じく、この手順書を作った時点（2026-10-06）の最新版に固定する。

```bash
pip install pyyaml==6.0.3

python -c "import rclpy; from nav_msgs.msg import Odometry; print('rclpy: OK')"
```

**期待する結果**:

```text
$ pip install pyyaml==6.0.3
...
Successfully installed pyyaml-6.0.3
$ python -c "import rclpy; ..."
rclpy: OK
```

`Successfully installed pyyaml-6.0.3` の後、`rclpy: OK` と表示されれば、仮想環境のPythonから、ROS2のライブラリとメッセージの型（ここではオドメトリの [`nav_msgs/msg/Odometry`](https://github.com/ros2/common_interfaces/blob/jazzy/nav_msgs/msg/Odometry.msg)）を使える。

> **補足: 仮想環境とROS2のライブラリの組み合わせ方**
>
> - **一般的な方法**: ROS2の公式の文書（Using Python Packages with ROS 2）は、特別なオプションを付けずに仮想環境を作り、要るパッケージを仮想環境に `pip` で入れる形を示している。注意として挙げているのは、ROS2を作ったのと同じPython（Ubuntu 24.04では、システムの `python3`）で仮想環境を作ることで、2節の作り方はこれを満たしている。
> - **ほかの方法との違い**: 仮想環境を作るときに `--system-site-packages` を付けると、aptで入ったPythonのパッケージもすべて見えるようになり、PyYAMLを入れなくても `rclpy` を読める。その代わり、仮想環境とシステムのパッケージが混ざり、どちらの版が読み込まれたのかが分かりにくくなる（たとえばnumpyは、仮想環境に入る版とaptの版で、大きく違う）。この教材では、「仮想環境の中にあるのは、自分で入れたものだけ」という形を保つため、足りないものを1つずつ入れる。
> - **実務の目安**: 仮想環境で使うパッケージを `pip freeze` で書き出しておくと、別のPCやコンテナで同じ環境を作り直せる。仮想環境の中でROS2のlaunchなど、ほかのライブラリも使うようになったら、足りないと言われたものを同じように足す。

## つまずきやすい点

| 症状 | 確認すること |
|---|---|
| `python3 -m venv` で `ensurepip is not available` と出る | 1節の `python3-venv` を入れたか |
| `pip install` で `externally-managed-environment` と出る | 仮想環境を有効にしたか（プロンプトに `(rl_venv)` があるか）。新しいターミナルでは、有効にし直す |
| 導入に長い時間がかかり、`nvidia-...` という名前のパッケージが次々にダウンロードされる | 3節より先にSB3を入れて、GPU向けのPyTorchが選ばれている。`Ctrl+C` で止め、6節で消して作り直す |
| `pip install` の最後に `launch-ros ... requires pyyaml, which is not installed.` という `ERROR` が出る | `Successfully installed` の行が出ていれば、導入は成功している。ROS2を読み込んだターミナルで出ることがあり、無視してよい（3節の注意書き）。7節でPyYAMLを入れた後は、出なくなる |
| `import rclpy` で `No module named 'yaml'` と出る | 7節のPyYAMLを、仮想環境を有効にしたターミナルで入れたか |
| `import rclpy` で `No module named 'rclpy'` と出る | ROS2を読み込んだターミナルか（`source /opt/ros/jazzy/setup.bash`。`~/.bashrc` で読み込んでいれば不要） |
| `import gymnasium` で `No module named 'gymnasium'` と出る | 仮想環境を有効にしたか。`which python` が `~/rl_venv/bin/python` を指しているか |
| ROS2のノードが、ビルドし直した後に動かなくなった | 仮想環境を有効にしたまま `colcon build` しなかったか（5節）。`deactivate` してから、ワークスペースの `build/`・`install/` を消してビルドし直す |

## 次へ

フェーズ7-0（[`docs/phase7_0_rl_intro.md`](phase7_0_rl_intro.md)）へ進む。強化学習の用語を押さえ、Gymnasiumの環境をでたらめに動かしてから、SB3で振り子を学習させ、手書きの制御器と比べる。

## 公式ドキュメント

- [Stable-Baselines3 Documentation — Installation](https://stable-baselines3.readthedocs.io/en/master/guide/install.html)（公式・英語）
- [Gymnasium Documentation — Basic Usage](https://gymnasium.farama.org/introduction/basic_usage/)（公式・英語）
- [PyTorch — Get Started](https://pytorch.org/get-started/locally/)（公式・英語）
- [venv — Creation of virtual environments](https://docs.python.org/3/library/venv.html)（Pythonの公式の文書・英語）
- [Using Python Packages with ROS 2](https://github.com/ros2/ros2_documentation/blob/jazzy/source/How-To-Guides/Using-Python-Packages.rst)（ROS 2の公式の文書の原稿（jazzy）・英語。7節の補足）
- [PyYAML（PyPI）](https://pypi.org/project/PyYAML/)（7節で入れるパッケージ。MIT License）
- PyPI: [stable-baselines3](https://pypi.org/project/stable-baselines3/)・[sb3-contrib](https://pypi.org/project/sb3-contrib/)・[gymnasium](https://pypi.org/project/gymnasium/)

> 出典: 4節の `pip install 'stable-baselines3[extra]'` と `pip install stable-baselines3` は、Stable-Baselines3の公式の文書（Installation）からの抜粋である（Copyright (c) 2019 Antonin Raffin, MIT License）。4節の `gymnasium[classic-control]` を入れる案内は、Gymnasiumの公式の文書（Basic Usage）に基づく（Copyright (c) 2016 OpenAI, Copyright (c) 2022 Farama Foundation, MIT License）。3節のコマンドは、PyTorchの公式の案内と同等の形である。MIT Licenseの全文は [`LICENSE-MIT-THIRD-PARTY`](../LICENSE-MIT-THIRD-PARTY)。説明の文章は、公式の文書を参考に自分の言葉で書いたもの。
