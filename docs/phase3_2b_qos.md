# フェーズ3-2b 手順書: QoSの相性を体験する

`docs/learning_plan.md` フェーズ3（idea_origin.md ステップ1のQoS補足）に対応する。

- 想定環境: WSL2 + Ubuntu 24.04 + ROS2 Jazzy
- 前提: フェーズ3-1完了（`talker` 等が動く）。フェーズ3-2a（Twistでturtlesimを動かす）とは独立したテーマで、3-2aで作ったものは使わない
- 所要目安: 半コマ
- 言語: **Python**（C++版は任意）
- OSS: なし

> **このフェーズの位置づけ**: 「QoSという設定があり、合わないと**エラーも出ずに黙ってつながらない**ことがある」と知るのが目的で、概要を掴む程度でよい（フェーズ5では、全ノードが既定のQoSのまま通信する）。2-2節の7通りの組み合わせのうち、①（talker・listenerとも `reliable`。つながる）と④（talkerが `best_effort`、listenerが `reliable`。つながらない）の2つを試せば十分で、残りは任意。C++版（4節）も任意とする。
>
> **進め方**: 3-1と同じく、2節の仕様は「何を作るか」の定義で、APIの使い方までは書いていない。3節冒頭の「主なAPI」表とサンプルコード・解説を読んで理解し、QoSのパラメータを変えて動かしながら体で覚える。サンプルはこの手順書の作成時にビルド確認済みで、ノードの実行結果は未確認（出力が違う場合は、実機の表示を優先する）。

> **実行環境が無くても読めるように**: 実行する手順の直後には「期待する結果」として、表示される内容や画面の様子とその読み方を載せている。コードとROS2の仕様から筆者が想定したもので、実機では時刻などの細部が異なる。

## 0. 学習目標と完了条件

必須:

1. QoSが合わないと、エラーにならずに黙ってつながらないことがある、と説明できる（5-1節で、2-2節の①と④を試す）。

任意（発展。余力があれば）:

2. 2-2節の7通りの組み合わせをすべて試し、`ros2 topic info -v` でQoSを読み取れる。
3. C++版を書き、QoSの扱いが言語によらないことを確かめる。

## 1. 全体像

同じトピック `qos_test` を、送り手 `qos_talker` と受け手 `qos_listener` でやり取りする。どちらもQoSを起動時のパラメータで切り替えられるようにしておき、組み合わせを変えて「つながる / つながらない」を観察する。

QoSの相性は、購読側の「要求」を、配信側の「提供」が満たせるかで決まる:

