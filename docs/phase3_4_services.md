# フェーズ3-4 手順書: サービス（要求と応答）

[`docs/learning_plan.md`](learning_plan.md) フェーズ3（idea_origin.md ステップ1の1-4）に対応する。「1回の要求に1回の応答を返す」通信を扱う（概要を掴む程度でよい。サンプルはPython版・C++版の両方を載せる）。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy
- 前提: フェーズ3-1〜3-3完了
- 所要目安: 1コマ
- 言語: **Python**（C++版は任意）
- 使う標準インターフェース: [`example_interfaces/srv/AddTwoInts`](https://github.com/ros2/example_interfaces/blob/jazzy/srv/AddTwoInts.srv)、[`std_srvs/srv/Trigger`](https://github.com/ros2/common_interfaces/blob/jazzy/std_srvs/srv/Trigger.srv)

> **このフェーズの位置づけ（概要を掴む程度でよい）**: サービスは、フェーズ5の車両シミュレーションでは使わない（ノード同士はトピックでつなぎ、ゲインはパラメータで変える）。ここでは「サービスは1回の要求に1回の応答を返す通信で、トピックとはこう使い分ける」という概念と、CLIからサービスを呼ぶ方法を掴めば十分である。なお、フェーズ3-3の `ros2 param set` も、裏ではノードが自動で持つパラメータ用のサービスを呼んでいる。
>
> 必須の範囲は、Python版の `add_server` を書いて、`ros2 service call` で呼ぶところまで（4-2節の `add_server.py` → 6-1節の「CLIからも呼べる」）。クライアントの非同期呼び出し（`add_client`）、`counter_node`（Trigger）、C++版（5節）は任意の発展とする。作らないノードがある場合は、`setup.py` の `entry_points` にもその行を書かない。
>
> **進め方**（サンプルまで書いて動かす場合）: 仕様（2節）は「何を作るか」の定義で、APIの使い方までは書いていない。まず「主なAPI」表でサービス関連のAPI（サーバ・クライアントの作成、非同期呼び出し）を把握し、サンプルコードと解説を読んで理解する。読んで分かったら、フィールドや計算内容を変える、待ち時間を変えるなど手を動かして改造してみると定着する。サンプルはこの手順書の作成時にビルド確認済みで、手順は実機で実行して確認済み（2026-09-27）。出力が細部で違う場合は、実機の表示を優先する。

> **実行環境が無くても読めるように**: 実行する手順の直後には「期待する結果」として、表示される内容の例とその読み方を載せている。コードとROS2の仕様から筆者が想定したもので、実機で手順を実行して問題が無いことを確かめてある（表示を1つずつ見比べてはいない）。時刻などの細部は、実行ごとに異なる。

## 0. 学習目標と完了条件

必須（概要を掴む）:

1. 標準の `.srv` 定義（要求と応答の項目）を `ros2 interface show` で読める。
2. サービスのサーバ（`create_service`）をPythonで書き、`ros2 service call` で呼べる。
3. トピックとの使い分け（継続的な流れ vs 要求→応答）を、自分の言葉で説明できる。

任意（発展。余力があれば）:

4. クライアント（`create_client`）の**非同期呼び出し**（`call_async` / `async_send_request`）と、結果の待ち方を説明できる。
5. 1つのノードにトピックとサービスを同居させる（`counter_node`）。
6. C++版を書く。

## 1. 全体像

この手順書のサンプルは、目的の違う2組に分かれている。

| 組 | ノード | 使うサービスの型 | 見どころ | 範囲 |
|---|---|---|---|---|
| ① 足し算 | `add_server`・`add_client` | `AddTwoInts` | サービスの基本（サーバが要求を受けて応答を返す。クライアントが呼んで結果を待つ） | `add_server` は必須、`add_client` は任意 |
| ② カウンタのリセット | `counter_node` | `Trigger` | 1つのノードに、トピックの配信とサービスのサーバを同居させる | 任意 |

2つの組は、互いに呼び合わない。以降の節（2節の仕様、4節のPython版、5節のC++版、6節の実験）も、この①・②の順に分けて書く。

①足し算の流れ（`add_client` が呼ぶ場合）:

![add_clientが要求を送り、add_serverが足し算して応答を返す](img/phase3_4_add_seq.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
sequenceDiagram
    participant C as add_client
    participant S as add_server
    C->>C: wait_for_service（サーバが居るか待つ）
    C->>S: 要求 Request（a=3, b=4）
    S->>S: sum = a + b
    S-->>C: 応答 Response（sum=7）
    C->>C: 結果をログに出して終了
```

</details>

②カウンタのリセットの流れ（`Trigger`）:

![ros2 service callがreset_counterでcounter_nodeを呼び、counter_nodeは/counterを1秒ごとに出す](img/phase3_4_reset_flow.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart LR
    CLI["ros2 service call<br/>/reset_counter"] -- "Trigger要求（中身なし）" --> N["counter_node"]
    N -- "success, message" --> CLI
    N -- "/counter（Int32）を1秒ごと" --> OUT["ros2 topic echo"]
```

</details>

インターフェースの定義を確認しておく:

```bash
ros2 interface show example_interfaces/srv/AddTwoInts

ros2 interface show std_srvs/srv/Trigger
```

**期待する結果**:

```text
$ ros2 interface show example_interfaces/srv/AddTwoInts
int64 a
int64 b
---
int64 sum

$ ros2 interface show std_srvs/srv/Trigger
---
bool success   # indicate successful run of triggered service
string message # informational, e.g. for error messages
```

`---` の上が要求（Request）、下が応答（Response）。`Trigger` は `---` の上に何も無い、つまり要求が空の型である。各行は「型 名前」の形で、`int64`（64ビットの整数）・`bool`（真偽値）・`string`（文字列）は、ROS2のメッセージで使える基本の型である。基本の型の一覧は、公式ドキュメントの [Interfaces — ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/Concepts/Basic/About-Interfaces.html) の「Field types」の節にある。

## 2. 仕様

### 2-1. ①足し算: `add_server` / `add_client`

| ノード | 役割 | サービス（型） | 動作 |
|---|---|---|---|
| `add_server` | サーバ | `add_two_ints`（`example_interfaces/srv/AddTwoInts`） | 要求の `a` と `b` を足した `sum` を返し、ログに出す |
| `add_client` | クライアント | `add_two_ints` | ROSパラメータ `a`、`b`（整数、既定 1 と 2）を要求に入れて1回呼び、結果をログに出して終了。サーバがいなければ5秒待って諦める |

### 2-2. ②カウンタのリセット: `counter_node`（Trigger）

| 項目 | 内容 |
|---|---|
| ノード名・実行ファイル名 | `counter_node`（Python版・C++版で同一） |
| トピック（型） | `counter`（[`std_msgs/msg/Int32`](https://github.com/ros2/common_interfaces/blob/jazzy/std_msgs/msg/Int32.msg)）に、1秒ごとにカウントアップした値を送る（0から） |
| サービス（型） | `reset_counter`（`std_srvs/srv/Trigger`）。呼ばれたらカウントを0に戻し、`success=true`、`message` に「リセット前の値」を入れて返す |

## 3. 準備: 依存の追加

`package.xml` に、次の2行を足す（C++版を作る場合は `learn_cpp` 側にも。必須の範囲だけなら `learn_py` 側の `example_interfaces` だけでよい）。

```xml
<depend>example_interfaces</depend>
<depend>std_srvs</depend>
```

C++版を作る場合は、`ws/src/learn_cpp/CMakeLists.txt` に `find_package` も足す（既存の `find_package(...)` の並びへ）。

<!-- snippet: cmake_find_services -->
```cmake
find_package(example_interfaces REQUIRED)
find_package(std_srvs REQUIRED)
```

導入確認（`example_interfaces` は Desktop Install に含まれる。無い場合は `sudo apt install ros-jazzy-example-interfaces` で導入する）:

```bash
ros2 interface show example_interfaces/srv/AddTwoInts
```

**期待する結果**: 1節と同じく `int64 a` から始まる定義が表示されれば、導入済み。導入されていない場合は `Unknown package 'example_interfaces'` のようなエラーになる。

## 4. Python版（`ws/src/learn_py`）

`ws/src/learn_py/learn_py/` に、①足し算の `add_server.py`・`add_client.py` と、②カウンタのリセットの `counter_node.py` を作る（作るのは、進める範囲の分だけでよい）。

### 4-1. 主なAPI（rclpy）

| やりたいこと | API |
|---|---|
| サーバの作成 | `self.create_service(型, 'サービス名', コールバック)`。コールバックは `(request, response)` を受け、`response` を**返す** |
| クライアントの作成 | `self.create_client(型, 'サービス名')` |
| サーバの待ち合わせ | `client.wait_for_service(timeout_sec=5.0)`（戻り値が真偽） |
| 非同期の呼び出し | `future = client.call_async(request)` |
| 結果の待ち方 | `rclpy.spin_until_future_complete(node, future)` の後に `future.result()` |

非同期の呼び出しと結果の待ち方（`future` の意味と、待ち方の注意）は、クライアントを書く4-3節で説明する。

### 4-2. ①足し算: サーバ（`add_server.py`。必須）

ファイル: `ws/src/learn_py/learn_py/add_server.py`

<!-- file: ws/src/learn_py/learn_py/add_server.py -->
```python
import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node


# add_two_ints サービスのサーバ。要求の a と b を足して返すノード。
class AddServer(Node):
    # サービスを作り、要求が来たら on_request を呼ぶよう登録する。
    def __init__(self):
        super().__init__('add_server')
        self.srv = self.create_service(AddTwoInts, 'add_two_ints', self.on_request)

    # 要求1件ごとに呼ばれる。response.sum に答えを書き、response を返す。
    def on_request(self, request, response):
        response.sum = request.a + request.b
        self.get_logger().info(f'{request.a} + {request.b} = {response.sum}')
        return response


# エントリポイント（setup.py の entry_points から呼ばれる）。
# ノードを作って spin で回し、Ctrl+C で後片付けして終わる。
def main(args=None):
    rclpy.init(args=args)
    node = AddServer()
    try:
        rclpy.spin(node)
    except (KeyboardInterrupt, ExternalShutdownException):
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

**解説: `add_server.py`**

役割は「`add_two_ints` という名前のサービスを公開し、要求が来るたびに `a + b` を計算して返す」こと。全体の流れは、ノードを作る → サービスを登録する → `spin` で要求を待ち続ける、の3段階。トピックの `listener` とほぼ同じ骨組みで、違いは「受け取ったら結果を返す」点だけである。

| 部分 | 何をしているか |
|---|---|
| `super().__init__('add_server')` | ノード名を `add_server` にして親クラスを初期化する。 |
| `self.create_service(AddTwoInts, 'add_two_ints', self.on_request)` | 引数は「サービス型」「サービス名」「要求が来たとき呼ぶ関数」の3つ。型は `.srv` ファイルに対応し、要求（`a`, `b`）と応答（`sum`）の形を決める。戻り値（`self.srv`）は変数に保持しておく。 |
| `on_request(self, request, response)` | サービスのコールバック。`request` に相手が送った値が、`response` に「中身が空の応答」が入って渡される。 |
| `response.sum = request.a + request.b` | 応答の欄を埋める。`AddTwoInts` の `a`, `b`, `sum` は `int64`（`ros2 interface show` で確認できる）。 |
| `return response` | **Pythonでは応答を戻り値として返す**。`return` を忘れると、サーバ側のコールバックでエラーになり、クライアントは応答を受け取れない（つまずきやすい点）。 |

`main` は、フェーズ3-1のノードと同じ定型である。`rclpy.spin(node)` の中で、要求が来るとコールバックが呼ばれる。`except (KeyboardInterrupt, ExternalShutdownException)` は `Ctrl+C` での終了を静かに扱うため、`finally` は後片付け（ノードの破棄とシャットダウン）のため。

観察ポイント: サーバだけを起動しても何も起きず、ログも出ない（要求が来て初めて動く）。`ros2 service list -t` で `/add_two_ints [example_interfaces/srv/AddTwoInts]` が見えること、`ros2 service call` で呼んだときにサーバ側のログが出ることを確認する。

### 4-3. ①足し算: クライアント（`add_client.py`。任意）

4-2節のサーバを呼ぶ側である。

ファイル: `ws/src/learn_py/learn_py/add_client.py`

<!-- file: ws/src/learn_py/learn_py/add_client.py -->
```python
import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.node import Node


# add_two_ints サービスのクライアント。パラメータ a, b を持つノード。
class AddClient(Node):
    # パラメータ a, b を宣言し、クライアントを作る（まだ呼ばない）。
    def __init__(self):
        super().__init__('add_client')
        self.declare_parameter('a', 1)
        self.declare_parameter('b', 2)
        self.cli = self.create_client(AddTwoInts, 'add_two_ints')


# エントリポイント。サーバを最大5秒待ち、a + b を1回だけ呼んで
# 結果をログに出したら終わる（spin し続けない）。
def main(args=None):
    rclpy.init(args=args)
    node = AddClient()
    try:
        if not node.cli.wait_for_service(timeout_sec=5.0):
            node.get_logger().error('service add_two_ints is not available')
            return
        request = AddTwoInts.Request()
        request.a = node.get_parameter('a').value
        request.b = node.get_parameter('b').value
        future = node.cli.call_async(request)
        rclpy.spin_until_future_complete(node, future)
        if future.result() is not None:
            node.get_logger().info(
                f'{request.a} + {request.b} = {future.result().sum}')
        else:
            node.get_logger().error(f'call failed: {future.exception()}')
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

**解説: `add_client.py`**

役割は「サーバを見つけ、`a`, `b` を送って1回だけ呼び、答えをログに出して終わる」こと。サーバやパブリッシャと違い、`spin` し続けるノードではなく、**1回の仕事が終わったら `main` を抜けて終了する**形になっている。

- **`__init__`**: パラメータ `a`, `b` を既定値付きで宣言し（フェーズ3-3の復習）、`create_client(AddTwoInts, 'add_two_ints')` でクライアントを作る。クライアントを作っただけでは、まだ通信は始まらない。
- **`wait_for_service(timeout_sec=5.0)`**: サーバが見つかるまで最大5秒待ち、見つかれば `True`、見つからなければ `False` を返す。これを挟まずにいきなり呼ぶと、サーバ起動前だった場合に要求が届かない。`False` のときはエラーログを出して `return` する。
- **要求の組み立て**: `AddTwoInts.Request()` で空の要求を作り、`node.get_parameter('a').value` で取り出した値を詰める。`.value` を付け忘れると、`Parameter` オブジェクトそのものを詰めようとして型エラーになる。
- **`call_async(request)`**: 要求を送って、すぐ `future`（あとで結果が入る箱）を返す。ここでは結果はまだ無い。
- **`rclpy.spin_until_future_complete(node, future)`**: `future` に結果が入るまでノードを回す（この間に応答を受け取る処理が動く）。これが終わってから `future.result()` で `Response` を取り出す。`result()` が `None` のときは呼び出しが失敗しており、`future.exception()` で原因が見られる。

> なぜ「非同期」か: `call_async` は応答を待たずにすぐ戻るので、「結果をいつ取りに行くか」は自分で決める必要がある。このコードは `main` の中で `spin_until_future_complete` を使って待つ。ノードのコールバックの中から別のサービスを呼びたい場合は、この待ち方は使えないので、futureの完了時に呼ばれるコールバックで結果を受ける、などの別の方法になる（このフェーズでは扱わない）。

> **重要**: `spin_until_future_complete` は、**コールバックの中では呼ばない**。`main` の中で使う。コールバックは `spin` に呼ばれて動いているので、その中でさらにノードを回して応答を待とうとすると、応答を受け取る処理が順番待ちのまま動けず、固まる。

落とし穴:

- `wait_for_service` が `False` のとき、このコードは `return` するだけで、終了コードは0のままになる（C++版は `return 1`）。スクリプトから結果を判定したい場合は、ここが差になる。
- `-p a:=3.0` のように実数を渡すと、宣言した型（整数）と合わず、パラメータの設定でエラーになる。整数で渡す。

動作確認: サーバを起動した状態で `ros2 run learn_py add_client`（既定なら `1 + 2 = 3`）と、`ros2 run learn_py add_client --ros-args -p a:=10 -p b:=20` を実行する。サーバを止めた状態で実行すると、約5秒後にエラーログが出て終わることも確認する（表示は6-1の「期待する結果」を参照）。

### 4-4. ②カウンタのリセット（`counter_node.py`。任意）

ここからは、①足し算とは別の組である。`add_server`・`add_client` とは関係なく、このノード1つで動く。

ファイル: `ws/src/learn_py/learn_py/counter_node.py`

<!-- file: ws/src/learn_py/learn_py/counter_node.py -->
```python
import rclpy
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node
from std_msgs.msg import Int32
from std_srvs.srv import Trigger


# 1秒ごとにカウントを counter へ送り、reset_counter サービスで0に戻せるノード。
class CounterNode(Node):
    # カウント・Publisher・タイマー・サービスを用意する。
    def __init__(self):
        super().__init__('counter_node')
        self.count = 0
        self.pub = self.create_publisher(Int32, 'counter', 10)
        self.timer = self.create_timer(1.0, self.on_timer)
        self.srv = self.create_service(Trigger, 'reset_counter', self.on_reset)

    # 今のカウントを送ってから1増やす。
    def on_timer(self):
        msg = Int32()
        msg.data = self.count
        self.pub.publish(msg)
        self.count += 1

    # reset_counter が呼ばれたとき、リセット前の値を応答に入れてから0に戻す。
    def on_reset(self, request, response):
        response.success = True
        response.message = f'counter reset (was {self.count})'
        self.count = 0
        self.get_logger().info(response.message)
        return response


# エントリポイント（setup.py の entry_points から呼ばれる）。
# ノードを作って spin で回し、Ctrl+C で後片付けして終わる。
def main(args=None):
    rclpy.init(args=args)
    node = CounterNode()
    try:
        rclpy.spin(node)
    except (KeyboardInterrupt, ExternalShutdownException):
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

**解説: `counter_node.py`**

役割は「1秒ごとにカウントを配信しつつ、`reset_counter` サービスで0に戻せる」こと。**1つのノードにトピックの発行（タイマー）とサービスのサーバが同居する**例で、トピックは継続的な流れ、サービスは必要なときの操作、という使い分け（7節「トピックとサービスの使い分け」で整理する）を1本のノードで体感できる。

| 部分 | 何をしているか |
|---|---|
| `self.count = 0` | ノードが持つ状態。タイマーのコールバックとサービスのコールバックの**両方が同じ変数を読み書きする**。 |
| `create_publisher` / `create_timer` | フェーズ3-1と同じ。1秒ごとに `on_timer` が呼ばれる。 |
| `create_service(Trigger, 'reset_counter', self.on_reset)` | `Trigger` は要求が空で、応答が `success`（真偽）と `message`（文字列）だけの標準サービス型。「引数の要らない合図」を送る用途に向く。 |
| `on_timer` | 今の `count` を配信してから `+1` する。最初に配信される値は0。 |
| `on_reset` | `success` を真にし、`message` に**リセット前の値**を入れてから `count` を0に戻す。順序が大事で、先に0にすると「リセット前の値」が取れない。 |

`on_reset` の引数 `request` は、`Trigger` の要求が空なので使っていない。それでも引数としては必要である（省くとコールバックの呼び出しでエラーになる）。

同じ変数を2つのコールバックが触っても安全なのは、`rclpy.spin` の既定の実行方式（シングルスレッド）では、コールバックが**同時には動かず1つずつ順に処理される**ため。複数スレッドの実行方式に変えると、この前提が崩れて排他制御が必要になる（このフェーズでは扱わない）。

観察ポイント: `ros2 topic echo /counter` を見ながら `ros2 service call /reset_counter std_srvs/srv/Trigger` を実行し、値が0に戻ること、応答の `message` に直前の値が入っていることを確認する（6-2節）。

### 4-5. 登録とビルド（①・②共通）

`setup.py` の `entry_points` に、作ったノードの行を足し、再ビルドする。下の3行のうち、1行目が①の `add_server`、2行目が①の `add_client`、3行目が②の `counter_node` である（作らなかったノードの行は書かない）。

<!-- snippet: py_entry_points_service -->
```python
            'add_server = learn_py.add_server:main',
            'add_client = learn_py.add_client:main',
            'counter_node = learn_py.counter_node:main',
```

```bash
cd ~/work/ros2MinimalPhysicalAi/ws

colcon build --symlink-install --packages-select learn_py

source install/setup.bash
```

**期待する結果**: `Finished <<< learn_py` と `Summary: 1 package finished` が出れば成功（表示の形は、[フェーズ3-1](phase3_1_pubsub.md)の3-3節と同じ）。`source` は、成功しても何も表示しない。

## 5. C++版（`ws/src/learn_cpp`）

> **このフェーズのC++版は任意（発展）**。フェーズ5の車両シミュレーションはPythonで実装すると決めているため、ここでC++版を作らなくても先へ進める。Python版との違いは、8節の「Python版とC++版の違いのまとめ」を読めば概要が掴める。C++版を作らない場合は、C++向けの依存の追加（`package.xml` と `CMakeLists.txt`）も不要。

`ws/src/learn_cpp/src/` に、①足し算の `add_server.cpp`・`add_client.cpp` と、②カウンタのリセットの `counter_node.cpp` を作る。

### 5-1. 主なAPI（rclcpp）

| やりたいこと | API |
|---|---|
| サーバの作成 | `create_service<型>("サービス名", コールバック)`。コールバックは `(Request::SharedPtr, Response::SharedPtr)` を受け、`response` に**書き込む**（Pythonと違い戻り値は無い） |
| クライアントの作成 | `create_client<型>("サービス名")` |
| サーバの待ち合わせ | `client->wait_for_service(5s)`（戻り値が真偽） |
| 非同期の呼び出し | `auto future = client->async_send_request(request);` |
| 結果の待ち方 | `rclcpp::spin_until_future_complete(node, future)` が `SUCCESS` なら `future.get()` |

Pythonとの違いの見どころ: 応答を「返す」のか「書き込む」のか、型名が `Request`/`Response` のネストした型になること。

### 5-2. ①足し算: サーバ（`add_server.cpp`）

ファイル: `ws/src/learn_cpp/src/add_server.cpp`

<!-- file: ws/src/learn_cpp/src/add_server.cpp -->
```cpp
#include <memory>

#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using AddTwoInts = example_interfaces::srv::AddTwoInts;

// add_two_ints サービスのサーバ。要求の a と b を足して返すノード（add_server.py と同じ仕様）。
class AddServer : public rclcpp::Node
{
public:
  // コンストラクタ: サービスを作る。要求が来たときの処理はラムダで直接書く。
  AddServer() : Node("add_server")
  {
    service_ = create_service<AddTwoInts>(
      "add_two_ints",
      [this](
        const std::shared_ptr<AddTwoInts::Request> request,
        std::shared_ptr<AddTwoInts::Response> response) {
        response->sum = request->a + request->b;
        RCLCPP_INFO(
          get_logger(), "%lld + %lld = %lld",
          static_cast<long long>(request->a), static_cast<long long>(request->b),
          static_cast<long long>(response->sum));
      });
  }

private:
  rclcpp::Service<AddTwoInts>::SharedPtr service_;
};

// エントリポイント。ノードを作って spin で回し、Ctrl+C で spin を抜けて終わる。
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<AddServer>());
  rclcpp::shutdown();
  return 0;
}
```

**解説: `add_server.cpp`**

Python版と同じ「要求を受けて `a + b` を返す」サーバ。差分を中心に読む。

- **`using AddTwoInts = ...`**: 長い型名に別名を付けている。`example_interfaces::srv::AddTwoInts` は `.srv` から自動生成された型で、`Request` と `Response` がその中にネストしている（`AddTwoInts::Request`）。ヘッダ名は `add_two_ints.hpp`（型名をスネークケースにしたもの）で、`CMakeLists.txt` の依存に `example_interfaces` が必要。
- **`create_service<AddTwoInts>("add_two_ints", ラムダ式)`**: 型はテンプレート引数で、サービス名とコールバックを渡す。戻り値は `rclcpp::Service<AddTwoInts>::SharedPtr` で、**メンバ変数 `service_` に保持する**。保持しないとこの行の終わりで破棄され、サービスが消える。
- **コールバックの引数**: `(request, response)` の両方が `shared_ptr` で渡されるので、`->` で欄にアクセスする。**応答は戻り値ではなく `response` に書き込む**（Pythonとの最大の違い）。ラムダは `[this]` で自分自身（ノード）を捕まえており、`get_logger()` を呼ぶために必要。
- **`static_cast<long long>` と `%lld`**: `a`, `b`, `sum` は `int64_t`。`printf` 系の書式指定は環境によって `int64_t` の実体（`long` か `long long`）が違うため、`long long` に揃えて `%lld` で出す、という安全策である。
- **`main`**: `rclcpp::spin(std::make_shared<AddServer>())` で、ノードの生成と待ち受けを1行で行う。`Ctrl+C` で `spin` から戻り、`shutdown()` で終わる。

観察ポイント: Python版のサーバと**入れ替えても、同じクライアントから同じ結果が返る**こと（サービス名と型が同じなら、実装言語は関係ない）。

### 5-3. ①足し算: クライアント（`add_client.cpp`）

ファイル: `ws/src/learn_cpp/src/add_client.cpp`

<!-- file: ws/src/learn_cpp/src/add_client.cpp -->
```cpp
#include <chrono>
#include <cstdint>
#include <memory>

#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using namespace std::chrono_literals;
using AddTwoInts = example_interfaces::srv::AddTwoInts;

// エントリポイント。ノードとクライアントをこの中で作り、サーバを最大5秒待って
// a + b を1回だけ呼び、結果をログに出したら終わる（クラスは作らない）。
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  auto node = rclcpp::Node::make_shared("add_client");
  const int64_t a = node->declare_parameter<int64_t>("a", 1);
  const int64_t b = node->declare_parameter<int64_t>("b", 2);

  auto client = node->create_client<AddTwoInts>("add_two_ints");
  if (!client->wait_for_service(5s)) {
    RCLCPP_ERROR(node->get_logger(), "service add_two_ints is not available");
    rclcpp::shutdown();
    return 1;
  }

  auto request = std::make_shared<AddTwoInts::Request>();
  request->a = a;
  request->b = b;
  auto future = client->async_send_request(request);
  if (rclcpp::spin_until_future_complete(node, future) == rclcpp::FutureReturnCode::SUCCESS) {
    RCLCPP_INFO(
      node->get_logger(), "%lld + %lld = %lld",
      static_cast<long long>(a), static_cast<long long>(b),
      static_cast<long long>(future.get()->sum));
  } else {
    RCLCPP_ERROR(node->get_logger(), "call failed");
  }
  rclcpp::shutdown();
  return 0;
}
```

**解説: `add_client.cpp`**

Python版と同じ流れ（待つ → 要求を作る → 非同期に送る → 結果を待つ）を、`main` の中に順に書いている。クラスを作らず `rclcpp::Node::make_shared("add_client")` でノードを直接作るのが、Python版との構造上の違い。

- **`declare_parameter<int64_t>("a", 1)`**: 型をテンプレート引数で指定して宣言し、宣言時に値を受け取る。Python版の `declare_parameter` + `get_parameter(...).value` の2段階が1行にまとまっている。
- **`using namespace std::chrono_literals;` と `wait_for_service(5s)`**: `5s` は「5秒」を表すリテラルで、この `using` が無いとコンパイルできない。戻り値が `false` ならエラーログを出し、`rclcpp::shutdown()` して**終了コード1**で終わる（Python版は0のまま）。
- **`make_shared<AddTwoInts::Request>()`**: 要求は `shared_ptr` で作り、`request->a = a;` のように詰める。
- **`async_send_request(request)`**: Pythonの `call_async` に相当し、`future`（`shared_future`）を返す。
- **`spin_until_future_complete(node, future)`**: 戻り値は `FutureReturnCode`。`SUCCESS` のときだけ `future.get()` で応答（`shared_ptr`）を取り出し、`->sum` を読む。それ以外（`TIMEOUT` や `INTERRUPTED`）は失敗として扱う。Pythonの `result() is not None` の判定に当たる。

落とし穴: `future.get()` は、結果が入っていない状態で呼ぶと待たされるか例外になる。**先に戻り値が `SUCCESS` であることを確認してから**呼ぶ、という順序を守る。また、`spin_until_future_complete` を**サービスのコールバックの中で呼ぶ**と固まる（4-3節の `add_client.py` の解説の末尾にある「重要」と同じ理由）。

動作確認: Python版と同様に、`ros2 run learn_cpp add_client` と `--ros-args -p a:=10 -p b:=20` を、サーバありとなしの両方で試す。サーバなしのときは約5秒後にエラーログが出て、`echo $?` で終了コード1が見える（表示は6-1節の「期待する結果」を参照）。

### 5-4. ②カウンタのリセット（`counter_node.cpp`）

ここからは、①足し算とは別の組である（4-4節のPython版と同じ仕様）。

ファイル: `ws/src/learn_cpp/src/counter_node.cpp`

<!-- file: ws/src/learn_cpp/src/counter_node.cpp -->
```cpp
#include <chrono>
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/int32.hpp"
#include "std_srvs/srv/trigger.hpp"

