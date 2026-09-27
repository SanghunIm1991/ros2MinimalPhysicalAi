# フィジカルAI開発学習のためのROS2最低限構成の構築

## 概要・きっかけ

フィジカルAI（実機ロボット・センサ・アクチュエータと連動するAI）の開発を学ぶために、ROS2（Robot Operating System 2）を使った「最低限の構成」を構築し、学習用の土台とする。アプリというより学習環境・教材寄りの企画。

**学習方針の軸**: 本企画の目的はフィジカルAIそのものを深く学ぶことではなく、開発に必要な技術スタックを一通り身に着けることにある。そのため各ステップの技術選定では、サポート終了が近い・レガシー化しつつある技術は（学習事例の豊富さ等のメリットがあっても）コストパフォーマンスが悪いと判断し避け、現行の後継技術を優先する（例: ステップ5でGazebo Classicではなく新系統のGazeboを選定）。

## 評価

- **良い点**
  - ROS2はフィジカルAI・ロボティクス分野の事実上の標準の一つで、習得の実務的価値が高い
  - 「最低限構成」から始めることで、いきなり複雑な既存パッケージに触れるより学習効率が良い
- **懸念点**
  - 実機（ロボットハードウェア）を用意するか、シミュレータ（Gazebo等）で完結させるかで必要な準備・難易度が大きく変わる
  - ROS2はLinux（Ubuntu）が第一級サポートで、Windows環境（本PC）での開発は制約が出やすい（WSL2利用が現実的）
  - 「最低限」の定義（トピック通信だけ確認するのか、センサ→判断→アクチュエータの一連の系まで含めるのか）を先に決めた方がよい

## 肉付け（想定内容）

- ROS2の基本概念（ノード、トピック、サービス、アクション）の実践
- 最小構成の例: パブリッシャ/サブスクライバノードのペア → センサ模擬ノード→判断ノード→アクチュエータ模擬ノードの3ノード構成
- シミュレータ（Gazebo, Isaac Sim等）または安価な実機（TurtleBot的な小型ロボット、あるいはマイコン+モータのみの自作機）との連携
- 学習ログ・つまずきポイントのメモ（本リポジトリの他アイデアである「エンジニアのスキルツリー」的な可視化と相性が良い可能性）

## 実現性

- ROS2自体のセットアップ・チュートリアル追従は難易度低〜中（公式ドキュメントが充実）
- Windows環境からの利用はWSL2 + Ubuntu上にROS2を構築するのが最も現実的（本PCはWindows 11）
- 実機を使う場合はハードウェア調達・電子工作の知識が必要になる（電子工作の別の取り組みとの連携も検討余地あり）
- 「最低限」に絞ればシミュレータのみで数日〜数週間で学習の土台は作れる規模感
- 作業に使うPCのスペックの制約（専用のGPUが無いPC。[README](../README.md)の「確認した環境」）: Isaac Simは使用不可、Gazeboも動作はするがぎりぎりの負荷。そのためWSL2上はフルスペックのGazebo環境を前提とせず、必要最低限の構成（後述の題材次第ではGazebo自体を使わない選択肢も含む）に絞る方針とする

## 実現方針（案）