![QoSの互換性。reliable/best_effort/transient_local/volatileのつながる組合せとつながらない組合せ](img/phase3_2_qos.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart LR
    subgraph OK["つながる"]
        A1["Pub: reliable"] --> B1["Sub: reliable"]
        A2["Pub: reliable"] --> B2["Sub: best_effort"]
        A3["Pub: transient_local"] --> B3["Sub: volatile"]
    end
    subgraph NG["つながらない"]
        A4["Pub: best_effort"] -. 非互換 .- B4["Sub: reliable"]
        A5["Pub: volatile"] -. 非互換 .- B5["Sub: transient_local"]
    end
```

</details>

## 2. 仕様

### 2-1. `qos_talker` / `qos_listener`

| ノード | 役割 | トピック（型） | 動作 |
|---|---|---|---|
| `qos_talker` | Publisher | `qos_test`（`std_msgs/msg/String`） | 1秒ごとに `msg 0`, `msg 1`, ... を送る。QoSはROSパラメータ `reliability`（`reliable` / `best_effort`）と `durability`（`volatile` / `transient_local`）で決める。既定は `reliable` と `volatile`。深さは10 |
| `qos_listener` | Subscriber | `qos_test`（`std_msgs/msg/String`） | 受信した文字列をログに出す。QoSはtalkerと同じパラメータで決める |

パラメータの与え方（コマンドライン）:

```bash
ros2 run learn_py qos_talker --ros-args -p reliability:=best_effort -p durability:=volatile
```

（パラメータの詳しい扱いはフェーズ3-3で学ぶ。ここでは「起動時に値を渡せる」ことだけ使う。）

### 2-2. 試す組み合わせ（①〜⑦）

5節の実験で試す組み合わせに、番号を振っておく。以降の解説と実験では、この番号で呼ぶ。

**reliability の組み合わせ**（durabilityは両方とも既定の `volatile`）

| 番号 | qos_talker | qos_listener | 期待 |
|---|---|---|---|
| ① | `reliable` | `reliable` | つながる |
| ② | `reliable` | `best_effort` | つながる |
| ③ | `best_effort` | `best_effort` | つながる |
| ④ | `best_effort` | `reliable` | **つながらない**（受信側に非互換の警告が出る） |

**durability の組み合わせ**（reliabilityは両方とも既定の `reliable`）

| 番号 | qos_talker | qos_listener | 期待 |
|---|---|---|---|
| ⑤ | `transient_local` | `transient_local` | つながり、**後から起動した**listenerにも過去のメッセージ（深さ10まで）が届く |
| ⑥ | `transient_local` | `volatile` | つながる（過去分は届かない） |
| ⑦ | `volatile` | `transient_local` | **つながらない** |

必須は①と④の2つ。残りは任意。

## 3. Python版（`ws/src/learn_py`）

`ws/src/learn_py/learn_py/` に `qos_talker.py`, `qos_listener.py` を作る。使うメッセージ型は `std_msgs` のもので、依存はフェーズ3-1で `package.xml` に足してあるので、新たに足す依存は無い。

主なAPI（rclpy）:

| やりたいこと | API |
|---|---|
| QoSの設定 | `from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy`、`QoSProfile(depth=10, reliability=..., durability=...)` |
| パラメータの宣言と取得 | `self.declare_parameter('名前', 既定値)`、`self.get_parameter('名前').get_parameter_value().string_value` |

### サンプルコードと解説

ファイル: `ws/src/learn_py/learn_py/qos_talker.py`

<!-- file: ws/src/learn_py/learn_py/qos_talker.py -->
```python
import rclpy
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node
from rclpy.qos import DurabilityPolicy, QoSProfile, ReliabilityPolicy
from std_msgs.msg import String


# パラメータの文字列（reliability / durability）から QoSProfile を組み立てる。
# 想定外の値なら ValueError で止める。
def make_qos(reliability, durability):
    if reliability not in ('reliable', 'best_effort'):
        raise ValueError(f'invalid reliability: {reliability}')
    if durability not in ('volatile', 'transient_local'):
        raise ValueError(f'invalid durability: {durability}')
    return QoSProfile(
        depth=10,
        reliability=(ReliabilityPolicy.RELIABLE if reliability == 'reliable'
                     else ReliabilityPolicy.BEST_EFFORT),
        durability=(DurabilityPolicy.TRANSIENT_LOCAL if durability == 'transient_local'
                    else DurabilityPolicy.VOLATILE),
    )


# パラメータで決めた QoS で、1秒ごとに "msg N" を qos_test へ送るノード。
class QosTalker(Node):
    # パラメータを宣言・取得し、その QoS で Publisher を作る。
    def __init__(self):
        super().__init__('qos_talker')
        self.declare_parameter('reliability', 'reliable')
        self.declare_parameter('durability', 'volatile')
        reliability = self.get_parameter('reliability').get_parameter_value().string_value
        durability = self.get_parameter('durability').get_parameter_value().string_value
        self.get_logger().info(f'QoS: reliability={reliability}, durability={durability}')
        self.pub = self.create_publisher(String, 'qos_test', make_qos(reliability, durability))
        self.count = 0
        self.timer = self.create_timer(1.0, self.on_timer)

    # 1通送ってログに出す（相手とつながっていなくてもログは出る）。
    def on_timer(self):
        msg = String()
        msg.data = f'msg {self.count}'
        self.pub.publish(msg)
        self.get_logger().info(f'publish: {msg.data}')
        self.count += 1


# エントリポイント（setup.py の entry_points から呼ばれる）。
# ノードを作って spin で回し、Ctrl+C で後片付けして終わる。
def main(args=None):
    rclpy.init(args=args)
    node = QosTalker()
    try:
        rclpy.spin(node)
    except (KeyboardInterrupt, ExternalShutdownException):
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

**`qos_talker.py` の解説**

役割は「QoSを起動時のパラメータで切り替えられる Publisher」。1秒ごとに `msg 0`, `msg 1`, ... と送る。QoSの相性実験（5節）の送り手になる。

- `make_qos(reliability, durability)`: 文字列2つから `QoSProfile` を作る関数。想定外の文字列は `ValueError` にして、綴りミスに気づけるようにしている（黙って既定値に落とすと、実験結果が何の設定だったか分からなくなる）。
- `QoSProfile(depth=10, reliability=..., durability=...)` の各項目:
  - `depth`: 履歴（history）の深さ。直近何件まで保持するか。ここでは「直近10件を保持する」設定（keep last）になる。
  - `reliability`: `RELIABLE` は届くまで再送を試みる。`BEST_EFFORT` は再送せず、取りこぼしを許す（その代わり軽い）。
  - `durability`: `VOLATILE` は「送った時点でつながっている相手にだけ届く」。`TRANSIENT_LOCAL` は「Publisherが直近の `depth` 件を覚えておき、あとから接続した購読側にも渡す」。
- `declare_parameter('reliability', 'reliable')`: パラメータを名前と既定値つきで宣言する。宣言していないパラメータは `-p` で渡しても受け付けられない。既定値の型（ここでは文字列）が、そのパラメータの型になる。
- `get_parameter(...).get_parameter_value().string_value`: 値は `ParameterValue` という入れ物で返るので、型に合ったフィールド（文字列なら `string_value`）を取り出す。整数なら `integer_value`、実数なら `double_value`。
- 起動時に `QoS: reliability=..., durability=...` をログに出しているのは、「今どの設定で動いているか」をターミナルで確認するため。実験で設定を取り違えないための工夫。
- `on_timer`: `f'msg {self.count}'` で連番の文字列を作り、`publish` してログに出す。ログの `publish:` は「送ろうとした」印であり、受け取り側がいるかどうかは関係なく出る。**つながらない組み合わせでも、talker側は普通にログを出し続ける**（相手が受け取れないだけで、送り手のエラーにはならない）。

観察ポイント（2-2節の番号で書く。実験はビルドの後に5節でまとめて行うので、ここでは何を見るかだけ押さえておけばよい）:

- ⑤（talker・listenerとも `durability` が `transient_local`）では、talkerを先に起動して5秒ほど待ってからlistenerを起動する。listenerが起動した直後に `msg 0` から数件が一気に届けば成功。`depth=10` なので、10件を超えて溜まっても、届くのは直近10件まで。
- ④（talkerが `best_effort`、listenerが `reliable`）と⑦（talkerが `volatile`、listenerが `transient_local`）のようにつながらない組み合わせでは、QoSが非互換だという警告がログに出ることが多い（出方はノードや言語で異なりうるので、実験で確認する）。`ros2 topic info /qos_test -v` で、PublisherとSubscriptionのQoSを見比べる。

つまずき: ノードのコンストラクタで例外が出ると、`spin` に入る前にプロセスごと落ちる。`ValueError: invalid reliability` は、パラメータの綴りか大文字小文字を疑う。

ファイル: `ws/src/learn_py/learn_py/qos_listener.py`

<!-- file: ws/src/learn_py/learn_py/qos_listener.py -->
```python
import rclpy
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node
from rclpy.qos import DurabilityPolicy, QoSProfile, ReliabilityPolicy
from std_msgs.msg import String


# パラメータの文字列から QoSProfile を組み立てる（qos_talker.py と同じ関数）。
def make_qos(reliability, durability):
    if reliability not in ('reliable', 'best_effort'):
        raise ValueError(f'invalid reliability: {reliability}')
    if durability not in ('volatile', 'transient_local'):
        raise ValueError(f'invalid durability: {durability}')
    return QoSProfile(
        depth=10,
        reliability=(ReliabilityPolicy.RELIABLE if reliability == 'reliable'
                     else ReliabilityPolicy.BEST_EFFORT),
        durability=(DurabilityPolicy.TRANSIENT_LOCAL if durability == 'transient_local'
                    else DurabilityPolicy.VOLATILE),
    )


# パラメータで決めた QoS で qos_test を購読し、届いた文字列をログに出すノード。
class QosListener(Node):
    # パラメータを宣言・取得し、その QoS で購読を作る。
    def __init__(self):
        super().__init__('qos_listener')
        self.declare_parameter('reliability', 'reliable')
        self.declare_parameter('durability', 'volatile')
        reliability = self.get_parameter('reliability').get_parameter_value().string_value
        durability = self.get_parameter('durability').get_parameter_value().string_value
        self.get_logger().info(f'QoS: reliability={reliability}, durability={durability}')
        self.sub = self.create_subscription(
            String, 'qos_test', self.on_message, make_qos(reliability, durability))

    # メッセージが1件届くたびに呼ばれ、ログに出す。
    def on_message(self, msg):
        self.get_logger().info(f'received: {msg.data}')


# エントリポイント（setup.py の entry_points から呼ばれる）。
# ノードを作って spin で回し、Ctrl+C で後片付けして終わる。
def main(args=None):
    rclpy.init(args=args)
    node = QosListener()
    try:
        rclpy.spin(node)
    except (KeyboardInterrupt, ExternalShutdownException):
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

**`qos_listener.py` の解説**

役割は「同じパラメータでQoSを決める Subscriber」。受け取った文字列をログに出す。`make_qos` と `declare_parameter` まわりは talker と同じなので、差分だけ書く。

- `create_subscription(String, 'qos_test', self.on_message, make_qos(...))`: 引数の順は「型・トピック名・**コールバック**・QoS」。Publisherの `create_publisher(型, 名前, QoS)` とは違い、コールバックがQoSより前に来る。順番を逆にすると、型エラーになる。
- `on_message(self, msg)`: メッセージが届くたびに呼ばれる。`msg` は `String` 型で、中身は `msg.data`。コールバックは `spin` の中で呼ばれるので、`spin` に入っていないと何も起きない。
- `self.sub` に保持しているのは、後から参照できるようにするため（Pythonではノードも内部で保持するので、必須ではない）。
- `make_qos` を talker と listener に同じ内容で二重に書いているのは、1ファイルで読み切れるようにするための割り切り。実務なら共通モジュールに切り出す。

観察ポイント: listenerを起動したときのログ `QoS: reliability=..., durability=...` を、talker側の値と見比べる。「つながる / つながらない」は、この2つの組（購読側の要求と配信側の提供）で決まる。つながらないとき、`received:` は1件も出ない。

`setup.py` の `entry_points` に2行を足し（前節までの行は残す）、再ビルドする。

<!-- snippet: py_entry_points_qos -->
```python
            'qos_talker = learn_py.qos_talker:main',
            'qos_listener = learn_py.qos_listener:main',
```

`'実行ファイル名 = パッケージ.モジュール:関数'` の形式で、`ros2 run learn_py qos_talker` の `qos_talker` が左辺、呼ばれる関数が右辺の `main`。`entry_points` を変えたときは `--symlink-install` でも再ビルドが必要。カンマの付け忘れや、リストの外へ書いてしまうミスに注意する。

```bash
cd ~/work/ros2MinimalPhysicalAi/ws
colcon build --symlink-install --packages-select learn_py
source install/setup.bash
```

期待する結果: フェーズ3-1と同じく、`Finished <<< learn_py` と `Summary: 1 package finished` が出れば成功。`ros2 pkg executables learn_py` を実行すると、今回足した `learn_py qos_listener`・`learn_py qos_talker` の2行が、既存の実行ファイルと一緒に並ぶ。

## 4. C++版（`ws/src/learn_cpp`）

> **このフェーズのC++版は任意（発展）**。フェーズ5の車両シミュレーションはPythonで実装すると決めているため、ここでC++版を作らなくても先へ進める。Python版との違いは、下の各ファイルの解説（特に `qos_talker.cpp` の解説にある対応表）を読めば概要が掴める。

`ws/src/learn_cpp/src/` に `qos_talker.cpp`, `qos_listener.cpp` を作る。`std_msgs` の依存はフェーズ3-1で足してあるので、`package.xml` と `find_package` の追加は要らない。

主なAPI（rclcpp）:

| やりたいこと | API |
|---|---|
| QoSの設定 | `rclcpp::QoS qos(10); qos.reliable(); qos.best_effort(); qos.transient_local(); qos.durability_volatile();` |
| パラメータの宣言と取得 | `declare_parameter<std::string>("名前", "既定値")`（宣言と同時に値が返る） |

### サンプルコードと解説

ファイル: `ws/src/learn_cpp/src/qos_talker.cpp`

<!-- file: ws/src/learn_cpp/src/qos_talker.cpp -->
```cpp
#include <chrono>
#include <memory>
#include <stdexcept>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

// パラメータの文字列から QoS を組み立てる。想定外の値なら例外を投げて止める。
static rclcpp::QoS make_qos(const std::string & reliability, const std::string & durability)
{
  if (reliability != "reliable" && reliability != "best_effort") {
    throw std::invalid_argument("invalid reliability: " + reliability);
  }
  if (durability != "volatile" && durability != "transient_local") {
    throw std::invalid_argument("invalid durability: " + durability);
  }
  rclcpp::QoS qos(10);
  if (reliability == "best_effort") {
    qos.best_effort();
  } else {
    qos.reliable();
  }
  if (durability == "transient_local") {
    qos.transient_local();
  } else {
    qos.durability_volatile();
  }
  return qos;
}

// パラメータで決めた QoS で、1秒ごとに "msg N" を qos_test へ送るノード。
class QosTalker : public rclcpp::Node
{
public:
  // コンストラクタ: パラメータを宣言・取得し、その QoS で Publisher とタイマーを作る。
  QosTalker() : Node("qos_talker")
  {
    const auto reliability = declare_parameter<std::string>("reliability", "reliable");
    const auto durability = declare_parameter<std::string>("durability", "volatile");
    RCLCPP_INFO(
      get_logger(), "QoS: reliability=%s, durability=%s",
      reliability.c_str(), durability.c_str());
    pub_ = create_publisher<std_msgs::msg::String>("qos_test", make_qos(reliability, durability));
    timer_ = create_wall_timer(1s, [this]() { on_timer(); });
  }

private:
  // 1通送ってログに出す（相手とつながっていなくてもログは出る）。
  void on_timer()
  {
    std_msgs::msg::String msg;
    msg.data = "msg " + std::to_string(count_++);
    pub_->publish(msg);
    RCLCPP_INFO(get_logger(), "publish: %s", msg.data.c_str());
  }

  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
  int count_ = 0;
};

// エントリポイント。ノードを作って spin で回し、Ctrl+C で spin を抜けて終わる。
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<QosTalker>());
  rclcpp::shutdown();
  return 0;
}
```

**`qos_talker.cpp` の解説**

Python版 `qos_talker.py` と同じ仕様（QoSをパラメータで切り替える Publisher）。QoSの各項目（depth・reliability・durability）の意味は Python版の解説を参照し、ここでは対応関係と言語固有の点を書く。

- `make_qos`: Pythonでは `QoSProfile(...)` に引数で渡していたものを、C++では `rclcpp::QoS qos(10);`（深さ10）を作ってから、メソッドで設定を上書きしていく。対応は次のとおり。

| Python | C++ |
|---|---|
| `ReliabilityPolicy.RELIABLE` | `qos.reliable()` |
| `ReliabilityPolicy.BEST_EFFORT` | `qos.best_effort()` |
| `DurabilityPolicy.TRANSIENT_LOCAL` | `qos.transient_local()` |
| `DurabilityPolicy.VOLATILE` | `qos.durability_volatile()` |

  最後だけ名前が `durability_volatile()` なのは、`volatile` がC++の予約語だから。
- `static`: この関数をそのファイル内だけで使う指定（他のファイルとの名前衝突を避ける）。引数の `const std::string &` は「コピーせず参照で受け取り、書き換えない」の意味。
- `throw std::invalid_argument(...)`: Pythonの `ValueError` に相当。コンストラクタで例外が出るとノードは作られず、プロセスが例外で終了する。
- `declare_parameter<std::string>("reliability", "reliable")`: 宣言と同時に**値が返る**（Pythonでは `declare_parameter` のあとに `get_parameter` が別途必要だった）。`const auto` で受けると `std::string` になる。テンプレート引数の型を付けないと、既定値の型から推論される場面もあるが、文字列は `<std::string>` と明示しておくほうが安全。
- `RCLCPP_INFO(get_logger(), "...%s...", reliability.c_str())`: ログ用のマクロで、書式はprintf形式。`%s` には `std::string` そのものではなく `.c_str()`（C形式の文字列）を渡す。`std::string` を直接渡すと、実行時に不正な表示になったり落ちたりする。
- `create_publisher<std_msgs::msg::String>("qos_test", make_qos(...))`: QoSの引数には、整数の代わりに `rclcpp::QoS` を渡せる。
- `create_wall_timer(1s, [this]() { on_timer(); })`: 1秒周期のタイマー。`1s` は `std::chrono_literals` の時間リテラルで、ラムダの `[this]` はメンバ関数を呼ぶためにオブジェクト自身を取り込む指定（フェーズ3-1の `talker.cpp` と同じ書き方）。
- `"msg " + std::to_string(count_++)`: `count_++` は現在の値を使ってから1増やす。`"msg "` は `const char *` だが、右辺が `std::string` なので連結できる。`std::to_string` を通さず `"msg " + count_` と書くと、意図しないポインタ演算になる。
- `count_` の初期化は `int count_ = 0;`。Pythonの `self.count = 0` にあたる。

観察ポイントと落とし穴は Python版と同じ。Python版とC++版でQoSの扱いは変わらないので、5-3節の課題3（Python版とC++版を組み合わせ、④・⑦が同じ結果になるか確かめる）で確認する。

ファイル: `ws/src/learn_cpp/src/qos_listener.cpp`

<!-- file: ws/src/learn_cpp/src/qos_listener.cpp -->
```cpp
#include <memory>
#include <stdexcept>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