using namespace std::chrono_literals;

// 1秒ごとにカウントを counter へ送り、reset_counter サービスで0に戻せるノード（counter_node.py と同じ仕様）。
class CounterNode : public rclcpp::Node
{
public:
  // コンストラクタ: Publisher・タイマー・サービスを作る。リセットの処理はラムダで直接書く。
  CounterNode() : Node("counter_node")
  {
    pub_ = create_publisher<std_msgs::msg::Int32>("counter", 10);
    timer_ = create_wall_timer(1s, [this]() { on_timer(); });
    service_ = create_service<std_srvs::srv::Trigger>(
      "reset_counter",
      [this](
        const std::shared_ptr<std_srvs::srv::Trigger::Request>,
        std::shared_ptr<std_srvs::srv::Trigger::Response> response) {
        response->success = true;
        response->message = "counter reset (was " + std::to_string(count_) + ")";
        count_ = 0;
        RCLCPP_INFO(get_logger(), "%s", response->message.c_str());
      });
  }

private:
  // 今のカウントを送ってから1増やす。
  void on_timer()
  {
    std_msgs::msg::Int32 msg;
    msg.data = count_++;
    pub_->publish(msg);
  }

  rclcpp::Publisher<std_msgs::msg::Int32>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
  rclcpp::Service<std_srvs::srv::Trigger>::SharedPtr service_;
  int count_ = 0;
};

