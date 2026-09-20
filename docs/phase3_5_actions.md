# フェーズ3-5 手順書: アクション（途中経過つき・中断できる長い処理）

`docs/learning_plan.md` フェーズ3（idea_origin.md ステップ1の1-5）に対応する。時間のかかる処理に対し、ゴールの送信・途中経過（フィードバック）・最終結果・中断を扱う通信を、Python・C++の両方で書く。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy
- 前提: フェーズ3-1〜3-4完了
- 所要目安: 2コマ（コード量が多い）
- 言語: **Python・C++の両方**
- 使う標準インターフェース: `example_interfaces/action/Fibonacci`（WSLの `/opt/ros/jazzy/share/example_interfaces/action/Fibonacci.action` で内容を確認済み）

> **進め方**: 今までより長いので、まず**Pythonを完成**させ、動作を理解してからC++に進む。詰まったら「サンプルコード」で答え合わせをする。サンプルはこの手順書の作成時にビルド確認済みで、ノードの実行結果は未確認（出力が違えば差分を貼ってほしい）。

## 0. 学習目標と完了条件

1. アクションサーバ・クライアントを、Python・C++の両方で書ける。
2. ゴール（goal）・フィードバック（feedback）・結果（result）・中断（cancel）の流れを説明できる。
3. **サーバの処理中に中断要求を受け付ける**ために、実行の仕組み（Pythonはマルチスレッドexecutor、C++は別スレッド）が必要なことを説明できる。
4. サービスとアクションの使い分けを説明できる。

## 1. 全体像

```mermaid
sequenceDiagram
    participant C as fibonacci_client
    participant S as fibonacci_server
    C->>S: goal（order=5）を送る
    S-->>C: 受理（accepted）
    loop 1秒ごとに
        S-->>C: feedback（途中までの数列）
    end
    S-->>C: result（完成した数列）と status=SUCCEEDED
    Note over C,S: 途中でクライアントが cancel を送ると、サーバは処理を止めて CANCELED を返す
```

ゴールの状態遷移:

```mermaid
stateDiagram-v2
    [*] --> ACCEPTED: ゴール受理
    ACCEPTED --> EXECUTING: 実行開始
    EXECUTING --> SUCCEEDED: 完了
    EXECUTING --> CANCELING: 中断要求
    CANCELING --> CANCELED: 中断完了
    EXECUTING --> ABORTED: サーバが失敗と判断
    SUCCEEDED --> [*]
    CANCELED --> [*]
    ABORTED --> [*]
```

インターフェースを確認する:

```bash
ros2 interface show example_interfaces/action/Fibonacci
```

`---` で3つに分かれる: 上から**ゴール**（`order`）、**結果**（`sequence`）、**フィードバック**（`sequence`）。

サービスとの違い:

| 観点 | サービス | アクション |
|---|---|---|
| 途中経過 | なし | あり（feedback） |
| 中断 | できない | できる（cancel） |
| 向く処理 | 短い | 数秒〜数分かかる |

## 2. 仕様

| ノード | 役割 | アクション名（型） | 動作 |
|---|---|---|---|
| `fibonacci_server` | サーバ | `fibonacci`（`example_interfaces/action/Fibonacci`） | ゴール `order`（1以上）を受け、数列 `[0, 1]` から始めて、**1秒ごとに**次の項を足し、その時点の数列をフィードバックとして送る。`order - 1` 回の加算を終えたら、結果に完成した数列（`order + 1` 項）を入れて成功で終わる。`order < 1` のゴールは**拒否**する。処理中に中断要求を受けたら、処理を止めて `CANCELED` で終わる |
| `fibonacci_client` | クライアント | `fibonacci` | ROSパラメータ `order`（整数、既定5）でゴールを送り、フィードバックをログに出し、結果を受けて終了する。パラメータ `cancel_after`（実数、秒、既定 `0.0`）が正なら、ゴール受理からその秒数後に中断要求を送る。サーバがいなければ5秒待って諦める |

例: `order = 5` → 結果の数列は `[0, 1, 1, 2, 3, 5]`（6項）。

## 3. 準備: 依存の追加

**Python（`ws/src/learn_py/package.xml`）**: 次の行を足す（`example_interfaces` は3-4で追加済み）。