// パラメータの文字列から QoS を組み立てる（qos_talker.cpp と同じ関数）。
static rclcpp::QoS make_qos(const std::string & reliability, const std::string & durability)
{
  if (reliability != "reliable" && reliability != "best_effort") {
    throw std::invalid_argument("invalid reliability: " + reliability);
  }
  if (durability != "volatile" && durability != "transient_local") {
    throw std::invalid_argument("invalid durability: " + durability);
  }
  rclcpp::QoS qos(10);
  if (reliability == "best_effort") {
    qos.best_effort();
  } else {
    qos.reliable();
  }
  if (durability == "transient_local") {
    qos.transient_local();
  } else {
    qos.durability_volatile();
  }
  return qos;
}

// パラメータで決めた QoS で qos_test を購読し、届いた文字列をログに出すノード。
class QosListener : public rclcpp::Node
{
public:
  // コンストラクタ: パラメータを宣言・取得し、その QoS で購読を作る。
  QosListener() : Node("qos_listener")
  {
    const auto reliability = declare_parameter<std::string>("reliability", "reliable");
    const auto durability = declare_parameter<std::string>("durability", "volatile");
    RCLCPP_INFO(
      get_logger(), "QoS: reliability=%s, durability=%s",
      reliability.c_str(), durability.c_str());
    sub_ = create_subscription<std_msgs::msg::String>(
      "qos_test", make_qos(reliability, durability),
      [this](const std_msgs::msg::String & msg) {
        RCLCPP_INFO(get_logger(), "received: %s", msg.data.c_str());
      });
  }

private:
  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
};