// エントリポイント。ノードを作って spin で回し、Ctrl+C で spin を抜けて終わる。
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<CounterNode>());
  rclcpp::shutdown();
  return 0;
}
```

**解説: `counter_node.cpp`**

Python版と同じく、タイマーによる配信と `reset_counter` サービスを1ノードに同居させている。

- **`create_wall_timer(1s, [this]() { on_timer(); })`**: 1秒周期のタイマー。ラムダ経由でメンバ関数を呼んでいる。
- **`create_service<std_srvs::srv::Trigger>(...)`**: `Trigger` の型もヘッダ（`std_srvs/srv/trigger.hpp`）も `std_srvs` パッケージのもので、`CMakeLists.txt` の依存に `std_srvs` が要る。
- **コールバックの第1引数（要求）に名前が無い**: `Trigger` の要求は空で使わないため、あえて変数名を書かない。名前を付けて使わないと「未使用引数」の警告が出るので、この書き方が定番である。
- **`response->message = "counter reset (was " + std::to_string(count_) + ")";`**: 文字列とカウンタを連結して、リセット前の値を入れる。その後で `count_ = 0;`。順序はPython版と同じで、先に0にするとリセット前の値が取れない。ログ出力の `%s` には `c_str()` が要る。
- **`msg.data = count_++;`**: 「今の値を `data` に入れてから、`count_` を1増やす」（後置インクリメント）。配信される最初の値は0で、Python版の「配信してから加算」と同じ結果になる。
- **メンバ変数**: `pub_`, `timer_`, `service_` はいずれも保持が必要（消えると機能が止まる）。`count_` は、`spin` の既定ではコールバックが1つずつ順に呼ばれるので、排他制御なしで2つのコールバックから触れる。

観察ポイント: Python版と同じ実験（6-2節）で、値が0に戻ること、`message` にリセット前の値が入ることを確認する。Python版とC++版で結果が同じになるかも見比べる。

### 5-5. 登録とビルド（①・②共通）

`CMakeLists.txt` に追記し、`install(TARGETS ...)` に名前を足す（これまでの分は残す。前のフェーズでC++版を作らなかったノードの名前は書かない。書くと、存在しないターゲットとしてビルドがエラーになる）。`add_executable` の3組のうち、最初の2組が①足し算、3組目が②カウンタのリセットである。

<!-- snippet: cmake_service -->
```cmake
add_executable(add_server src/add_server.cpp)
ament_target_dependencies(add_server rclcpp example_interfaces)