- 第一段階: WSL2 + Ubuntu + ROS2をセットアップし（ステップ0）、ROS2の基礎（ノード・トピック・パラメータ・サービス・アクション・launch）をPython/C++の両方で動かすところ（ステップ1の1-0〜1-7）までを「最低限構成」のゴールとする
- 第二段階（題材決定）: 「慣性のある車両を目標速度に合わせて加減速させるシミュレーション」を最初の題材として採用する。Gazebo等の3D物理エンジンは必須とせず、まずは自作の疑似プラントノード（F=maの数値積分で車両の速度・加速度を計算）＋PI制御ノードによるpub/sub構成で、「センサ模擬（現在速度）→判断（目標速度との差分から制御量を計算）→アクチュエータ模擬（加減速の反映）」という最小ループを体験する。二足歩行案と比べて接触判定・剛体動力学が不要で計算負荷が軽く、作業に使うPCのスペックの制約（Isaac Sim不可・Gazeboぎりぎり）に適合する
- 将来的な二足歩行ロボットへの拡張を見据え、車両モデル固有の実装に閉じない設計を意識しながら学習する（例: アクチュエータ／センサのノード間インターフェースを抽象化しておく、トピック設計を「対象がタイヤでも脚でも通用する形」にしておく等）。ただし現段階では車両題材の完遂を優先し、拡張性のための過度な作り込みはしない
- 二足歩行（接触判定を含む剛体動力学・バランス制御・歩行パターン生成）は必要な計算負荷・専門知識ともに大きく増えるため、より性能の高いPCを使えるようになってから、または別プロジェクトとして切り出して着手する発展項目とする
- 実機連携は学習が進んでから別プロジェクトとして切り出す（作業PCのセキュリティに配慮し、外部から取得するROSパッケージ・サンプルコードの出所確認は都度行う）
- **実装時のClaude Code運用**: 実装フェーズ（別プロジェクトフォルダ）ではWSL2のUbuntu環境内にClaude Codeをネイティブインストールし、WSL2内のターミナルから直接呼び出す方針とする。ROS2のビルド（colcon）や実行（`ros2 run`等）はWSL2内で完結するため、本セッション（Windows側）からWSL越しにコマンドを都度中継するより素直
  - セキュリティ上の注意: 作業PCのセキュリティのためのグローバルの安全ルール（Windows側の`~/.claude/CLAUDE.md`・`settings.json`の`permissions.deny`）はWindows側ホームディレクトリに置かれておりWSL2側のClaude Codeセッションには自動適用されない。実装プロジェクト開始時にWSL2側の`~/.claude/CLAUDE.md`へ同等の安全ルールを複製しておく
  - PC負荷への配慮: Claude Code自体はCLIツールで待機時間が大半を占めるため、WSL2内で動かすこと自体が追加する負荷はごく小さい。負荷の主因はWSL2仮想環境自体・Gazebo・ステップ6の深層強化学習トレーニングであり、これらはROS2をWindows上で扱う以上どのみち避けられない。重い作業（Gazebo実行時、RLトレーニング時）を行う間は、本ブレインストーミング用セッション（Windows側）を閉じておくとPC全体の負荷を抑えられる
- **作業の流れと成果物の同期（GitHub経由）**: まずWindows側（本リポジトリ）で必要な情報（方針・段階分解・公式リンク等）を整理し、その後WSL2へ移行して実装する。成果物（学習資料・コード）はGitHubを経由して同期し、Windows側とWSL2側でファイルを直接共有しない
  - 役割分担: 本リポジトリ（`brainstorming`）は整理・方針の場、実装と学習資料は別リポジトリ（実装プロジェクト）で作る。実装リポジトリも**private**とし、公開範囲拡大は事前確認する
  - **責務範囲**: 実装リポジトリへの移行作業と、移行後の進捗・成果物の追跡は、本プロジェクト（`brainstorming`）の責務外とする。本リポジトリは整理と方針までを担い、移行後のリンク追記やステータス管理は行わない
  - WSL2側はcloneしたリポジトリをWSLのLinuxファイルシステム（`~`配下）に置く。Windows側のドライブ（`/mnt/c`・`/mnt/d`配下）上での作業は、ビルドが遅くなり改行コードやパーミッションの食い違いも起きやすいため避ける
  - 実装リポジトリには`.gitattributes`で改行コードをLFに統一しておく（Windows側で編集する場合のCRLF混入対策）
  - セキュリティ上の注意: WSL2側でGitHubへ認証する際は、トークン・パスワードをファイルやチャットに残さない方法（`gh auth login`の対話ログイン等）を、ユーザー自身の手で行う。WSL2側のgit設定（author名・メールアドレス）も、Windows側の設定は自動では引き継がれないため、実装開始時に確認して設定する。push前の機密情報スキャン（`git log`のメタデータ含む）は、WSL2側リポジトリでも同様に行う