// エントリポイント。ノードを作って spin で回し、Ctrl+C で spin を抜けて終わる。
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<QosListener>());
  rclcpp::shutdown();
  return 0;
}
```

**`qos_listener.cpp` の解説**

Python版 `qos_listener.py` と同じ Subscriber。`make_qos` とパラメータ取得は talker と同内容なので、差分だけ書く。

- `create_subscription<std_msgs::msg::String>("qos_test", make_qos(...), コールバック)`: 引数の順は「トピック名・QoS・コールバック」。**Python版はコールバックが先でQoSが後**なので、言語を行き来するときに取り違えやすい。
- コールバックは `[this](const std_msgs::msg::String & msg) {...}` というラムダ。引数は「メッセージへのconst参照」で、コピーが起きない。本文中で `get_logger()` を呼ぶために `[this]` の取り込みが要る。`msg.data` は `std::string` なので、ログには `.c_str()` を付ける。
- `sub_` を `SharedPtr` のメンバとして保持するのは、Publisherと同じ理由（消えると購読が解除される）。
- `#include <memory>` は `SharedPtr` まわりのために置いてある。

観察ポイント: 2-2節のつながらない組み合わせ（④: talkerが `best_effort`・listenerが `reliable`、⑦: talkerが `volatile`・listenerが `transient_local`）では `received:` が出ず、警告だけが出る。警告の末尾には、どのQoSポリシーが非互換かが示される（文言の例は5-1節の「期待する結果」を参照）。

