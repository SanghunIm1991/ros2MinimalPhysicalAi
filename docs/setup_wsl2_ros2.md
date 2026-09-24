# 環境構築 手順書: WSL2 + Ubuntu 24.04 + ROS2 Jazzy（ステップ0）

WindowsのPCに、WSL2（Windowsの上でLinuxを動かす仕組み）でUbuntu 24.04を入れ、その中にROS2 Jazzyを導入する。フェーズ0（読み物）の後、フェーズ1で実際に手を動かす前に行う、この教材の最初の作業である。

- 想定環境: Windows 11（WSL2が使えるWindows 10でもよい）。CPUの仮想化支援機能が有効であること（1節）
- 所要目安: 2時間程度（ダウンロードの時間とWindowsの再起動を含む。回線の速さで前後する。任意の2b節・7節は含まない）
- 必須の範囲: 1〜5節と、6節（作業ディレクトリ）
- 任意の範囲: **2b節（仮想ディスクを別のドライブへ移す）** と、**7節（GitHubの認証・Claude Codeの環境）**

> **この手順書の位置づけ**: 必須の範囲だけで、フェーズ1以降の手順書をすべて進められる。任意の2つの節は、必要な人だけが行えばよい。2b節は、Cドライブの空きが少ないPCで、Ubuntuの保存先を別のドライブへ移す手順である。7節は、非公開のGitHubリポジトリを扱う場合や、Claude Code（AIのコーディング支援ツール）をWSLの中で使う場合の準備で、この教材の作者の運用に合わせたものである。

> **実行環境が無くても読めるように**: 確認のコマンドの直後には「期待する結果」として、表示される内容の例とその読み方を載せている。Windows側（PowerShell）の表示は各ツールの仕様から想定したもの、Ubuntu側の表示は筆者の環境で確かめたものと仕様から想定したものが混ざっていて、版の数字や言語（日本語/英語）は環境によって異なる。

## 目的・ゴール

WSL2上にUbuntu 24.04とROS2 Jazzy（aptのバイナリパッケージ）を導入し、ROS2が動作することを確認する。

**完了の判定基準**（すべて満たしたら完了）

1. `wsl -l -v` で、Ubuntu-24.04 が VERSION 2 として表示される（2節）
2. Ubuntuの中で `lsb_release -a` が 24.04 を示す（3節）
3. `printenv ROS_DISTRO` が `jazzy` を返す（5節）
4. `ros2 run demo_nodes_cpp talker` と `ros2 run demo_nodes_py listener` を別々のターミナルで動かし、メッセージが届く（5節）
5. 作業ディレクトリが、WSLのLinuxのファイルシステム（`~` の下）にある（`/mnt/c`・`/mnt/d` の下ではない）（6節）

2b節を行った場合は、あわせて「WSLの仮想ディスク（`ext4.vhdx`）が移動先のドライブにある」ことも確認する。

## 注意

- 導入の作業は、**自分の手で**行う。コマンドの意味が分からないまま貼り付けて実行しない。
- パスワード（Linuxのユーザー、`sudo`）は、チャット・ファイル・ログに残さない。
- ROS2の導入は、aptのリポジトリとGPGキーの追加を伴う。**公式ページに載っている手順・取得元だけ**を使い、ブログ等に載っている別の取得元は使わない。
- この手順では、Windows側のセキュリティ設定（Defender・ファイアウォール等）は変更しない。変更が必要に見える場面では、止まって確認する。

## 手順

### 1. 事前確認（Windows側）

- **仮想化支援**: BIOS/UEFIで、CPUの仮想化支援（Intel VT-x / AMD-V）が有効か確認する。タスクマネージャー → パフォーマンス → CPU の「仮想化: 有効」で確認できる。
- **空き容量**: 目安は20GB以上（Ubuntu + ROS2 desktop）。WSLの仮想ディスクは既定でCドライブに作られる。Cドライブの空きが少なく、別のドライブ（Dドライブなど）に余裕がある場合は、2節の後に2b節（任意）で移す。
- **メモリ**: WSL2の既定（物理メモリの約半分をWSL2に割り当てる設定）のまま始めてよい。この教材のビルド（フェーズ2〜4）は、既定の設定で足りることを確認している。不足を感じた場合は、上限の調整（`.wslconfig`）を別途検討する。