add_executable(add_client src/add_client.cpp)
ament_target_dependencies(add_client rclcpp example_interfaces)

add_executable(counter_node src/counter_node.cpp)
ament_target_dependencies(counter_node rclcpp std_msgs std_srvs)

install(TARGETS
  hello
  talker
  listener
  sine_pub
  sine_sub
  turtle_circle
  qos_talker
  qos_listener
  param_talker
  add_server
  add_client
  counter_node
  DESTINATION lib/${PROJECT_NAME})
```

```bash
cd ~/work/ros2MinimalPhysicalAi/ws

colcon build --symlink-install --packages-select learn_cpp

source install/setup.bash
```

**期待する結果**: `Finished <<< learn_cpp` と `Summary: 1 package finished` が出れば成功（表示の形は、[フェーズ3-1](phase3_1_pubsub.md)の3-3節と同じ）。`source` は、成功しても何も表示しない。

## 6. 実験

### 6-1. ①足し算（`add_server`・`add_client`）

必須の範囲だけを進めている場合は、T1でサーバを起動したら、T2のクライアントの代わりに、この後の「CLIからも呼べる」の `ros2 service call` を使う。

```bash
# T1
ros2 run learn_py add_server

# T2（add_client を作った場合）
ros2 run learn_py add_client --ros-args -p a:=3 -p b:=4
```

**期待する結果**: T2のクライアントは1行出して、すぐに終了する（プロンプトに戻る）。T1のサーバは、呼ばれるたびに1行ずつログを出し、動き続ける。

```text
# T1（add_server）
[INFO] [1790262000.500000000] [add_server]: 3 + 4 = 7