`CMakeLists.txt` に追記し、`install(TARGETS ...)` へ名前を足す（前節までの分は残す）。

<!-- snippet: cmake_qos -->
```cmake
add_executable(qos_talker src/qos_talker.cpp)
ament_target_dependencies(qos_talker rclcpp std_msgs)

add_executable(qos_listener src/qos_listener.cpp)
ament_target_dependencies(qos_listener rclcpp std_msgs)

install(TARGETS
  hello
  talker
  listener
  sine_pub
  sine_sub
  turtle_circle
  qos_talker
  qos_listener
  DESTINATION lib/${PROJECT_NAME})
```

- `add_executable(実行ファイル名 ソース)`: ソースから実行ファイルをビルドする指定。Pythonの `entry_points` にあたる。
- `ament_target_dependencies(ターゲット 依存...)`: そのターゲットが使うパッケージ（ヘッダ・ライブラリ）を、ターゲットごとに列挙する。`qos_talker` と `qos_listener` は `String` を使うので `std_msgs`、それと `rclcpp` が必要。
- `install(TARGETS ... DESTINATION lib/${PROJECT_NAME})`: ビルドした実行ファイルを `install/learn_cpp/lib/learn_cpp/` へ置く。`ros2 run learn_cpp ...` はこの場所を探すので、ここに名前がないと `No executable found` になる。既存の名前は消さずに残し、新しい2つを足す。
- `install(TARGETS ...)` の `turtle_circle` は、フェーズ3-2aでC++版の `turtle_circle` を作った場合だけ書く。作っていないのに名前を書くと、存在しないターゲットを指定したことになり、CMakeの段階でビルドがエラーになる。