### 2. WSL2 と Ubuntu 24.04 の導入（Windows側）

1. **管理者権限**のPowerShellで `wsl --install -d Ubuntu-24.04` を実行する。案内は https://aka.ms/wslinstall にある。
2. 再起動を求められたら再起動する。
3. Ubuntuの初回起動で、Linuxのユーザー名とパスワードを設定する（Windowsのアカウントとは別のもの）。
4. 確認: PowerShellで `wsl -l -v`（VERSIONが2であること）と、`wsl --version` を実行する。
   - VERSIONが1の場合は `wsl --set-version Ubuntu-24.04 2` で変換する。

期待する結果（版の数字は更新によって変わる。表示の言語はWindowsの設定による）:

```text
PS> wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-24.04    Running         2

PS> wsl --version
WSL バージョン: 2.x.x.x
カーネル バージョン: 6.x.x.x-x
WSLg バージョン: 1.0.xx
...
```

- `wsl -l -v` の `*` は既定のディストリビューション（`wsl` とだけ打ったときに起動するもの）の印。`VERSION` が `2` であることが大事で、`1` のままだとROS2のGUI表示（WSLg）などが使えない。
- `wsl --version` に `WSLg バージョン` の行があれば、フェーズ1で使うGUIの表示の仕組み（WSLg）が入っている。

### 2b.（任意）Ubuntuの保存先を別のドライブへ移す（Windows側・ROS2導入の前に行う）

**この節が必要な人**: Cドライブの空きが少ない（目安として20GBを大きく下回る）場合だけ行う。空きに余裕があれば飛ばして3節へ進む。以下は、移動先をDドライブ（`D:\WSL\...`）とした例で書く。別のドライブにする場合は読み替える。

**目的**: WSL2の仮想ディスク（`ext4.vhdx`）は、既定でCドライブに作られる。ROS2を導入する前の、中身がほぼ空の状態で、別のドライブへ移す。**必ずROS2の導入（4節）より前に行う**（導入後に行うと、エクスポートのファイルが大きくなり、失敗したときの影響も大きくなる）。

**影響**: 移動先のドライブにフォルダを作り、WSLのディスクイメージを置く。外部との通信や、Windowsのセキュリティ設定の変更はない。`wsl --unregister` は対象のディストリビューションのデータを完全に削除するため、**エクスポートしたファイルの存在とサイズを確認してから**実行する（この時点のUbuntuは導入直後で、失うものはほぼない）。

**方法A: エクスポート/インポート（どのWSLバージョンでも使える標準の手順）**

1. 2節の初回起動でユーザー名・パスワードを設定済みであること、およびUbuntuの中に自分で作ったデータ・ファイルが無いこと（導入直後の状態）を確認する。データがある場合は、先に必要なものを退避する。
2. Ubuntuの中で、既定のユーザーを固定する。インポートした後は既定のユーザーがrootに戻るため、その対策である。まず `cat /etc/wsl.conf` で既存の内容を確認する。Ubuntu 24.04では、systemd関連の設定（`[boot]` 節など）で既に存在することがある。その場合は既存の内容を消さず、`[user]` の節だけを追記する。存在しない場合は新しく作る。編集は `sudo nano /etc/wsl.conf` で行い、次の内容を書いて保存する（`<ユーザー名>` は2節で作った名前）。
   ```
   [user]
   default=<ユーザー名>
   ```
   保存後に `cat /etc/wsl.conf` をもう一度実行し、`[user]` 節が入っていること、既存の節が残っていることを確認する。期待する結果の例（`[boot]` 節が元からあった場合）:
   ```text
   [boot]
   systemd=true

   [user]
   default=<ユーザー名>
   ```