```xml
<depend>action_msgs</depend>
```

**C++（`ws/src/learn_cpp/package.xml`）**: 次の行を足す。

```xml
<depend>rclcpp_action</depend>
```

C++の `CMakeLists.txt` にも足す（既存の `find_package(...)` の並びへ）。

<!-- snippet: cmake_find_action -->
```cmake
find_package(rclcpp_action REQUIRED)
```

## 4. Python版（`ws/src/learn_py`）

`ws/src/learn_py/learn_py/` に `fibonacci_server.py`, `fibonacci_client.py` を作る。

主なAPI（rclpy）:

| やりたいこと | API |
|---|---|
| サーバの作成 | `ActionServer(self, 型, 'アクション名', execute_callback=..., goal_callback=..., cancel_callback=...)` |
| ゴールを受けるか決める | `goal_callback` が `GoalResponse.ACCEPT` / `REJECT` を返す |
| 中断を受けるか決める | `cancel_callback` が `CancelResponse.ACCEPT` / `REJECT` を返す |
| 実行処理 | `execute_callback(goal_handle)`。`goal_handle.request` が受けたゴール。`goal_handle.publish_feedback(...)`、`goal_handle.succeed()` / `canceled()`、最後に**結果を返す** |
| 中断の検知 | `goal_handle.is_cancel_requested` |
| クライアントの作成 | `ActionClient(self, 型, 'アクション名')` |
| ゴール送信 | `client.send_goal_async(goal, feedback_callback=...)`（Futureを返す） |
| 結果の取得 | `goal_handle.get_result_async()`（Futureを返す）。`add_done_callback` で受ける |
| 中断要求 | `goal_handle.cancel_goal_async()` |

> **重要（中断の仕組み）**: サーバの実行処理が `time.sleep` で待っている間、通常の単一スレッドのexecutorでは、他のコールバック（中断要求の受付）が動けない。そこで、`MultiThreadedExecutor` と `ReentrantCallbackGroup` を使い、実行処理と並行して中断要求を処理できるようにする。C++版では代わりに、実行処理を**別スレッド**で行う。

<details>
<summary>サンプルコード（答え合わせ用）</summary>

ファイル: `ws/src/learn_py/learn_py/fibonacci_server.py`

<!-- file: ws/src/learn_py/learn_py/fibonacci_server.py -->
```python
import time

import rclpy
from example_interfaces.action import Fibonacci
from rclpy.action import ActionServer, CancelResponse, GoalResponse
from rclpy.callback_groups import ReentrantCallbackGroup
from rclpy.executors import ExternalShutdownException, MultiThreadedExecutor
from rclpy.node import Node


class FibonacciServer(Node):
    def __init__(self):
        super().__init__('fibonacci_server')
        self.server = ActionServer(
            self, Fibonacci, 'fibonacci',
            execute_callback=self.execute,
            goal_callback=self.on_goal,
            cancel_callback=self.on_cancel,
            callback_group=ReentrantCallbackGroup())

    def on_goal(self, goal_request):
        if goal_request.order < 1:
            self.get_logger().warn(f'reject goal: order={goal_request.order}')
            return GoalResponse.REJECT
        self.get_logger().info(f'accept goal: order={goal_request.order}')
        return GoalResponse.ACCEPT

    def on_cancel(self, goal_handle):
        self.get_logger().info('cancel requested')
        return CancelResponse.ACCEPT

    def execute(self, goal_handle):
        order = goal_handle.request.order
        sequence = [0, 1]
        feedback = Fibonacci.Feedback()
        result = Fibonacci.Result()
        for i in range(1, order):
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                result.sequence = sequence
                self.get_logger().info('goal canceled')
                return result
            sequence.append(sequence[i] + sequence[i - 1])
            feedback.sequence = sequence
            goal_handle.publish_feedback(feedback)
            time.sleep(1.0)
        goal_handle.succeed()
        result.sequence = sequence
        self.get_logger().info(f'goal succeeded: {list(sequence)}')
        return result


def main(args=None):
    rclpy.init(args=args)
    node = FibonacciServer()
    executor = MultiThreadedExecutor()
    try:
        rclpy.spin(node, executor=executor)
    except (KeyboardInterrupt, ExternalShutdownException):
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

ファイル: `ws/src/learn_py/learn_py/fibonacci_client.py`

<!-- file: ws/src/learn_py/learn_py/fibonacci_client.py -->
```python
import rclpy
from action_msgs.msg import GoalStatus
from example_interfaces.action import Fibonacci
from rclpy.action import ActionClient
from rclpy.node import Node