```bash
cd ~/work/ros2MinimalPhysicalAi/ws
colcon build --symlink-install --packages-select learn_cpp
source install/setup.bash
```

期待する結果: `Finished <<< learn_cpp` と `Summary: 1 package finished` が出れば成功。

## 5. 実験: QoSの相性を確かめる

以下、`ros2 run learn_py qos_talker` / `qos_listener`（C++版でも同様）で、2-2節の表のとおりにパラメータを変えて起動する。ターミナルを2つ使う（T1にtalker、T2にlistener）。概要を掴むだけなら、5-1節の①と④の2つを試せば十分（0節の完了条件）。

### 5-1. reliability（①〜④）

まず①（両方とも既定の `reliable`）で、つながる様子を見る。

```bash
# T1
ros2 run learn_py qos_talker
# T2
ros2 run learn_py qos_listener
```

期待する結果（抜粋）:

```text
# T1（qos_talker）
[INFO] [1790242000.100000000] [qos_talker]: QoS: reliability=reliable, durability=volatile
[INFO] [1790242001.101234567] [qos_talker]: publish: msg 0
[INFO] [1790242002.101198765] [qos_talker]: publish: msg 1

# T2（qos_listener）
[INFO] [1790242000.900000000] [qos_listener]: QoS: reliability=reliable, durability=volatile
[INFO] [1790242001.101987654] [qos_listener]: received: msg 0
[INFO] [1790242002.101954321] [qos_listener]: received: msg 1
```