# T2（add_client）
[INFO] [1790262000.501000000] [add_client]: 3 + 4 = 7
```

同じ計算結果が、サーバ側（要求を受けて計算した記録）とクライアント側（応答を受け取った記録）の両方に出る。サーバを起動せずにクライアントだけを動かすと、約5秒待った後に次のエラーを出して終わる（この節の末尾の課題1で試す）。

```text
[ERROR] [1790262010.000000000] [add_client]: service add_two_ints is not available
```

CLIからも呼べる:

```bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 10, b: 20}"
```

**期待する結果**:

```text
waiting for service to become available...
requester: making request: example_interfaces.srv.AddTwoInts_Request(a=10, b=20)

response:
example_interfaces.srv.AddTwoInts_Response(sum=30)
```

T1のサーバには `10 + 20 = 30` のログが出る。サーバから見ると、自作のクライアントからの要求もCLIからの要求も区別が無い。4つの言語の組み合わせ（下の表）でも、表示は同じになる。

言語の組み合わせを試す（`add_client` とC++版を作った場合。任意）:

| 組み合わせ | T1 | T2 |
|---|---|---|
| Python × Python | `ros2 run learn_py add_server` | `ros2 run learn_py add_client ...` |
| Python × C++ | `ros2 run learn_py add_server` | `ros2 run learn_cpp add_client ...` |
| C++ × Python | `ros2 run learn_cpp add_server` | `ros2 run learn_py add_client ...` |
| C++ × C++ | `ros2 run learn_cpp add_server` | `ros2 run learn_cpp add_client ...` |

> 課題1（`add_client` を作った場合）: サーバを**起動しない**でクライアントを起動し、5秒待って諦めるメッセージが出ることを確認する。
>
> 課題2（`add_client` を作った場合）: クライアントを先に起動し、5秒以内にサーバを起動して、呼び出しが成功することを確認する（`wait_for_service` の効果）。
>
> 課題3: `ros2 service list -t`、`ros2 service type /add_two_ints`、`ros2 node info /add_server` で、サービスがどう見えるか確認する。
>
> 課題4: サーバ（`add_server`）の `on_request` の中で `time.sleep(3)`（C++は `std::this_thread::sleep_for`）を入れて、クライアントの呼び出しが3秒ブロックされることを確認する。その3秒の間に、別のターミナルから `ros2 service call` でもう1回呼ぶとどうなるかも考える（既定の `spin` は1つのスレッドでコールバックを1つずつ順に処理するので、1つ目の処理が終わるまで2つ目は待たされる）。

### 6-2. ②カウンタのリセット（`counter_node`）（任意）

```bash
# T1
ros2 run learn_py counter_node