STATUS_NAMES = {
    GoalStatus.STATUS_SUCCEEDED: 'SUCCEEDED',
    GoalStatus.STATUS_CANCELED: 'CANCELED',
    GoalStatus.STATUS_ABORTED: 'ABORTED',
}


class FibonacciClient(Node):
    def __init__(self):
        super().__init__('fibonacci_client')
        self.declare_parameter('order', 5)
        self.declare_parameter('cancel_after', 0.0)
        self.client = ActionClient(self, Fibonacci, 'fibonacci')
        self.goal_handle = None
        self.cancel_timer = None
        self.done = False

    def send_goal(self):
        if not self.client.wait_for_server(timeout_sec=5.0):
            self.get_logger().error('action server fibonacci is not available')
            self.done = True
            return
        goal = Fibonacci.Goal()
        goal.order = self.get_parameter('order').value
        future = self.client.send_goal_async(goal, feedback_callback=self.on_feedback)
        future.add_done_callback(self.on_goal_response)

    def on_goal_response(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().error('goal rejected')
            self.done = True
            return
        self.get_logger().info('goal accepted')
        self.goal_handle = goal_handle
        goal_handle.get_result_async().add_done_callback(self.on_result)
        cancel_after = self.get_parameter('cancel_after').value
        if cancel_after > 0.0:
            self.cancel_timer = self.create_timer(cancel_after, self.on_cancel_timer)

    def on_cancel_timer(self):
        self.cancel_timer.cancel()
        self.get_logger().info('send cancel request')
        self.goal_handle.cancel_goal_async()

    def on_feedback(self, feedback_msg):
        self.get_logger().info(f'feedback: {list(feedback_msg.feedback.sequence)}')

    def on_result(self, future):
        response = future.result()
        name = STATUS_NAMES.get(response.status, str(response.status))
        self.get_logger().info(f'result [{name}]: {list(response.result.sequence)}')
        self.done = True


def main(args=None):
    rclpy.init(args=args)
    node = FibonacciClient()
    try:
        node.send_goal()
        while rclpy.ok() and not node.done:
            rclpy.spin_once(node, timeout_sec=0.1)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

</details>

`setup.py` の `entry_points` に2行を足し、再ビルドする。

<!-- snippet: py_entry_points_action -->
```python
            'fibonacci_server = learn_py.fibonacci_server:main',
            'fibonacci_client = learn_py.fibonacci_client:main',
```

```bash
cd ~/work/ros2MinimalPhysicalAi/ws
colcon build --symlink-install --packages-select learn_py
source install/setup.bash
```

## 5. C++版（`ws/src/learn_cpp`）

`ws/src/learn_cpp/src/` に `fibonacci_server.cpp`, `fibonacci_client.cpp` を作る。

主なAPI（rclcpp_action。`#include "rclcpp_action/rclcpp_action.hpp"`）:

| やりたいこと | API |
|---|---|
| サーバの作成 | `rclcpp_action::create_server<型>(this, "アクション名", goal処理, cancel処理, accepted処理)` |
| ゴールの受理判断 | `GoalResponse::ACCEPT_AND_EXECUTE` / `REJECT` を返す |
| 中断の受理判断 | `CancelResponse::ACCEPT` / `REJECT` を返す |
| 受理後の実行 | accepted処理の中で**別スレッド**（`std::thread`）を起こして実行する（executorを塞がない） |
| 実行中の操作 | `goal_handle->publish_feedback(...)`、`is_canceling()`、`succeed(結果)` / `canceled(結果)` |
| クライアントの作成 | `rclcpp_action::create_client<型>(this, "アクション名")` |
| ゴール送信 | `client->async_send_goal(goal, options)`。`options` に `goal_response_callback`、`feedback_callback`、`result_callback` を設定 |
| 中断要求 | `client->async_cancel_goal(goal_handle)` |

Pythonとの違いの見どころ: Pythonは `Future` の完了コールバックをつなぐが、C++は送信時にまとめて `options` に3つのコールバックを渡す。

<details>
<summary>サンプルコード（答え合わせ用）</summary>

ファイル: `ws/src/learn_cpp/src/fibonacci_server.cpp`

<!-- file: ws/src/learn_cpp/src/fibonacci_server.cpp -->
```cpp
#include <memory>
#include <thread>

#include "example_interfaces/action/fibonacci.hpp"
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"

using Fibonacci = example_interfaces::action::Fibonacci;
using GoalHandle = rclcpp_action::ServerGoalHandle<Fibonacci>;

class FibonacciServer : public rclcpp::Node
{
public:
  FibonacciServer() : Node("fibonacci_server")
  {
    server_ = rclcpp_action::create_server<Fibonacci>(
      this, "fibonacci",
      [this](const rclcpp_action::GoalUUID &, std::shared_ptr<const Fibonacci::Goal> goal) {
        if (goal->order < 1) {
          RCLCPP_WARN(get_logger(), "reject goal: order=%d", goal->order);
          return rclcpp_action::GoalResponse::REJECT;
        }
        RCLCPP_INFO(get_logger(), "accept goal: order=%d", goal->order);
        return rclcpp_action::GoalResponse::ACCEPT_AND_EXECUTE;
      },
      [this](const std::shared_ptr<GoalHandle>) {
        RCLCPP_INFO(get_logger(), "cancel requested");
        return rclcpp_action::CancelResponse::ACCEPT;
      },
      [this](const std::shared_ptr<GoalHandle> goal_handle) {
        // executorを塞がないよう、実行は別スレッドで行う
        std::thread([this, goal_handle]() { execute(goal_handle); }).detach();
      });
  }

private:
  void execute(const std::shared_ptr<GoalHandle> goal_handle)
  {
    const auto goal = goal_handle->get_goal();
    auto feedback = std::make_shared<Fibonacci::Feedback>();
    auto result = std::make_shared<Fibonacci::Result>();
    auto & sequence = feedback->sequence;
    sequence = {0, 1};
    rclcpp::Rate rate(1);

    for (int i = 1; i < goal->order && rclcpp::ok(); ++i) {
      if (goal_handle->is_canceling()) {
        result->sequence = sequence;
        goal_handle->canceled(result);
        RCLCPP_INFO(get_logger(), "goal canceled");
        return;
      }
      sequence.push_back(sequence[i] + sequence[i - 1]);
      goal_handle->publish_feedback(feedback);
      rate.sleep();
    }

    if (rclcpp::ok()) {
      result->sequence = sequence;
      goal_handle->succeed(result);
      RCLCPP_INFO(get_logger(), "goal succeeded");
    }
  }

  rclcpp_action::Server<Fibonacci>::SharedPtr server_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<FibonacciServer>());
  rclcpp::shutdown();
  return 0;
}
```

ファイル: `ws/src/learn_cpp/src/fibonacci_client.cpp`

<!-- file: ws/src/learn_cpp/src/fibonacci_client.cpp -->
```cpp
#include <chrono>
#include <cstdint>
#include <memory>
#include <sstream>
#include <string>
#include <vector>

#include "example_interfaces/action/fibonacci.hpp"
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"

using namespace std::chrono_literals;
using Fibonacci = example_interfaces::action::Fibonacci;
using GoalHandle = rclcpp_action::ClientGoalHandle<Fibonacci>;

static std::string to_text(const std::vector<int32_t> & values)
{
  std::ostringstream out;
  out << "[";
  for (size_t i = 0; i < values.size(); ++i) {
    out << (i == 0 ? "" : ", ") << values[i];
  }
  out << "]";
  return out.str();
}

class FibonacciClient : public rclcpp::Node
{
public:
  FibonacciClient() : Node("fibonacci_client")
  {
    order_ = declare_parameter<int64_t>("order", 5);
    cancel_after_ = declare_parameter<double>("cancel_after", 0.0);
    client_ = rclcpp_action::create_client<Fibonacci>(this, "fibonacci");
  }

  void send_goal()
  {
    if (!client_->wait_for_action_server(5s)) {
      RCLCPP_ERROR(get_logger(), "action server fibonacci is not available");
      rclcpp::shutdown();
      return;
    }

    auto goal = Fibonacci::Goal();
    goal.order = static_cast<int32_t>(order_);

    rclcpp_action::Client<Fibonacci>::SendGoalOptions options;
    options.goal_response_callback = [this](const GoalHandle::SharedPtr & goal_handle) {
      if (!goal_handle) {
        RCLCPP_ERROR(get_logger(), "goal rejected");
        rclcpp::shutdown();
        return;
      }
      RCLCPP_INFO(get_logger(), "goal accepted");
      goal_handle_ = goal_handle;
      if (cancel_after_ > 0.0) {
        cancel_timer_ = create_wall_timer(
          std::chrono::duration<double>(cancel_after_), [this]() {
            cancel_timer_->cancel();
            RCLCPP_INFO(get_logger(), "send cancel request");
            client_->async_cancel_goal(goal_handle_);
          });
      }
    };
    options.feedback_callback = [this](
      GoalHandle::SharedPtr, const std::shared_ptr<const Fibonacci::Feedback> feedback) {
      RCLCPP_INFO(get_logger(), "feedback: %s", to_text(feedback->sequence).c_str());
    };
    options.result_callback = [this](const GoalHandle::WrappedResult & result) {
      const char * name = "UNKNOWN";
      switch (result.code) {
        case rclcpp_action::ResultCode::SUCCEEDED: name = "SUCCEEDED"; break;
        case rclcpp_action::ResultCode::CANCELED: name = "CANCELED"; break;
        case rclcpp_action::ResultCode::ABORTED: name = "ABORTED"; break;
        default: break;
      }
      RCLCPP_INFO(
        get_logger(), "result [%s]: %s", name, to_text(result.result->sequence).c_str());
      rclcpp::shutdown();
    };
    client_->async_send_goal(goal, options);
  }

private:
  int64_t order_;
  double cancel_after_;
  rclcpp_action::Client<Fibonacci>::SharedPtr client_;
  GoalHandle::SharedPtr goal_handle_;
  rclcpp::TimerBase::SharedPtr cancel_timer_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<FibonacciClient>();
  node->send_goal();
  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

</details>

`CMakeLists.txt` に追記し、`install(TARGETS ...)` に名前を足す（これまでの分は残す）。

<!-- snippet: cmake_action -->
```cmake
add_executable(fibonacci_server src/fibonacci_server.cpp)
ament_target_dependencies(fibonacci_server rclcpp rclcpp_action example_interfaces)

add_executable(fibonacci_client src/fibonacci_client.cpp)
ament_target_dependencies(fibonacci_client rclcpp rclcpp_action example_interfaces)

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
  fibonacci_server
  fibonacci_client
  DESTINATION lib/${PROJECT_NAME})