## 学習ロードマップ（段階的な進め方・技術要素の整理）

いきなり最終ゴール（車両速度制御ループの完成）を目指すのではなく、以下の順で段階的に技術要素を積み上げる。各ステップに複数の選択肢がある場合は両方を挙げ、決定はステップに着手する時点でQA表に記録する。

### ステップ0: 開発環境の選択

- **Ubuntuディストリビューション ＋ ROS2バージョンの組み合わせ**
  - 選択肢A: Ubuntu 22.04 LTS + ROS2 Humble Hawksbill（安定・情報量が多いが、サポート終了が近い）
  - 選択肢B: Ubuntu 24.04 LTS + ROS2 Jazzy Jalisco（新しいLTSでサポート期間が長いが、Humbleほど情報が出揃っていない）
  - **決定: 選択肢B**。手元にある教科書がJazzyを使用しているため
- **ROS2のインストール方法**
  - 選択肢A: aptによるバイナリインストール（簡単、公式入門ルート）
  - 選択肢B: ソースからのビルド（学習にはなるが最初のステップとしては複雑すぎる）
  - **決定: 選択肢A**。インストール作業自体はユーザー自身の手で行い、Claude Codeは実行結果（コマンド履歴・設定ファイル等）のレビュー役として関わる
- 参考資料:
  - [ROS 2 Documentation: Jazzy — Installation](https://docs.ros.org/en/jazzy/Installation.html)（公式・英語）
  - [ROS 2のインストール #ROS2 - Qiita](https://qiita.com/tomoswifty/items/a2afed1d23e8ea790c9c)（日本語）
  - [Ubuntu24.04でのROS2セットアップ #ROS2 - Qiita](https://qiita.com/KimuraTomohiro/items/f3b75d204f7e962d5d21)（日本語）

### ステップ1: ROS2の基礎（Hello World相当）

- **実装言語**
  - 選択肢A: Python（rclpy）— 記述が簡単で反復学習に向く
  - 選択肢B: C++（rclcpp）— 実務での採用例が多いが学習コストが高い
  - **決定: 基礎部分（ステップ1）はPythonとC++の両方で同じ処理を作る**。同一の処理を2言語で書き比べることで、ROS2のAPI構造（ノード・コールバック・executor）と各言語固有の差を切り分けて理解する。ステップ2以降の実装言語は着手時に改めて決める（ステップ6のStable-Baselines3がPythonのため、少なくともRL連携側はPythonになる見込み）
- **インターフェース（メッセージ型）の方針**: トピック・サービス・アクションとも、まず**標準パッケージのインターフェースを優先して使う**。独自の`.msg`/`.srv`/`.action`は、標準で表現できない場合に限り後段で導入する。標準の型を使うことで、`ros2 topic echo`・rqt・rosbag2・turtlesim等の既存ツールとそのまま連携でき、二足歩行への拡張時も再利用しやすい
  - 主に使う標準パッケージ: `std_msgs`（String, Float64等）、`geometry_msgs`（Twist等）、`example_interfaces`（AddTwoInts等のサービス・アクション例）、`std_srvs`（Trigger, SetBool等）、`turtlesim`（動作確認用の標準シミュレータ）。`std_msgs`等は`common_interfaces`リポジトリに含まれる
  - 車両題材（ステップ2〜3）への接続の目安（未決定）: 目標速度・現在速度・制御出力を`std_msgs/msg/Float64`や`geometry_msgs/msg/Twist`、`nav_msgs/msg/Odometry`等の標準型で表現できるかを、ステップ1-2の学習を通じて検討する
- **資料作成方針**: ステップ1で作る学習資料（手順書・解説メモ）は、末尾に「公式ドキュメント」節を必ず設け、(1)ROS2（Jazzy版に固定したURL）、(2)利用ツール（colcon, rqt, turtlesim, rosbag2等）、(3)OSS（common_interfaces等のリポジトリ）の公式サイト・ドキュメントへのリンクを完備する。URLは推測せず、実在を確認したものだけを載せる（docs.ros.orgはボット対策で自動取得ができないため、WebSearch結果に現れたURLで実在確認する）
- 学ぶ概念: ワークスペース、パッケージ作成（`ros2 pkg create`）、ビルドツール（colcon）、ノード、トピック（pub/sub）、パラメータ、サービス、アクション、launchファイル、QoS
- **段階的な進め方（案。Claudeによる分解で、着手時にQA表で確定する。各段階で「Python版」「C++版」を同じ仕様で作り、両者の差分をメモする）**

| 段階 | 内容 | 使う標準インターフェース（例） | Python / C++で作る同一処理 |
|---|---|---|---|
| 1-0 | ワークスペース・パッケージ作成・colconビルド・`source` | （なし） | `ament_python`と`ament_cmake`で空パッケージを1つずつ作り、ビルドと実行の流れの違いを確認 |
| 1-1 | CLIツールでROS2グラフを観察（コード無し） | `turtlesim`、`geometry_msgs/msg/Twist` | `ros2 node/topic/service/param/action`、`rqt_graph`、`rqt_console`でturtlesimの通信を観察 |
| 1-2 | Publisher/Subscriber（トピック） | `std_msgs/msg/String`、`std_msgs/msg/Float64`、`geometry_msgs/msg/Twist` | ①String型のtalker/listener、②Float64を一定周期で送るtimer付きpublisherとsubscriber、③Twistをpublishしてturtlesimを動かす。あわせてQoS（reliability/durability等）を変えたpub/subで、接続できる組み合わせ・できない組み合わせを体験 |
| 1-3 | パラメータ | （パラメータ機能） | 周期・メッセージ内容をパラメータ化し、`ros2 param set`と起動時のYAML指定の両方で変更 |
| 1-4 | サービス | `example_interfaces/srv/AddTwoInts`、`std_srvs/srv/Trigger` | AddTwoIntsのサーバ・クライアント、およびTriggerでノードの状態をリセットするサーバ |
| 1-5 | アクション | `example_interfaces/action/Fibonacci`（標準の例。公式チュートリアルは独自インターフェース定義を使うため、着手時にどちらを使うか確認） | Fibonacciのアクションサーバ・クライアント（フィードバック付き） |
| 1-6 | launchファイル | （なし） | 1-2のpublisher/subscriberを1つのlaunchで起動。Python版・C++版ノードを同一launchで切り替え可能にし、相互接続（Pythonのpub × C++のsub）も確認 |
| 1-7 | 振り返り | （なし） | Python版とC++版の比較表（コード量・型の扱い・ビルド手順・起動速度・つまずき）を作り、ステップ2以降の言語選定材料にする |

- 1-2の③（Twist→turtlesim）は、標準インターフェースだけで「指令を送って動きを観察する」体験ができ、ステップ3の制御ループの導入にもなる
- ゴール: 上表の各段階について、Python版・C++版の両方を自分のWSL2環境で動かせ、両者の差分を説明できる
- 参考資料（公式ドキュメント。ROS2はJazzy版。詳細な段階別リンクは資料作成時に各資料末尾へ完備する）:
  - ROS 2全般・目次:
    - [ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/)
    - [Beginner: CLI tools — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools.html)
    - [Beginner: Client libraries — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries.html)（ワークスペース・パッケージ作成・colcon・pub/subのチュートリアルはこの目次から辿る。この4件はJazzy版の個別ページを今回の検索で直接確認できなかったため、URLの掲載は目次に留める。以下に載せた他の個別ページは、検索結果で実在を確認したもの）
    - [Intermediate — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate.html)
    - [Basic Concepts — Jazzy](https://docs.ros.org/en/jazzy/Concepts/Basic.html)
  - 1-1（CLIツール）:
    - [Using turtlesim, ros2, and rqt — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Introducing-Turtlesim/Introducing-Turtlesim.html)
    - [Using rqt_console to view logs — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Using-Rqt-Console/Using-Rqt-Console.html)
    - [Introspection with command line tools — Jazzy](https://docs.ros.org/en/jazzy/Concepts/Basic/About-Command-Line-Tools.html)
    - [Nodes — Jazzy](https://docs.ros.org/en/jazzy/Concepts/Basic/About-Nodes.html)
    - [Launching nodes — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Launching-Multiple-Nodes/Launching-Multiple-Nodes.html)
  - 1-0（ビルド）:
    - [colcon documentation](https://colcon.readthedocs.io/)（ビルドツール公式）
    - [ament_cmake user documentation — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Ament-CMake-Documentation.html)
    - [ament_cmake_python user documentation — Jazzy](https://docs.ros.org/en/jazzy/How-To-Guides/Ament-CMake-Python-Documentation.html)（純Pythonパッケージは`ament_python`を使う旨の記載あり）
  - 1-2（トピックと標準インターフェース）:
    - [ros2/common_interfaces（GitHub）](https://github.com/ros2/common_interfaces)（std_msgs, geometry_msgs, sensor_msgs, nav_msgs, std_srvs等の定義）
    - [geometry_msgs — Jazzy](https://docs.ros.org/en/jazzy/p/geometry_msgs/)
    - [rclpy API — Jazzy](https://docs.ros.org/en/jazzy/p/rclpy/)、[rclpy Node](https://docs.ros.org/en/jazzy/p/rclpy/api/node.html)
    - [C++ API rclcpp — Jazzy](https://docs.ros.org/en/jazzy/p/rclcpp/generated/index.html)
  - 1-3（パラメータ）:
    - [Understanding parameters — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters/Understanding-ROS2-Parameters.html)
    - [Using parameters in a class (Python) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-Python.html)
    - [Using parameters in a class (C++) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-CPP.html)
  - 1-4（サービス）:
    - [Writing a simple service and client (Python) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Service-And-Client.html)
    - [Writing a simple service and client (C++) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.html)
  - 1-5（アクション）:
    - [Writing an action server and client (Python) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Py.html)
    - [Writing an action server and client (C++) — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Cpp.html)
  - 1-6（launch）:
    - [Creating a launch file — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Creating-Launch-Files.html)
    - [Integrating launch files into ROS 2 packages — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-system.html)
    - [Launch tutorials — Jazzy](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-Main.html)
  - QoS（1-2の補足）:
    - [Quality of Service settings — Jazzy](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html)
    - [Class QoS — rclcpp Jazzy](https://docs.ros.org/en/jazzy/p/rclcpp/generated/classrclcpp_1_1QoS.html)
  - 日本語の補助資料:
    - [実習ROS 2 Pub&Sub通信 #ROS2 - Qiita](https://qiita.com/s-kitajima/items/5a4d7f06413120010e6b)（日本語）
    - [ROS 2のワークスペース：colconとパッケージ](https://gbiggs.github.io/rosjp_ros2_intro/workspaces_and_colcon.html)（日本語・colcon解説）

### ステップ2: 疑似プラント（車両シミュレーション）ノードの自作

- **車両モデルの単純化レベル**
  - 選択肢A: 1D質点モデル（直進のみ、F=maを積分）— 今回の題材（目標速度への加減速）には最小限で十分
  - 選択肢B: 自転車モデル（Bicycle Model、操舵を含む2D運動）— より発展的。二足歩行より先に「非直進」の運動を扱う中間ステップとしても使える
  - **決定: 選択肢A**。今回の題材には最小限で十分と判断
- **数値積分方法**
  - 選択肢A: オイラー法（実装が簡単、精度は粗い）
  - 選択肢B: ルンゲクッタ法（RK4）（実装はやや複雑、精度が高い）
  - **決定: 選択肢A**。実装が簡単なため
- 設計上の配慮: 上記2点はいずれも将来モデルを差し替えやすいよう、プラントノードの内部実装（質点モデル／積分方法）をpub/subのトピックインターフェース（目標入力・状態出力）から分離しておく。自転車モデルやRK4への入れ替え、あるいは将来の二足歩行モデルへの置き換えの際に、制御ノード側や周辺のインターフェースに影響が及ばないようにする
- **アクチュエータダイナミクス**: 制御ノードからの指令値（アクセル／ブレーキ）が力に変換されるまでを単純な瞬時変換ではなく、アクセルとブレーキそれぞれ別の伝達関数（例: 時定数の異なる一次遅れ）を通す構成とする。両者を合成した力をF=maの積分入力とする。左右非対称なアクチュエータ特性を扱う練習として、質点モデル自体を発展させる形で組み込む
- ゴール: 目標速度・現在速度をトピックとして流し、時間経過とともに車両速度が変化するノードが動く
- 参考資料:
  - [PID制御の基本理論と設計法：幅広く使われるPID制御 - 制御工学ブログ](https://blog.control-theory.com/entry/pid-control)（日本語・伝達関数・一次遅れ系の考え方）
  - [PID制御チューニング完全ガイド：基礎理論からプロの実践技術まで](https://denki-study.com/pid%e5%88%b6%e5%be%a1%e3%83%81%e3%83%a5%e3%83%bc%e3%83%8b%e3%83%b3%e3%82%b0%e5%ae%8c%e5%85%a8%e3%82%ac%e3%82%a4%e3%83%89%ef%bc%9a%e5%9f%ba%e7%a4%8e%e7%90%86%e8%ab%96%e3%81%8b%e3%82%89%e3%83%97/)（日本語・FOPDTモデルの解説）

### ステップ3: 制御ノードの自作

- **制御アルゴリズム**
  - 選択肢A: P制御（比例制御のみ）— シンプルで挙動を理解しやすい
  - 選択肢B: PID制御（比例・積分・微分）— オーバーシュート抑制等、より実用的
  - **決定: PI制御（比例・積分）**。PID制御（選択肢B）は他の開発環境で使用経験があるため、微分項は省略し選択肢Aを積分項まで拡張する形（定常偏差の解消）に留める
- ゲイン（Kp・Ki）の具体値は本アイデア段階では未定。アクセル／ブレーキが異なる伝達関数を持つ非対称なプラント（ステップ2参照）に対してチューニングする必要があるため、実装段階でシミュレーション上の試行錯誤により決定する
- パラメータの持たせ方: ステップ1-3で学んだROS2のパラメータ機能（`ros2 param`、YAMLファイル）を応用し、Kp・Kiを起動時のYAMLで与え、実行中に`ros2 param set`で変更してチューニングする
- ゴール: 「センサ模擬（現在速度）→判断（制御ノード）→アクチュエータ模擬（プラントノードへの入力）」のループが閉じ、目標速度に追従する
- 参考資料:
  - [PID制御とは？仕組みと動作イメージを分かりやすく解説！](https://controlabo.com/pid-control-introduction/)（日本語）
  - [YAMLファイルによるROS2のパラメータ設定 #ROS2 - Qiita](https://qiita.com/NeK/items/15bf1e657d8d694592ed)（日本語）

### ステップ4: 可視化・分析

- **可視化ツール**
  - 選択肢A: rqt_plot（ROS2標準ツール、手軽）
  - 選択肢B: PlotJuggler（より高機能、`ros2 bag`との連携も強力）
  - **決定: 選択肢A**。手軽さを優先し、まずはこちらで始める
- 設計上の配慮: 可視化ツールはいずれもROS2標準のトピック購読で動くため、プラントノード側の設計変更（ステップ2参照）は不要で、後からPlotJuggler等へ入れ替える余地を残せる
- ログ収集: `ros2 bag record`/`play`
- ゴール: 目標速度・実速度・制御出力の時系列グラフから制御の効き方を評価できる
- 参考資料:
  - [【ROS2講座⑥】rosbag2とrqtツールで記録と可視化【Python】](https://sakigake-robo.com/ros2-6/)（日本語）
  - [ROS 2 For Beginners (ROS Jazzy – 2025): トピック・ツール｜Hafnium](https://note.com/hafnium/n/ne97ab2501c12)（日本語）

### ステップ5（任意・スペック次第）: Gazebo導入検討

- **Gazeboのバージョン**
  - 選択肢A: Gazebo Classic（gazebo11）— ROS2との連携事例が豊富だが公式サポート終了が近い
  - 選択肢B: 新しいGazebo（Gazebo Harmonic等、旧Ignition系）— 公式後継だが情報はまだ少なめ
  - 選択肢C（見送り案）: Gazeboを使わず自作ノードのみで完結させる — 車両速度制御の題材自体はGazeboが無くても成立するため、作業に使うPCのスペックの制約が厳しい場合はこちらが現実的
  - **決定: 導入する場合は選択肢B**。選択肢Aは連携事例こそ豊富だが公式サポート終了が近く、本企画の目的（フィジカルAIそのものではなく開発に必要な技術スタックを一通り身に着けること）に照らすとコストパフォーマンスが悪いと判断した。ただしステップ5自体が「任意・スペック次第」の位置づけである点は変わらず、実施可否・見送り（選択肢C）の判断は着手時点で改めて行う
- ゴール: 余力があれば、車両を模した簡単な3Dモデル（箱一つ等）をGazebo上で動かし、可視化の質を上げる
- 参考資料:
  - [Installing Gazebo with ROS — Gazebo documentation](https://gazebosim.org/docs/latest/ros_installation/)（公式・英語）
  - [ROS 2初心者向けレベル2 – TF | URDF | RViz | Gazebo：Gazebo｜Hafnium](https://note.com/hafnium/n/n29e7e879da18)（日本語）
  - [ros2 jazzy gazebo harmonic インストール手順 #ROS - Qiita](https://qiita.com/akinami/items/06ad4926e414355e48e1)（日本語）
  - [gazeboとROS2を連携してロボットを操作する #Gazebo - Qiita](https://qiita.com/N622/items/9d89f77d85d9da0af29e)（日本語）

### ステップ6: 深層強化学習によるPIゲイン調整

- 位置づけ: ステップ3で導入したPI制御のゲイン（Kp・Ki）を、手動チューニングではなく深層強化学習エージェントに調整させる。二足歩行ロボットのバランス制御は状況に応じた動的なゲイン調整（あるいは方策そのものの学習）が要となるため、車両題材のうちに深層強化学習の基礎（環境設計・報酬設計・学習ループ）を身につけておく橋渡しとして位置づける
- **強化学習ライブラリ**
  - 選択肢A: Stable-Baselines3（PPO/SAC等の主要アルゴリズムが揃った定番ライブラリ、学習コストが低い）
  - 選択肢B: 自作実装（学習にはなるが、本題である「PIゲイン調整」から逸れて実装コストが増す）
- **学習環境の構成**
  - 選択肢A: ステップ2のプラントノード・ステップ3の制御ノードをROS2越しにGym的なインターフェースでラップし、実際のpub/sub構成のまま学習させる（ROS2との統合を体験できるが、通信オーバーヘッドで学習が遅くなる）
  - 選択肢B: プラントモデル（質点＋アクチュエータダイナミクス）をROS2から切り離したPython単体の環境として再実装して高速に学習させ、得られたゲイン（またはゲインスケジューリング方策）をROS2の制御ノードに反映する（学習は高速だが、二重実装の手間・仕様乖離のリスクがある）
- 報酬設計の検討事項: 目標速度への追従誤差・オーバーシュート・制御出力の変化量（滑らかさに相当）等をどう重みづけるか
- ゴール: 深層強化学習によって手動チューニングと同等以上にPIゲインを調整できることを確認し、二足歩行ロボットのバランス制御学習に進むための強化学習の基礎（環境設計・報酬設計・学習ループ）を身につける
- 参考資料:
  - [Stable-Baselines3 Documentation](https://stable-baselines3.readthedocs.io)（公式・英語）
  - [Stable Baselines 3 入門 (1) - 強化学習アルゴリズム実装セット｜npaka](https://note.com/npaka/n/nb615fd590274)（日本語）
  - [強化学習を用いたセルフチューニングPID制御器の設計 - J-STAGE](https://www.jstage.jst.go.jp/article/jsmecs/2007.45/0/2007.45_327/_article/-char/ja/)（日本語・論文、PIDゲインをRLで調整する着想の参考）
  - [ROS 2によるデータ駆動PIDゲイン自動調整ツール #C++ - Qiita](https://qiita.com/KariControl/items/8e269415692312a6c5f4)（日本語・本ステップと直接関連するテーマ）
  - [Stable Baselines3 を使って強化学習をカスタムSIに使ってみよう #PyTorch - Qiita](https://qiita.com/hara2dev/items/5adc655ec3867a4b862a)（日本語）

### ステップ7: 二足歩行への拡張を見据えた橋渡し学習

- 座標変換ライブラリ: tf2（車両・歩行ロボットのどちらでも必須となる共通基礎）
- ロボット記述フォーマット: URDF / xacro（車両の簡単なURDFを先に作り、後で歩行ロボットURDFへ発展させる練習にする）
- **アクチュエータ制御の枠組み**
  - 選択肢A: 自作ノードのみで完結（学習コストは低いが標準的な枠組みには触れない）
  - 選択肢B: ros2_controlフレームワークに早めに触れる（車両のモータ制御にも歩行の関節制御にも使う標準的な基盤。学習コストはやや高いが、将来の歩行拡張時の学び直しが減る）
- ゴール: 車両題材が一段落した時点で、二足歩行に必要な知識ギャップ（接触判定を含む動力学、バランス制御、歩行パターン生成等）を整理し、次プロジェクトの検討材料にする
- 参考資料:
  - [ROS 2初心者向けレベル2 – TF | URDF | RViz | Gazebo：TF｜Hafnium](https://note.com/hafnium/n/nc80fc646521f)（日本語）
  - [ROS 2初心者向けレベル2 – TF | URDF | RViz | Gazebo：URDF｜Hafnium](https://note.com/hafnium/n/n927d0e312e21)（日本語）
  - [ROS 2初心者向けレベル2 – TF | URDF | RViz | Gazebo：Xacro｜Hafnium](https://note.com/hafnium/n/nda1e9fadfd0c)（日本語）
  - [ros2_control documentation — Jazzy](https://control.ros.org/jazzy/index.html)（公式・英語）
  - [micro-ROSとros2_controlで構成したDiff Botでナビゲーションしてみた #ROS2 - Qiita](https://qiita.com/BEIKE/items/03fa2e4f43ee570bddd1)（日本語）

## 類似の既存事例（Web調査）

- The Construct「ROS2 Basics Course」: ROS2初心者向けの体系的なオンライン講座
- NVIDIA Isaac Sim + ROS2の連携チュートリアル（NVIDIA公式ブログ）
- ROBOTIS公式「Setup Guide — ROS 2 (Physical AI Tools)」: Physical AI向けの環境構築ガイド
- Gazebo MCPチュートリアルシリーズ: 初心者から段階的にAI支援ツールを使ったロボティクス開発を学べる教材
- **差別化点・示唆**: ROS2＋Physical AIの学習教材自体は充実しているが、いずれも個別領域（基礎操作、シミュレータ連携、特定ハードウェア）に特化している。「最低限構成」に絞って全体を見通せる個人用スターターキットという切り口は、既存教材の要点を取捨選択して束ねる位置づけになりそう