3. PowerShellで作業フォルダを作る: `New-Item -ItemType Directory -Force D:\WSL\backup, D:\WSL\Ubuntu-24.04`
4. Ubuntuを停止する: `wsl --shutdown`（他のWSLのディストリビューションも含め、WSL全体が停止する。他に動いているものがあれば、先に作業を保存しておく）
5. エクスポートする: `wsl --export Ubuntu-24.04 D:\WSL\backup\ubuntu-24.04-fresh.tar`
6. **確認**: `Get-Item D:\WSL\backup\ubuntu-24.04-fresh.tar` でファイルがあり、サイズ（`Length` の列。単位はバイト）が数百MB以上であることを確認する。極端に小さい・0の場合は、エクスポートに失敗している。5番目の手順をやり直す。
7. Cドライブ側の登録を解除する。実行前に `wsl -l -v` で、対象が `Ubuntu-24.04` であることを再確認する。確認が取れてから `wsl --unregister Ubuntu-24.04` を実行する（他の名前のディストリビューションを指定しない）。
8. 移動先へインポートする: `wsl --import Ubuntu-24.04 D:\WSL\Ubuntu-24.04 D:\WSL\backup\ubuntu-24.04-fresh.tar --version 2`
9. 確認: `wsl -l -v`（Ubuntu-24.04がVERSION 2）を実行し、`wsl -d Ubuntu-24.04` で入って、`whoami` が2節のユーザー名になっていることを確かめる。`root` と表示されたら、2番目の手順の `/etc/wsl.conf` を確認する。なお、インポートした後は、Windows Terminalのプロファイルやスタートメニューの登録が変わる場合がある。`wsl -d Ubuntu-24.04` で入れれば問題ない。
10. 仮想ディスクの場所を確認する: `Test-Path D:\WSL\Ubuntu-24.04\ext4.vhdx` が `True` と表示されること。
11. バックアップのtarファイルは、環境が安定して動くことを確認できるまで残す。その後は削除してよい。

**方法B: `wsl --manage --move`（対応する版のWSLだけ・任意）**

新しい版のWSLには、登録済みのディストリビューションを移動するコマンドがある。`wsl --manage --help` に `--move` が表示される場合に限り、`wsl --manage Ubuntu-24.04 --move D:\WSL\Ubuntu-24.04` で移せる。移動先のフォルダは、事前に方法Aの3番目の手順のコマンドで作っておく。表示されない場合は方法Aを使う。なお、方法Aの2番目の手順（既定のユーザーの固定）は、方法Bでは不要である。

### 3. Ubuntuの更新（Ubuntu内）

1. `sudo apt update && sudo apt upgrade` を実行する。
2. 確認: `lsb_release -a` が24.04を示す。期待する結果の例:

   ```text
   No LSB modules are available.
   Distributor ID: Ubuntu
   Description:    Ubuntu 24.04.1 LTS
   Release:        24.04
   Codename:       noble
   ```

   `Release` が `24.04`、`Codename` が `noble` であればよい（`Description` の末尾の小さな版数は、更新によって変わる）。
