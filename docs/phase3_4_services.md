# フェーズ3-4 手順書: サービス（要求と応答）

`docs/learning_plan.md` フェーズ3（idea_origin.md ステップ1の1-4）に対応する。「1回の要求に1回の応答を返す」通信を、Python・C++の両方で書く。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy
- 前提: フェーズ3-1〜3-3完了
- 所要目安: 1コマ
- 言語: **Python・C++の両方**
- 使う標準インターフェース: `example_interfaces/srv/AddTwoInts`、`std_srvs/srv/Trigger`

> **進め方**: 仕様を見て自分で書き、詰まったら「サンプルコード」で答え合わせをする。サンプルはこの手順書の作成時にビルド確認済みで、ノードの実行結果は未確認（出力が違えば差分を貼ってほしい）。

## 0. 学習目標と完了条件

1. サービスのサーバ（`create_service`）とクライアント（`create_client`）を、Python・C++の両方で書ける。
2. 標準の `.srv` 定義（要求と応答の項目）を `ros2 interface show` で読める。
3. クライアントの**非同期呼び出し**（`call_async` / `async_send_request`）と、結果の待ち方を説明できる。
4. トピックとの使い分け（継続的な流れ vs 要求→応答）を、自分の言葉で説明できる。

## 1. 全体像

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

リセット用サービス（`Trigger`）の使い方:

```mermaid
flowchart LR
    CLI["ros2 service call<br/>/reset_counter"] -- "Trigger要求（中身なし）" --> N["counter_node"]
    N -- "success, message" --> CLI
    N -- "/counter（Int32）を1秒ごと" --> OUT["ros2 topic echo"]
```

インターフェースの定義を確認しておく:

```bash
ros2 interface show example_interfaces/srv/AddTwoInts
ros2 interface show std_srvs/srv/Trigger
```

`---` の上が要求（Request）、下が応答（Response）。

## 2. 仕様

### 2-1. `add_server` / `add_client`

| ノード | 役割 | サービス（型） | 動作 |
|---|---|---|---|
| `add_server` | サーバ | `add_two_ints`（`example_interfaces/srv/AddTwoInts`） | 要求の `a` と `b` を足した `sum` を返し、ログに出す |
| `add_client` | クライアント | `add_two_ints` | ROSパラメータ `a`、`b`（整数、既定 1 と 2）を要求に入れて1回呼び、結果をログに出して終了。サーバがいなければ5秒待って諦める |

### 2-2. `counter_node`（Trigger）

| 項目 | 内容 |
|---|---|
| ノード名・実行ファイル名 | `counter_node`（Python版・C++版で同一） |
| トピック（型） | `counter`（`std_msgs/msg/Int32`）に、1秒ごとにカウントアップした値を送る（0から） |
| サービス（型） | `reset_counter`（`std_srvs/srv/Trigger`）。呼ばれたらカウントを0に戻し、`success=true`、`message` に「リセット前の値」を入れて返す |

## 3. 準備: 依存の追加

両パッケージの `package.xml` に、次の2行を足す。

```xml
<depend>example_interfaces</depend>
<depend>std_srvs</depend>
```

C++（`ws/src/learn_cpp/CMakeLists.txt`）には、`find_package` も足す（既存の `find_package(...)` の並びへ）。

<!-- snippet: cmake_find_services -->
```cmake
find_package(example_interfaces REQUIRED)
find_package(std_srvs REQUIRED)
```

導入確認（`example_interfaces` が無い場合は導入をユーザーが行う）:

```bash
ros2 interface show example_interfaces/srv/AddTwoInts
```

## 4. Python版（`ws/src/learn_py`）

`ws/src/learn_py/learn_py/` に `add_server.py`, `add_client.py`, `counter_node.py` を作る。

主なAPI（rclpy）:

| やりたいこと | API |
|---|---|
| サーバの作成 | `self.create_service(型, 'サービス名', コールバック)`。コールバックは `(request, response)` を受け、`response` を**返す** |
| クライアントの作成 | `self.create_client(型, 'サービス名')` |
| サーバの待ち合わせ | `client.wait_for_service(timeout_sec=5.0)`（戻り値が真偽） |
| 非同期の呼び出し | `future = client.call_async(request)` |
| 結果の待ち方 | `rclpy.spin_until_future_complete(node, future)` の後に `future.result()` |

> **重要**: `spin_until_future_complete` は、**コールバックの中では呼ばない**（自分自身を止めて固まる）。`main` の中で使う。

<details>
<summary>サンプルコード（答え合わせ用）</summary>

ファイル: `ws/src/learn_py/learn_py/add_server.py`

<!-- file: ws/src/learn_py/learn_py/add_server.py -->
```python
import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node


class AddServer(Node):
    def __init__(self):
        super().__init__('add_server')
        self.srv = self.create_service(AddTwoInts, 'add_two_ints', self.on_request)

    def on_request(self, request, response):
        response.sum = request.a + request.b
        self.get_logger().info(f'{request.a} + {request.b} = {response.sum}')
        return response


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

ファイル: `ws/src/learn_py/learn_py/add_client.py`

<!-- file: ws/src/learn_py/learn_py/add_client.py -->
```python
import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.node import Node


class AddClient(Node):
    def __init__(self):
        super().__init__('add_client')
        self.declare_parameter('a', 1)
        self.declare_parameter('b', 2)
        self.cli = self.create_client(AddTwoInts, 'add_two_ints')


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

ファイル: `ws/src/learn_py/learn_py/counter_node.py`

<!-- file: ws/src/learn_py/learn_py/counter_node.py -->
```python
import rclpy
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node
from std_msgs.msg import Int32
from std_srvs.srv import Trigger


class CounterNode(Node):
    def __init__(self):
        super().__init__('counter_node')
        self.count = 0
        self.pub = self.create_publisher(Int32, 'counter', 10)
        self.timer = self.create_timer(1.0, self.on_timer)
        self.srv = self.create_service(Trigger, 'reset_counter', self.on_reset)

    def on_timer(self):
        msg = Int32()
        msg.data = self.count
        self.pub.publish(msg)
        self.count += 1

    def on_reset(self, request, response):
        response.success = True
        response.message = f'counter reset (was {self.count})'
        self.count = 0
        self.get_logger().info(response.message)
        return response


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

</details>

`setup.py` の `entry_points` に3行を足し、再ビルドする。

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

## 5. C++版（`ws/src/learn_cpp`）

`ws/src/learn_cpp/src/` に `add_server.cpp`, `add_client.cpp`, `counter_node.cpp` を作る。

主なAPI（rclcpp）:

| やりたいこと | API |
|---|---|
| サーバの作成 | `create_service<型>("サービス名", コールバック)`。コールバックは `(Request::SharedPtr, Response::SharedPtr)` を受け、`response` に**書き込む**（Pythonと違い戻り値は無い） |
| クライアントの作成 | `create_client<型>("サービス名")` |
| サーバの待ち合わせ | `client->wait_for_service(5s)`（戻り値が真偽） |
| 非同期の呼び出し | `auto future = client->async_send_request(request);` |
| 結果の待ち方 | `rclcpp::spin_until_future_complete(node, future)` が `SUCCESS` なら `future.get()` |

Pythonとの違いの見どころ: 応答を「返す」のか「書き込む」のか、型名が `Request`/`Response` のネストした型になること。

<details>
<summary>サンプルコード（答え合わせ用）</summary>

ファイル: `ws/src/learn_cpp/src/add_server.cpp`

<!-- file: ws/src/learn_cpp/src/add_server.cpp -->
```cpp
#include <memory>

#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using AddTwoInts = example_interfaces::srv::AddTwoInts;

class AddServer : public rclcpp::Node
{
public:
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

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<AddServer>());
  rclcpp::shutdown();
  return 0;
}
```

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

class CounterNode : public rclcpp::Node
{
public:
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

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<CounterNode>());
  rclcpp::shutdown();
  return 0;
}
```

