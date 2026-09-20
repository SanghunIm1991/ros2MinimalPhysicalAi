# 手順書: WSL2 + Ubuntu 24.04 + ROS2 Jazzy の環境構築（ステップ0）

## 目的・ゴール

WSL2上にUbuntu 24.04とROS2 Jazzy（aptバイナリ）を構築し、ROS2が動作することを確認する。

**完了の判定基準**（全て満たしたら完了）

1. `wsl -l -v` で Ubuntu-24.04 が VERSION 2 として表示される
2. Ubuntu内で `lsb_release -a` が 24.04 を示す
3. `printenv ROS_DISTRO` が `jazzy` を返す
4. `ros2 run demo_nodes_cpp talker` と `ros2 run demo_nodes_py listener` を別ターミナルで動かし、メッセージが届く
5. 作業ディレクトリが WSL の Linux ファイルシステム（`~` 配下）である（`/mnt/c`・`/mnt/d` 配下ではない）
6. WSLの仮想ディスクがD:にある（`D:\WSL\Ubuntu-24.04\ext4.vhdx` が存在する）

**規模の目安**: 半日程度（ダウンロード時間・再起動を含む）。

## 役割分担・注意

- インストール作業は**ユーザー自身の手**で行う。Claude Code は、貼られたコマンド履歴・出力のレビューを担当する（元アイデアの決定事項）。
- パスワード（Linuxユーザー、`sudo`）はチャット・ファイル・ログに残さない。
- ROS2の導入はaptリポジトリとGPGキーの追加を伴う。**公式ページに載っている手順・出所のみ**を使い、ブログ等に載っている別の取得元は使わない。
- 本手順ではWindows側のセキュリティ設定（Defender・ファイアウォール等）は変更しない。変更が必要に見える場面では止まって確認する。

## 手順

### 1. 事前確認（Windows側）

- BIOS/UEFIで仮想化支援（Intel VT-x / AMD-V）が有効か確認する。タスクマネージャー → パフォーマンス → CPU の「仮想化: 有効」で確認できる。
- 空き容量の目安: 20GB以上（Ubuntu + ROS2 desktop）。C:の空きは30GB程度と少ないため、仮想ディスクは手順2bでD:（空き651GB）へ移す。
- メモリ（実測7.7GB）は、WSL2の既定（約半分）のまま動かして様子を見る。不足が出た場合に、上限の調整を別途検討する。

### 2. WSL2 と Ubuntu 24.04 の導入（Windows側）

1. **管理者権限**のPowerShellで `wsl --install -d Ubuntu-24.04` を実行する。案内は https://aka.ms/wslinstall にある。
2. 再起動を求められたら再起動する。
3. Ubuntuの初回起動でLinuxのユーザー名とパスワードを設定する。
4. 確認: PowerShellで `wsl -l -v`（VERSIONが2であること）、`wsl --version`。
   - VERSIONが1の場合は `wsl --set-version Ubuntu-24.04 2` で変換する。

### 2b. Ubuntuの保存先をDドライブへ移す（Windows側・ROS2導入の前に行う）

**目的**: WSL2の仮想ディスク（`ext4.vhdx`）は既定でC:に作られる。C:の空きは30GB程度のため、ROS2導入前の空の状態でD:へ移す。**必ずROS2導入（手順4）より前に行う**（導入後に行うと、エクスポートのファイルが大きくなり、失敗時の影響も大きくなる）。

**影響**: D:にフォルダを作り、WSLのディスクイメージを置く。外部通信・Windowsのセキュリティ設定の変更はない。`wsl --unregister` は対象ディストリビューションのデータを完全に削除するため、**エクスポートファイルの存在とサイズを確認してから**実行する（この時点のUbuntuは導入直後で、失うものはほぼない）。

**方法A: エクスポート/インポート（どのWSLバージョンでも使える標準手順）**

1. 手順2の初回起動でユーザー名・パスワードを設定済みであること、およびUbuntu内に手作業で作ったデータ・ファイルがないこと（導入直後の状態）を確認する。データがある場合は、先に必要なものを退避する。
2. Ubuntu内で、既定ユーザーを固定する。インポート後は既定ユーザーがrootに戻るため、その対策である。まず `cat /etc/wsl.conf` で既存の内容を確認する（通常は存在しない）。既存の設定がある場合は上書きせず、`[user]` の節を追記する。ない場合は `sudo nano /etc/wsl.conf` で次を書き、保存する（`<ユーザー名>` は手順2で作った名前）。
   ```
   [user]
   default=<ユーザー名>
   ```
3. PowerShellで作業フォルダを作る: `New-Item -ItemType Directory -Force D:\WSL\backup, D:\WSL\Ubuntu-24.04`
4. Ubuntuを停止: `wsl --shutdown`（他のWSLディストリビューションも含め、WSL全体が停止する。他に稼働中のものがあれば先に保存しておく）
5. エクスポート: `wsl --export Ubuntu-24.04 D:\WSL\backup\ubuntu-24.04-fresh.tar`
6. **確認**: `Get-Item D:\WSL\backup\ubuntu-24.04-fresh.tar` でファイルがあり、サイズが数百MB以上であることを確認する（極端に小さい・0の場合はエクスポート失敗。手順5をやり直す）。
7. C:側を登録解除する。実行前に `wsl -l -v` で、対象が `Ubuntu-24.04` であることを再確認する。確認が取れてから `wsl --unregister Ubuntu-24.04` を実行する（他の名前のディストリビューションを指定しない）。
8. D:へインポート: `wsl --import Ubuntu-24.04 D:\WSL\Ubuntu-24.04 D:\WSL\backup\ubuntu-24.04-fresh.tar --version 2`
9. 確認: `wsl -l -v`（Ubuntu-24.04がVERSION 2）、`wsl -d Ubuntu-24.04` で入り、`whoami` が手順2のユーザー名になっていること。rootなら手順2の `/etc/wsl.conf` を確認する。
10. 仮想ディスクの場所を確認: `Test-Path D:\WSL\Ubuntu-24.04\ext4.vhdx` が `True` であること。
11. バックアップのtarは、環境が安定して動くことを確認できるまで残す。その後は削除してよい（D:に651GBの空きがあるため急がない）。

**方法B: `wsl --manage --move`（対応バージョンのWSLのみ・任意）**

新しいバージョンのWSLには、登録済みディストリビューションを移動するコマンドがある。`wsl --manage --help` に `--move` が表示される場合に限り、`wsl --manage Ubuntu-24.04 --move D:\WSL\Ubuntu-24.04` で移せる。表示されない場合は方法Aを使う。

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
- インポート後にrootでログインされる → `/etc/wsl.conf` の `[user] default=` が未設定（手順2b-2）
- ターミナルを開き直すと `ros2` が見つからない → `source /opt/ros/jazzy/setup.bash` が未実行

## 公式ドキュメント

- [ROS 2 Documentation: Jazzy — Installation](https://docs.ros.org/en/jazzy/Installation.html)（公式・英語）
- [ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/)
- [colcon documentation](https://colcon.readthedocs.io/)
- WSLのインストール案内: https://aka.ms/wslinstall（`wsl` コマンドが案内するMicrosoft公式の短縮URL）
- 日本語の補助資料:
  - [ROS 2のインストール #ROS2 - Qiita](https://qiita.com/tomoswifty/items/a2afed1d23e8ea790c9c)
  - [Ubuntu24.04でのROS2セットアップ #ROS2 - Qiita](https://qiita.com/KimuraTomohiro/items/f3b75d204f7e962d5d21)