起動直後の1行目は、パラメータから読み取ったQoSの設定。ここで意図した組み合わせになっているかを、まず確かめる。T2に `received:` が1秒ごとに出れば、つながっている。時刻（`[...]` の数字）は実行ごとに変わる。

次に、両方を `Ctrl+C` で止めてから④（talkerが `best_effort`、listenerが `reliable`）を試す。

```bash
# T1
ros2 run learn_py qos_talker --ros-args -p reliability:=best_effort
# T2
ros2 run learn_py qos_listener --ros-args -p reliability:=reliable
```

期待する結果: talkerは `publish: msg N` を出し続けるが、listenerには `received:` が1行も出ない。その代わり、両方のターミナルに警告が出る（Python版の場合。C++版では末尾の方針名が `RELIABILITY_QOS_POLICY` になる）。

```text
# T2（qos_listener、reliable）
[WARN] [1790242010.200000000] [qos_listener]: New publisher discovered on topic '/qos_test', offering incompatible QoS. No messages will be received from it. Last incompatible policy: RELIABILITY

# T1（qos_talker、best_effort）
[WARN] [1790242010.200000000] [qos_talker]: New subscription discovered on topic '/qos_test', requesting incompatible QoS. No messages will be sent to it. Last incompatible policy: RELIABILITY
```

警告は「相手を見つけたが、QoSが合わないのでメッセージをやり取りしない」という意味で、最後の `Last incompatible policy` が食い違っている項目を示す。エラーで止まるわけではないので、ログを見落とすと「何も起きない」ように見える。

④のまま、3つ目のターミナルでQoSを確かめる。

```bash
ros2 topic info /qos_test -v
```

期待する結果: Publisher・SubscriptionそれぞれのQoS（`Reliability`、`Durability`）が表示される（抜粋）。

```text
Type: std_msgs/msg/String

Publisher count: 1

Node name: qos_talker
...
Endpoint type: PUBLISHER
QoS profile:
  Reliability: BEST_EFFORT
  History (Depth): KEEP_LAST (10)
  Durability: VOLATILE
  ...

Subscription count: 1

Node name: qos_listener
...
Endpoint type: SUBSCRIPTION
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (10)
  Durability: VOLATILE
  ...
```

`Publisher count` と `Subscription count` はどちらも1で、ROS2から見ると「両方いる」。それでもつながらない理由は、`Reliability` の行の食い違い（Publisherが `BEST_EFFORT`、Subscriptionが `RELIABLE`）にある。**つながらない場合は、この2つを見比べて、どちらが食い違っているかを読み取る**。

②・③（任意）も同じ要領で、`-p reliability:=...` の値を2-2節の表のとおりに変えて起動する。どちらもつながる。

### 5-2. durability（⑤〜⑦。任意）

⑤（両方とも `transient_local`）は、talkerを先に起動して5秒ほど待ってから、listenerを起動する。

```bash
# T1（先に起動し、5秒ほど待つ）
ros2 run learn_py qos_talker --ros-args -p durability:=transient_local
# T2
ros2 run learn_py qos_listener --ros-args -p durability:=transient_local
```

期待する結果: listenerの起動直後に、それまでの分がまとめて届く。

```text
# T2（qos_listener、⑤）
[INFO] [...] [qos_listener]: QoS: reliability=reliable, durability=transient_local
[INFO] [...] [qos_listener]: received: msg 0
[INFO] [...] [qos_listener]: received: msg 1
[INFO] [...] [qos_listener]: received: msg 2
[INFO] [...] [qos_listener]: received: msg 3
[INFO] [...] [qos_listener]: received: msg 4
[INFO] [...] [qos_listener]: received: msg 5     ← ここからは1秒ごと
```