```

```bash
cd ~/work/ros2MinimalPhysicalAi/ws
colcon build --symlink-install --packages-select learn_cpp
source install/setup.bash
```

## 6. 実験

### 6-1. CLIから使う

```bash
# T1
ros2 run learn_py fibonacci_server
# T2
ros2 action list -t
ros2 action info /fibonacci
ros2 action send_goal /fibonacci example_interfaces/action/Fibonacci "{order: 5}" --feedback
```

フィードバックが1秒ごとに流れ、最後に結果が出る。**処理の途中で `Ctrl+C`** を押すと、中断要求が送られる（サーバ側のログに `cancel requested`）。

### 6-2. 自作クライアントから使う

```bash
# T1
ros2 run learn_py fibonacci_server
# T2
ros2 run learn_py fibonacci_client --ros-args -p order:=6
# 中断も試す（3秒後に中断要求）
ros2 run learn_py fibonacci_client --ros-args -p order:=10 -p cancel_after:=3.0
```

言語の組み合わせを試す:

| 組み合わせ | T1 | T2 |
|---|---|---|
| Python × Python | `ros2 run learn_py fibonacci_server` | `ros2 run learn_py fibonacci_client ...` |
| Python × C++ | `ros2 run learn_py fibonacci_server` | `ros2 run learn_cpp fibonacci_client ...` |
| C++ × Python | `ros2 run learn_cpp fibonacci_server` | `ros2 run learn_py fibonacci_client ...` |
| C++ × C++ | `ros2 run learn_cpp fibonacci_server` | `ros2 run learn_cpp fibonacci_client ...` |

> 課題1: `order:=0` でゴールを送り、拒否されること（クライアントとサーバ双方のログ）を確認する。
>
> 課題2: `cancel_after:=3.0` で、フィードバックが3回ほど流れた後に中断され、結果のstatusが `CANCELED` になることを確認する。
>
> 課題3: Python版サーバで、`MultiThreadedExecutor` を**外して**（`rclpy.spin(node)` にして）、`ReentrantCallbackGroup` も外し、中断要求がどうなるか確認する（処理が終わるまで中断が届かない、またはゴールが最後まで走ることを観察する）。確認後は元に戻す。
>
> 課題4: サーバが動いている間に、クライアントを2つ同時に起動して、2つのゴールが並行して処理されることを確認する（Python版は `ReentrantCallbackGroup` が効いている）。
>
> 課題5（発展）: `rqt_graph` でアクションが内部的に「サービス3つ＋トピック2つ」（`_action/send_goal`、`_action/cancel_goal`、`_action/get_result`、`_action/feedback`、`_action/status`）でできていることを、`ros2 topic list -t` / `ros2 service list -t` で見つける。

## 7. 記録用の表

| 観点 | Python | C++ |
|---|---|---|
| サーバの構成（コールバックの数と役割） | | |
| 実行処理の動かし方（executor / スレッド） | | |
| クライアントの結果の受け方（Future / options） | | |
| コード行数（サーバ+クライアント） | | |
| つまずいた点 | | |

## 8. つまずきやすい点

| 症状 | 確認すること |
|---|---|
| 中断要求が効かない（Python） | `MultiThreadedExecutor` と `ReentrantCallbackGroup` を使っているか。`time.sleep` 中は他のコールバックが動けない |
| クライアントが終わらない（Python） | `on_result` で `self.done = True` にしているか。ゴールが拒否された場合も `done` にしているか |
| C++でビルドエラー（`rclcpp_action` が見つからない） | `package.xml` の `<depend>rclcpp_action</depend>`、`find_package(rclcpp_action REQUIRED)`、`ament_target_dependencies` |
| C++の `goal_response_callback` の型エラー | Jazzyでは引数が `GoalHandle::SharedPtr`（ゴールが拒否されると空）。古い資料の `std::shared_future` 形式は使わない |
| Ctrl+Cで終了するとき、C++サーバでエラーや異常終了が出る | 実行処理を別スレッド（`detach`）で動かしているため、処理中のスレッドが残りうる。学習用としては許容する。本格的に作る場合は、スレッドを管理して終了時に待つ |
| 実行が終わってもプロセスが残る | C++クライアントは `rclcpp::shutdown()` を結果コールバックで呼んでいるか |
| `ros2 action send_goal` で型が見つからない | 型名の綴り `example_interfaces/action/Fibonacci`（`action` が入る） |

## 9. 次へ

フェーズ4（`docs/phase4_launch.md`）で、これまでのノードをlaunchファイルで束ねる。フェーズ3の総括（1-7の比較表）は、フェーズ3-1〜3-5の各「記録用の表」を材料に、ユーザーが `docs/` に「1-7の比較表」としてまとめる（完成後にレビューを依頼できる）。

## 10. 公式ドキュメント・参考資料

確認状況（2026-09-20）: 下記は `docs/idea_origin.md` に掲載済みのURLで、今回は再確認していない（docs.ros.orgは本文取得がボット対策で拒否される）。

### 公式（ROS 2 Jazzy）

- [Writing an action server and client (Python) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Py.html)
- [Writing an action server and client (C++) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Cpp.html)
- [Intermediate — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate.html)

> 公式チュートリアルは独自の `.action` 定義（`action_tutorials_interfaces`）を使うが、本手順書は標準の `example_interfaces` を使う。公式ドキュメントと食い違う場合は、公式を優先する。

> 出典: 各サンプルのAPIの使い方は、上記の公式チュートリアルを参考にした（ROS 2ドキュメントはCC BY 4.0）。ノード名・仕様・コード・文章は独自に書いたもので、逐語の転載ではない。