3. 作業場所は `~`（ホームディレクトリ）の下にする。`pwd` が `/home/<ユーザー名>` の下であることを確認する。
4. `pwd` が `/mnt/c/...` や `/mnt/d/...` になる場合: `wsl` は、Windows側の現在のフォルダ（例: `D:\`）をそのまま引き継ぐ仕様のためで、異常ではない。ホームで開始するには、起動時に `wsl --cd ~ -d Ubuntu-24.04`（または `wsl ~ -d Ubuntu-24.04`）と指定する。毎回指定したくない場合は、Windows Terminalの設定で、Ubuntu-24.04のプロファイルの「開始ディレクトリ」を `\\wsl.localhost\Ubuntu-24.04\home\<ユーザー名>`（古い形式は `\\wsl$\...`）にする（Windows Terminalを使い、かつUbuntu-24.04のプロファイルがある場合だけ。2b節の `wsl --import` で作った環境では、プロファイルが自動で作られないことがある）。`~/.bashrc` に `cd ~` を書く方法は、`wsl --cd <パス>` で意図的に別の場所から開始したときも上書きされるため勧めない。

### 4. ROS2 Jazzy の導入（Ubuntu内・公式手順）

公式の Installation ページ（Jazzy）の、**Ubuntu (deb packages)** の手順に上から従う。

- URL: <https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html>
- 以下のコマンドは、上記ページ（ソースは `ros2/ros2_documentation` リポジトリの `jazzy` ブランチ）から抜粋し、コメントと注意書きを加えたもの（改変あり）。ROS 2ドキュメントは [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) で公開されている。
- 取得は要約を経由したため、**逐語の転記ではない**。内容は2026-09-20時点のもので、版により変わりうる。**実行前に公式ページと照合し、食い違いがあれば公式ページを正とする**。
- ページ上部の Anubis（bot対策）により自動取得ができなかったため、GitHub上のソースから取得した。

1. ロケール（UTF-8）の設定。`locale` で UTF-8 になっていれば不要。

   ```bash
   sudo apt update && sudo apt install locales
   sudo locale-gen en_US en_US.UTF-8
   sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
   export LANG=en_US.UTF-8
   locale  # 確認
   ```

   期待する結果（`locale` の抜粋）: `LANG=` の行が `UTF-8` で終わっていればよい。WSLのUbuntuでは、最初から `C.UTF-8` になっていることが多い。

   ```text
   LANG=en_US.UTF-8
   LANGUAGE=
   LC_CTYPE="en_US.UTF-8"
   ...
   LC_ALL=en_US.UTF-8
   ```

2. universeリポジトリと、ROS2のaptソースの追加。

   ```bash
   sudo apt install software-properties-common
   sudo add-apt-repository universe

   sudo apt update && sudo apt install curl -y
   export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F'"' '{print $4}')
   curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
   ```

   ここで一度止まり、下の注記の確認をしてから、次のコマンドを実行する。

   ```bash
   dpkg -I /tmp/ros2-apt-source.deb   # 確認用（インストールはしない）
   sudo dpkg -i /tmp/ros2-apt-source.deb
   ```

   - この手順は、GitHubから取得した `.deb` を管理者権限でインストールし、以後のaptの取得元にROS2のリポジトリを追加する。取得元は公式（`ros-infrastructure/ros-apt-source`）だが、外部から取得したパッケージを導入する操作なので、URLが上記のとおりであることを確認してから実行する。
   - `sudo dpkg -i` の前に、`dpkg -I` の出力でパッケージ名・版・依存関係を目視確認する。これは内容の確認であり、改ざんを検知する整合性検証ではない。
   - `ROS_APT_SOURCE_VERSION` が空になる場合は、GitHub APIの未認証アクセスの回数制限に当たっている可能性がある。`echo "$ROS_APT_SOURCE_VERSION"` で確認し、しばらく待ってからやり直す。

   期待する結果（`dpkg -I` の抜粋。版の数字は取得した時期によって変わる）: `Package:` が `ros2-apt-source` で、`Description:` にROS 2のaptの取得元を設定するパッケージである旨が書かれていれば、意図したファイルである。

   ```text
    Package: ros2-apt-source
    Version: 1.x.x~noble
    Architecture: all
    ...
    Description: ...
   ```

3. 開発ツール（`ros-dev-tools`。colconを含む）の導入。

   ```bash
   sudo apt update && sudo apt install ros-dev-tools
   ```

   - `ros-dev-tools` は、`colcon` のほかに、C++のコンパイラ（`g++`）・`make`・`cmake` も依存として一緒に入れる（`ros-build-essential` → `build-essential` 経由）。C++のコンパイラを別途入れる必要はない。詳細は `docs/phase2_packages.md` の1-3節の補足を参照。

4. ROS2本体の導入。学習用には **Desktop Install**（rqt・turtlesim等を含む）を選ぶ。

   ```bash
   sudo apt update
   sudo apt upgrade
   sudo apt install ros-jazzy-desktop
   ```

   期待する結果: 導入するパッケージの一覧と容量が表示され、`Do you want to continue? [Y/n]` と聞かれるので `Y` で進める。数百のパッケージが入るため、回線によっては数十分かかる。最後にエラー（`E:` で始まる行）が出ずにプロンプトへ戻れば成功。`ls /opt/ros` を実行すると `jazzy` と表示される。

5. 環境の読み込み。

   ```bash
   source /opt/ros/jazzy/setup.bash
   ```

   - この行は `~/.bashrc` に追記する（新しいターミナルを開くたびに手動 `source` が必要だと、フェーズ1以降の手順書が前提とする「ROS2は読み込み済み」という前提が崩れ、ターミナルごとに挙動が食い違う原因になる）。追記内容は自分で確認したうえで行う。
   - **`~/.bashrc` 追記行（累積・現時点）**: 以後のフェーズ手順書で追加が必要になった場合、その手順書内で本行を含めた累積リストを記載する。

     ```bash
     source /opt/ros/jazzy/setup.bash   # 本手順で追加。ROS2本体の読み込み（常時必要、削除しない）
     ```

   - `source` は、成功しても何も表示しない。

### 5. 動作確認（Ubuntu内）

1. `printenv ROS_DISTRO` が `jazzy` を返す。
2. ターミナルA: `ros2 run demo_nodes_cpp talker`
3. ターミナルB: `ros2 run demo_nodes_py listener`
4. Bにメッセージが表示されることを確認し、A・Bとも Ctrl+C で終了する。期待する結果の例（時刻の数字は実行ごとに変わる）:

   ```text
   # ターミナルA（C++のtalker）
   [INFO] [1790072000.123456789] [talker]: Publishing: 'Hello World: 1'
   [INFO] [1790072001.123456789] [talker]: Publishing: 'Hello World: 2'

   # ターミナルB（Pythonのlistener）
   [INFO] [1790072001.124567890] [listener]: I heard: [Hello World: 2]
   ```

   Aは1秒ごとに番号を増やしながら送り、Bは同じ番号の文を受け取る。C++で書かれた送信側とPythonで書かれた受信側がつながっており、ROS2の通信が言語をまたいで動くことの最初の確認になる。

なお `rqt` や `turtlesim` などGUIアプリの表示（WSLg）は、フェーズ1の1-2節で確認する。この手順書の完了条件には含めない。

### 6. 作業ディレクトリを作る（Ubuntu内）

フェーズ1以降の手順書は、`~/work/ros2MinimalPhysicalAi` を作業の起点として書いている（フェーズ2で、この下に `ws/` を作る）。

```bash
mkdir -p ~/work/ros2MinimalPhysicalAi
cd ~/work/ros2MinimalPhysicalAi
pwd
```

期待する結果: `pwd` が `/home/<ユーザー名>/work/ros2MinimalPhysicalAi` と表示される。`mkdir` と `cd` は、成功しても何も表示しない。

- `/mnt/c`・`/mnt/d` の下（Windows側のドライブ）は使わない。WindowsとLinuxのファイルシステムをまたぐため、ビルドが大幅に遅くなる。
- この教材のリポジトリを手元に置いて読む場合は、フォルダを作る代わりに、`~/work` の下へ `git clone` してもよい（リポジトリが公開されていれば、HTTPSのURLで認証なしにclone できる。非公開の場合は7節の認証が要る）。フォルダ名が `ros2MinimalPhysicalAi` になるので、以降の手順書のパスはそのまま使える。

### 7.（任意）GitHubの認証・Claude Codeの環境（Ubuntu内）

**この節が必要な人**: 非公開のGitHubリポジトリをWSLの中でclone・pushする場合と、Claude CodeをWSLの中で使う場合だけ行う。フェーズ1以降の学習には不要。この節は、この教材の作者の運用（Claude Codeに手順書の作成やレビューを手伝わせる）に合わせて書いている。

1. **GitHubの認証**: 非公開のリポジトリは、認証していないWSLでは `git clone` が失敗する（`Repository not found` や `Authentication failed` となる）。WSLのgitはWindows側の設定・認証を引き継がないため、WSLの中で次のどちらかを設定する。**推奨はA（SSH鍵）**（追加のツールの導入が不要）。
   - **A. SSH鍵（推奨）**
     1. WSLの中で鍵を新しく作る（**パスフレーズの設定は必須**。この鍵はアカウント配下の全リポジトリにアクセスできるため、パスフレーズなしにしない）:
        ```bash
        ssh-keygen -t ed25519 -C "wsl-ubuntu" -f ~/.ssh/id_ed25519
        ```
     2. **公開鍵**（`.pub`）だけを表示してコピーする。秘密鍵（拡張子なしのファイル）は絶対に表示・共有しない:
        ```bash
        cat ~/.ssh/id_ed25519.pub
        ```
     3. ブラウザでGitHubにログインし、Settings → SSH and GPG keys → New SSH key に貼り付けて登録する（Titleは「WSL Ubuntu」など）。
     4. 接続確認（初回は接続先のフィンガープリントの確認が出る。GitHub公式ドキュメントの「GitHub's SSH key fingerprints」に記載のものと一致する場合だけ `yes`。一致しなければ `yes` とせず中断する）:
        ```bash
        ssh -T git@github.com
        ```
        `Hi <GitHubのユーザー名>! You've successfully authenticated...` と出れば成功。
     5. cloneはSSH形式のURLを使う（この節の2番目の手順のコマンドを参照）。
     - パスフレーズは、clone・pushのたびに入力を求められる。学習用途なら毎回入力で問題ない。手間なら `ssh-agent` に鍵を読み込ませて、セッション中の入力を省ける（`eval "$(ssh-agent -s)"` → `ssh-add ~/.ssh/id_ed25519`。ssh-agentはシェルを閉じると終了する）。
     - 鍵の権限範囲について: 対象のリポジトリだけに効く「Deploy key」という登録方法もある。同じアカウントでpushも行うなら、アカウント全体に効く鍵として扱う。
   - **B. GitHub CLI（`gh`）のブラウザ認証**
     - `gh` はUbuntu 24.04に標準では入っていない場合がある。導入する場合は公式の手順に従う。
     - `gh auth login` を実行し、GitHub.com → HTTPS → ブラウザ認証を選ぶ。表示されたワンタイムコードをブラウザで入力して承認する。
     - 完了後は `gh auth status` で確認できる。cloneはHTTPS形式のURLで行う。
     - **トークンの保管に注意**: WSLにはキーリングが無いことが多く、その場合トークンは `~/.config/gh/hosts.yml` に**平文で保存**されうる。既定のスコープも広め。不要になったら `gh auth logout` を実行し、GitHubのSettingsのアプリ/トークン一覧からも失効させる。
   - **アカウントは、対象のリポジトリにアクセスできるものを使う**。登録前に、ブラウザでログイン中のアカウントを確認する。別のアカウントだと `Repository not found` になる。認証情報（鍵・トークン）はWSL専用の別物で、Windows側とは共有されない。
   - パスワード・PAT（Personal Access Token）を、コマンドの履歴やファイルに残さない。Windows側の `.ssh`・資格情報は流用しない。
   - 鍵を登録するのはこのWSL専用。不要になったら、GitHubのSSH keysの画面から削除できる。