最初の数行は、時刻がほぼ同じ（一気に届いた）になる。talkerを10秒以上待ってから起動した場合は、深さ10を超えた古い分は捨てられているので、直近10件だけが届く。

⑥（talkerが `transient_local`、listenerが `volatile`）では、同じ手順でも過去分は届かず、listenerの起動後に送られた番号から始まる。

⑦（talkerが `volatile`、listenerが `transient_local`）はつながらず、5-1節の④と同じ形の警告が出る。違いは末尾の `Last incompatible policy` が `DURABILITY` になる点。`ros2 topic info /qos_test -v` では、`Durability` の行が食い違っている。

### 5-3. 課題

> 課題1（任意）: 2-2節の表の①〜⑦をすべて試し、期待どおりの結果になるか確認する。つながらない組み合わせ（④・⑦）では、受信側のログに非互換のポリシー名を含む警告が出ることを確認する。
>
> 課題2（任意）: つながらない組み合わせのまま、`ros2 topic info /qos_test -v` で、PublisherとSubscriptionのどの行が食い違っているかを読み取る。
>
> 課題3（C++版を作った場合）: Python版のtalker × C++版のlistener（およびその逆）でも、④と⑦が同じ結果になることを確認する。QoSの扱いが言語に依存しないことを確認する。

## 6. QoS互換性のまとめ

5節で確認した組み合わせ（番号は2-2節の表）を、判定ルールとして整理する。

- **reliability**: `best_effort`側のPublisherと`reliable`側のSubscriberの組み合わせ（④）だけがつながらない。Subscriberが要求する品質（`reliable`）を、Publisherが提供できない（`best_effort`）ため。それ以外の3通り（①②③）はつながる。
- **durability**: `volatile`側のPublisherと`transient_local`側のSubscriber（⑦）だけがつながらない。理由はreliabilityと同じ構造（Subscriberが要求する`transient_local`をPublisherが提供できない）。
- 共通の考え方: QoSの互換性は「Subscriberが要求する最低品質 ≦ Publisherが提供する品質」で決まる（「要求 vs 提供」モデル）。**Publisher側の設定を緩めるとつながらなくなる**ことがある、と覚えておくと判断しやすい。
- この互換性判定は、Python版・C++版のどちらの組み合わせでも同じ結果になる（QoSはROS2共通の仕組みで、言語には依存しない。5-3節の課題3で確かめられる）。

## 7. 実務への手がかり

センサデータは「多少欠けても最新が大事」なので、`best_effort` にすることが多い。ROS2には、そのための定義済み設定がある。

- Python: `from rclpy.qos import qos_profile_sensor_data`
- C++: `rclcpp::SensorDataQoS()`

フェーズ5の車両シミュレーションで、速度（センサ相当）と制御指令のトピックにどのQoSを使うか、考える材料になる。

## 8. つまずきやすい点

| 症状 | 確認すること |
|---|---|
| `ValueError: invalid reliability` | パラメータの綴り。`reliable` か `best_effort`（小文字） |
| 2-2節の⑤（両方 `transient_local`）で過去分が届かない | talkerが `transient_local` か、listener側も `transient_local` か。talkerを先に起動して待ったか |
| 非互換の警告が出ない | 警告はノードのログ（ターミナル）に出る。`rqt_console` でも確認できる |
| C++版のビルドで `install TARGETS given target "turtle_circle" which does not exist` のようなエラー | `install(TARGETS ...)` に、作っていないノード（フェーズ3-2aでC++版を作らなかった場合の `turtle_circle` など）の名前を書いていないか |

## 9. 次へ

フェーズ3-3（`docs/phase3_3_parameters.md`）で、パラメータを本格的に扱う。ここで使った `-p` の指定と、YAMLでの指定を学ぶ。

## 10. 公式ドキュメント・参考資料

確認状況（2026-09-20）: 下記は `docs/idea_origin.md` に掲載済みのURLで、今回は再確認していない（docs.ros.orgは本文取得がボット対策で拒否される）。

### 公式（ROS 2 Jazzy）

- [Quality of Service settings — Jazzy](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html)（互換性の一覧表がある）
- [Class QoS — rclcpp Jazzy](https://docs.ros.org/en/jazzy/p/rclcpp/generated/classrclcpp_1_1QoS.html)

> 公式ドキュメントと食い違う場合は、公式を優先する。

> 出典: 各サンプルのAPIの使い方は、上記の公式チュートリアルを参考にした（ROS 2ドキュメントはCC BY 4.0）。ノード名・仕様・コード・文章は独自に書いたもので、逐語の転載ではない。