# T2
ros2 topic echo /counter

# T3
ros2 service call /reset_counter std_srvs/srv/Trigger
```

**期待する結果**（T2で数字が7まで進んだところでT3を実行した例）:

```text
# T2（ros2 topic echo /counter）
data: 6
---
data: 7
---
data: 0
---
data: 1
---

# T3（ros2 service call）
requester: making request: std_srvs.srv.Trigger_Request()

response:
std_srvs.srv.Trigger_Response(success=True, message='counter reset (was 8)')

# T1（counter_node）
[INFO] [1790262100.300000000] [counter_node]: counter reset (was 8)
```

- `counter_node` は配信のたびにはログを出さず、リセットされたときだけ1行出す。
- 応答の `was 8` が、最後に見えた `7` より1大きいのは、`on_timer` が「配信してから `+1`」する作りのため。`count` には「次に配信する予定の値」が入っている。どちらの値を返すのが仕様として自然か、考えてみるとよい。

`echo` の値が0に戻り、サービスの応答に `success=True` と `message`（リセット前の値）が入っていることを確認する。C++版を作った場合は、C++版でも同様に確認する。

> 課題5（発展）: `Trigger` のクライアントを自分で書き、`counter_node` をリセットするだけのノードを作る（AddTwoIntsのクライアントとの違いは、要求に中身がない点だけ）。

## 7. トピックとサービスの使い分け

| 観点 | トピック | サービス |
|---|---|---|
| 通信の形 | 一方向・継続的 | 要求→応答（1回） |
| 相手を待つか | 待たない | 応答を待つ（非同期でも結果を待つ） |
| 向く用途 | センサ値、指令の連続送信 | 状態のリセット、設定の取得、一度だけの計算 |
| フェーズ5での使いどころ | 速度、制御指令 | シミュレーションのリセット、制御の有効/無効 |

長くかかる処理で、途中経過を知りたい場合は、サービスではなく**アクション**（フェーズ3-5）を使う。

## 8. Python版とC++版の違いのまとめ

| 観点 | Python | C++ |
|---|---|---|
| サーバのコールバック（応答の返し方） | `(request, response)` を受け取り、`response` を埋めて**`return response`する**（忘れるとエラー） | `(Request::SharedPtr, Response::SharedPtr)` を受け取り、戻り値は無く`response->sum = ...`のように**直接書き込む** |
| クライアントの構造 | `Node`を継承したクラスを作り、`main`から呼ぶ | クラスを作らず`rclcpp::Node::make_shared("add_client")`で直接ノードを作り、`main`の中に手続き的に書く |
| `wait_for_service` | `client.wait_for_service(timeout_sec=5.0)` | `client->wait_for_service(5s)`（`std::chrono_literals`の`5s`を使う） |
| 結果の待ち方 | `rclpy.spin_until_future_complete(node, future)` の後、`future.result()`（`None`なら失敗） | `rclcpp::spin_until_future_complete(node, future)`の戻り値が`SUCCESS`かを判定し、成功なら`future.get()` |
| 型名（`Request`/`Response`） | `AddTwoInts.Request()` | `AddTwoInts::Request`（`::`でネストした型としてアクセス。`using AddTwoInts = example_interfaces::srv::AddTwoInts;`で別名を付けるのが定石） |
| サーバ未起動時の終了コード | `return`するだけで終了コードは`0`のまま | `return 1`で明示的に異常終了を示す |

サーバ側の「応答を返す（Python）」と「応答に書き込む（C++）」の違いが最も間違えやすい。C++で`return`を書いてもコンパイルエラーにならない（単に無視される）ため、気づきにくい落とし穴になる。

## 9. つまずきやすい点

| 症状 | 確認すること |
|---|---|
| クライアントが固まる | サーバが動いているか。`ros2 service list` に出るか。サービス名の綴り |
| Pythonのクライアントが無反応 | `spin_until_future_complete` を呼んでいるか。コールバック内で呼んでいないか |
| C++で `%ld` の警告 | `int64_t` は環境で `long` / `long long` が異なるため、`static_cast<long long>` と `%lld` にそろえる |
| サーバが応答しない（Python） | コールバックの最後で `return response` しているか |
| ビルドで `example_interfaces` が見つからない | `package.xml` の `<depend>`、`CMakeLists.txt` の `find_package(example_interfaces REQUIRED)` |
| `ros2 service call` の引数が通らない | YAMLの書式（`"{a: 10, b: 20}"`、コロンの後ろに空白） |

## 10. 次へ

フェーズ3-5（[`docs/phase3_5_actions.md`](phase3_5_actions.md)）で、途中経過を返せる「アクション」を扱う。

## 11. 公式ドキュメント・参考資料

確認状況（2026-09-20）: 下記は [`docs/idea_origin.md`](idea_origin.md) に掲載済みのURLで、今回は再確認していない（docs.ros.orgは本文取得がボット対策で拒否される）。

### 公式（ROS 2 Jazzy）

- [Writing a simple service and client (Python) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Service-And-Client.html)
- [Writing a simple service and client (C++) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.html)
- [ros2/common_interfaces（GitHub）](https://github.com/ros2/common_interfaces)（`std_srvs` の定義）

> 公式ドキュメントと食い違う場合は、公式を優先する。

> 出典: 各サンプルのAPIの使い方は、上記の公式チュートリアルを参考にした（ROS 2ドキュメントはCC BY 4.0）。ノード名・仕様・コード・文章は独自に書いたもので、逐語の転載ではない。期待する結果に載せた `ros2 interface show` の表示は、型の定義（`example_interfaces`・`common_interfaces`、Apache License 2.0）である。