2. **リポジトリのclone**（A: SSHの場合はSSH形式、B: `gh` の場合はHTTPS形式）:
   ```bash
   cd ~/work
   git clone git@github.com:<GitHubのユーザー名>/ros2MinimalPhysicalAi.git      # A: SSH
   # git clone https://github.com/<GitHubのユーザー名>/ros2MinimalPhysicalAi.git  # B: gh認証済みの場合
   cd ros2MinimalPhysicalAi
   ```
   - Claude Codeにコミットさせる場合は、コミットの作者（author）を区別する規約を、リポジトリの `CLAUDE.md` に書いておく（この教材のリポジトリの例は `CLAUDE.md` の「Git運用」を参照）。committerのメールアドレスには、実際のメールアドレスではなく、GitHubが用意するnoreplyのアドレスを使う（リポジトリ単位の `git config user.email` で設定する）。実際のメールアドレスは、一度コミットに入ると履歴から消すのが難しい。
3. **Claude Codeの導入**: Anthropic公式のClaude Codeのドキュメントで最新のインストール手順を確認し、その手順で導入する（リンク先は変わりうるため、公式サイトから探す）。出所の分からないスクリプトは使わない。
4. **Claude Codeの設定を別のPCから引き継ぐ場合**: Claude Codeの設定ディレクトリ `~/.claude` は、Windows側から引き継がれない。自分の設定（グローバルのルール等）をGitHubのリポジトリで管理している場合は、ここへcloneする。
   - **Claude Codeを初めて起動する前に行う**（起動すると `~/.claude` が作られ、clone先が空でなくなる）。既に存在する場合は、中身を確認してから退避する（例: `mv ~/.claude ~/.claude.bak`）。
   ```bash
   git clone <設定のリポジトリのURL> ~/.claude
   ```
   - clone後、内容にWindows固有の記述（`settings.json` の `permissions` のパス、PowerShell前提のルール等）が残っていないか確認し、必要ならWSL用に調整する。
   - 認証情報・ブラウザのプロファイル等の機微なファイルが含まれていないことも確認する。あわせて `projects/`（メモリ・セッションの履歴）など、Windows側の作業履歴が含まれていないかも確認する。
