# 手順書: WSL2 + Ubuntu 24.04 + ROS2 Jazzy の環境構築（ステップ0）

## 目的・ゴール

WSL2上にUbuntu 24.04とROS2 Jazzy（aptバイナリ）を構築し、ROS2が動作することを確認する。

**完了の判定基準**（全て満たしたら完了）

1. `wsl -l -v` で Ubuntu-24.04 が VERSION 2 として表示される
2. Ubuntu内で `lsb_release -a` が 24.04 を示す
3. `printenv ROS_DISTRO` が `jazzy` を返す
4. `ros2 run demo_nodes_cpp talker` と `ros2 run demo_nodes_py listener` を別ターミナルで動かし、メッセージが届く
5. 作業ディレクトリが WSL の Linux ファイルシステム（`~` 配下）である（`/mnt/c`・`/mnt/d` 配下ではない）

**規模の目安**: 半日程度（ダウンロード時間・再起動を含む）。

## 役割分担・注意

- インストール作業は**ユーザー自身の手**で行う。Claude Code は、貼られたコマンド履歴・出力のレビューを担当する（元アイデアの決定事項）。
- パスワード（Linuxユーザー、`sudo`）はチャット・ファイル・ログに残さない。
- ROS2の導入はaptリポジトリとGPGキーの追加を伴う。**公式ページに載っている手順・出所のみ**を使い、ブログ等に載っている別の取得元は使わない。
- 本手順ではWindows側のセキュリティ設定（Defender・ファイアウォール等）は変更しない。変更が必要に見える場面では止まって確認する。

## 手順

### 1. 事前確認（Windows側）

- BIOS/UEFIで仮想化支援（Intel VT-x / AMD-V）が有効か確認する。タスクマネージャー → パフォーマンス → CPU の「仮想化: 有効」で確認できる。
- 空き容量の目安: 20GB以上（Ubuntu + ROS2 desktop）。

### 2. WSL2 と Ubuntu 24.04 の導入（Windows側）

1. **管理者権限**のPowerShellで `wsl --install -d Ubuntu-24.04` を実行する。案内は https://aka.ms/wslinstall にある。
2. 再起動を求められたら再起動する。
3. Ubuntuの初回起動でLinuxのユーザー名とパスワードを設定する。
4. 確認: PowerShellで `wsl -l -v`（VERSIONが2であること）、`wsl --version`。
   - VERSIONが1の場合は `wsl --set-version Ubuntu-24.04 2` で変換する。

### 3. Ubuntuの更新（Ubuntu内）

1. `sudo apt update && sudo apt upgrade` を実行する。
2. 確認: `lsb_release -a` が24.04を示す。
3. 作業場所は `~` 配下にする。`pwd` が `/home/<ユーザー名>` 配下であることを確認する。

### 4. ROS2 Jazzy の導入（Ubuntu内・公式手順）

公式の Installation ページ（Jazzy）の、**Ubuntu (deb packages)** の手順に上から従う。大まかな流れは次のとおり（具体的なコマンドは公式ページの記載を正とし、ここには書き写さない）。

1. ロケール（UTF-8）の設定
2. 必要なリポジトリ（universe）と、ROS2のaptソースの追加
3. 開発ツール（`ros-dev-tools`。colconを含む）の導入
4. `sudo apt update` / `sudo apt upgrade` の後に、ROS2本体の導入。学習用には **Desktop Install**（ros-jazzy-desktop。rqt・turtlesim等を含む）を選ぶ。
5. 環境の読み込み: `source /opt/ros/jazzy/setup.bash`
   - 毎回入力しないよう `~/.bashrc` へ追記する場合は、追記内容を自分で確認したうえで行う。

### 5. 動作確認（Ubuntu内）

1. `printenv ROS_DISTRO` が `jazzy` を返す。
2. ターミナルA: `ros2 run demo_nodes_cpp talker`
3. ターミナルB: `ros2 run demo_nodes_py listener`
4. Bにメッセージが表示されることを確認し、A・Bとも Ctrl+C で終了する。

なお `rqt` や `turtlesim` などGUIアプリの表示（WSLg）は、ステップ1-1で必要になる。本手順の完了条件には含めない。

### 6. Claude Codeによるレビュー（任意）

実施した内容のレビューを希望する場合は、次を貼り付ける。**認証情報・パスワードが含まれないことを確認してから**貼ること。

- `history` の該当部分（または実行したコマンド）
- 各確認コマンドの出力
- `~/.bashrc` に追記した場合は、追記した行

## つまずきやすい点

- 仮想化が無効だと `wsl --install` が失敗する → 手順1のBIOS設定を確認する
- `/mnt/c`・`/mnt/d` 配下で作業するとビルドが遅くなる → `~` 配下で作業する
- ターミナルを開き直すと `ros2` が見つからない → `source /opt/ros/jazzy/setup.bash` が未実行

## 公式ドキュメント

- [ROS 2 Documentation: Jazzy — Installation](https://docs.ros.org/en/jazzy/Installation.html)（公式・英語）
- [ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/)
- [colcon documentation](https://colcon.readthedocs.io/)
- WSLのインストール案内: https://aka.ms/wslinstall（`wsl` コマンドが案内するMicrosoft公式の短縮URL）
- 日本語の補助資料:
  - [ROS 2のインストール #ROS2 - Qiita](https://qiita.com/tomoswifty/items/a2afed1d23e8ea790c9c)
  - [Ubuntu24.04でのROS2セットアップ #ROS2 - Qiita](https://qiita.com/KimuraTomohiro/items/f3b75d204f7e962d5d21)