</details>

`CMakeLists.txt` に追記し、`install(TARGETS ...)` に名前を足す（これまでの分は残す）。

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

## 6. 実験

### 6-1. AddTwoInts

```bash
# T1
ros2 run learn_py add_server
# T2
ros2 run learn_py add_client --ros-args -p a:=3 -p b:=4
```

CLIからも呼べる:

```bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 10, b: 20}"
```

言語の組み合わせを試す:

| 組み合わせ | T1 | T2 |
|---|---|---|
| Python × Python | `ros2 run learn_py add_server` | `ros2 run learn_py add_client ...` |
| Python × C++ | `ros2 run learn_py add_server` | `ros2 run learn_cpp add_client ...` |
| C++ × Python | `ros2 run learn_cpp add_server` | `ros2 run learn_py add_client ...` |
| C++ × C++ | `ros2 run learn_cpp add_server` | `ros2 run learn_cpp add_client ...` |

> 課題1: サーバを**起動しない**でクライアントを起動し、5秒待って諦めるメッセージが出ることを確認する。
>
> 課題2: クライアントを先に起動し、5秒以内にサーバを起動して、呼び出しが成功することを確認する（`wait_for_service` の効果）。
>
> 課題3: `ros2 service list -t`、`ros2 service type /add_two_ints`、`ros2 node info /add_server` で、サービスがどう見えるか確認する。

### 6-2. Trigger（カウンタのリセット）

```bash
# T1
ros2 run learn_py counter_node
# T2
ros2 topic echo /counter
# T3
ros2 service call /reset_counter std_srvs/srv/Trigger
```

`echo` の値が0に戻り、サービスの応答に `success=True` と `message`（リセット前の値）が入っていることを確認する。C++版でも同様に確認する。

> 課題4: サーバ（`add_server`）の中で `time.sleep(3)`（C++は `std::this_thread::sleep_for`）を入れて、クライアントの呼び出しが3秒ブロックされることを確認する。その間、サーバの他のコールバック（タイマー等）はどうなるか考える（現状は1スレッドの `spin` なので止まる）。
>
> 課題5（発展）: `Trigger` のクライアントを自分で書き、`counter_node` をリセットするだけのノードを作る（AddTwoIntsのクライアントとの違いは、要求に中身がない点だけ）。

## 7. トピックとサービスの使い分け

| 観点 | トピック | サービス |
|---|---|---|
| 通信の形 | 一方向・継続的 | 要求→応答（1回） |
| 相手を待つか | 待たない | 応答を待つ（非同期でも結果を待つ） |
| 向く用途 | センサ値、指令の連続送信 | 状態のリセット、設定の取得、一度だけの計算 |
| フェーズ5での使いどころ | 速度、制御指令 | シミュレーションのリセット、制御の有効/無効 |

長くかかる処理で、途中経過を知りたい場合は、サービスではなく**アクション**（フェーズ3-5）を使う。

## 8. 記録用の表

| 観点 | Python | C++ |
|---|---|---|
| サーバのコールバックの書き方（応答の返し方） | | |
| クライアントの待ち方 | | |
| `wait_for_service` の書き方 | | |
| 型名（`Request`/`Response`）の扱い | | |
| つまずいた点 | | |

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

フェーズ3-5（`docs/phase3_5_actions.md`）で、途中経過を返せる「アクション」を扱う。

## 11. 公式ドキュメント・参考資料

確認状況（2026-09-20）: 下記は `docs/idea_origin.md` に掲載済みのURLで、今回は再確認していない（docs.ros.orgは本文取得がボット対策で拒否される）。

### 公式（ROS 2 Jazzy）

- [Writing a simple service and client (Python) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Service-And-Client.html)
- [Writing a simple service and client (C++) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.html)
- [ros2/common_interfaces（GitHub）](https://github.com/ros2/common_interfaces)（`std_srvs` の定義）

> 公式ドキュメントと食い違う場合は、公式を優先する。