5. cloneしたプロジェクトのディレクトリで `claude` を起動する。
6. **導入作業のレビューを頼む場合**: 実施した内容をClaude Codeなどにレビューしてもらうときは、次を貼り付ける。**認証情報・パスワードが含まれないことを確認してから**貼ること。
   - 実行したコマンド（`history` の該当部分など）
   - 各確認コマンドの出力
   - `~/.bashrc` に追記した場合は、追記した行

## つまずきやすい点

- 仮想化が無効だと `wsl --install` が失敗する → 1節のBIOS/UEFIの設定を確認する
- `/mnt/c`・`/mnt/d` の下で作業するとビルドが遅くなる → `~` の下で作業する（6節）
- `wsl` を起動した直後の `pwd` が `/mnt/c`・`/mnt/d` 等になる → Windows側の現在のフォルダを引き継いでいるだけ。`wsl --cd ~ -d Ubuntu-24.04` で起動する（3節の4番目の項目）
- ターミナルを開き直すと `ros2` が見つからない → `source /opt/ros/jazzy/setup.bash` が `~/.bashrc` に追記されていない（4節の5番目の手順）
- （2b節を行った場合）インポート後にrootでログインされる → `/etc/wsl.conf` の `[user] default=` が未設定（2b節の方法Aの2番目の手順）
- （7節を行う場合）非公開のリポジトリの `git clone` が認証エラー（`Repository not found` / `Authentication failed`）になる → WSLのgitはWindows側の認証を引き継がない。7節の1番目の手順で、SSH鍵または `gh` の認証を設定する。`ssh -T git@github.com` で認証の状態を切り分けられる
- （7節を行う場合）`~/.claude` へのcloneが「already exists and is not an empty directory」で失敗する → Claude Codeを先に起動して作られている。中身を確認して退避してからcloneする（7節の4番目の手順）

## 次へ

フェーズ1（`docs/phase1_cli_turtlesim.md`）へ進む。フェーズ1の1-2節で、この手順書では確かめなかったGUIの表示（WSLg）を確認し、turtlesimを動かしながら `ros2` コマンドでROS2の通信を観察する。ROS2の全体像を先に知りたい場合は、コマンドを使わない読み物のフェーズ0（`docs/phase0_overview.md`）を先に読んでもよい。

## 公式ドキュメント

- [ROS 2 Documentation: Jazzy — Installation](https://docs.ros.org/en/jazzy/Installation.html)（公式・英語）
- [ROS 2 Documentation: Jazzy](https://docs.ros.org/en/jazzy/)
- [colcon documentation](https://colcon.readthedocs.io/)
- WSLのインストール案内: https://aka.ms/wslinstall（`wsl` コマンドが案内するMicrosoft公式の短縮URL）
- 日本語の補助資料:
  - [ROS 2のインストール #ROS2 - Qiita](https://qiita.com/tomoswifty/items/a2afed1d23e8ea790c9c)
  - [Ubuntu24.04でのROS2セットアップ #ROS2 - Qiita](https://qiita.com/KimuraTomohiro/items/f3b75d204f7e962d5d21)
