# 公開前レビューの観点

このリポジトリを公開（Public化）する前と、更新が続く間の定期的な点検で使う観点をまとめた。学習者向けの手順書ではなく、リポジトリを保守するための文書である。

- 対象: リポジトリが追跡しているすべてのファイル（`git ls-files`）と、全履歴（すべてのブランチ・コミット）
- 実施の目安: 手順書を数冊更新したとき、GitHubへpushする前、Public化の直前（直前は全履歴を対象にする）
- 観点は3つ: 1節「機密情報の流出」、2節「権利侵害」、3節「リンク先が意図どおりか」
- 各節は、コマンドで機械的に拾う「機械点検」と、読んで判断する「目視点検」に分けている。機械点検で何も出なくても、目視点検は省かない
- 結果は、末尾の4節「実施記録」に1行ずつ追記する。指摘と対応の判断は [`docs/qa_log.md`](qa_log.md) にも記録する

コマンドは、リポジトリの直下で実行する。どれもファイルを読むだけで、変更や外部への送信はしない（外部への通信を伴うのは3-2節の外部リンクの確認だけ）。

## 1. 機密情報の流出

### 1-1. 見る観点

| 観点 | 具体例 | 主な置き場所 |
|---|---|---|
| 認証情報 | APIキー、トークン、パスワード、秘密鍵 | 全ファイル・全履歴 |
| コミットの名義 | author・committerの名前とメールアドレスが、GitHubのnoreplyアドレス（`<数字>+<ユーザー名>@users.noreply.github.com`）になっているか | `git log` のメタデータ |
| 本文の個人情報 | 実名、実メールアドレス、GitHubのアカウント名（手順書では `<ユーザー名>` に伏せる。`LICENSE`・`README.md`・`CLAUDE.md` の著作権者の表示と、リポジトリのURLにあるアカウント名は意図して載せているので、対象外） | [`docs/qa_log.md`](qa_log.md) の過去の行、手順書 |
| PC固有の情報 | ユーザー名を含むパス（`/home/<名前>`、`C:\Users\<名前>`）、ドライブ構成、RAM・ディスク容量などの実測スペック（[`README.md`](../README.md) の「確認した環境」は、学習者が見比べるために意図して載せたもので対象外。ただし項目が増えていないかは見る） | `docs/` 全体、特に環境構築とフェーズ0 |
| PCの利用状況 | PCの用途など、個人の生活や資産を推測させる記述、スペックの制約、グローバルの安全ルールへの言及 | [`docs/idea_origin.md`](idea_origin.md)、[`CLAUDE.md`](../CLAUDE.md) |
| 追跡してはいけないもの | 個人メモ（`notes/`）、練習用のワークスペース（`ws/`）、`.env`・鍵ファイル、WSLのエクスポート | `.gitignore` と `git ls-files` |
| 画像の写り込み | スクリーンショットや写真に、画面上の個人情報・顔・書類が写っていないか（現在、追跡している画像は自作のSVGだけ） | `docs/img/` |

### 1-2. 機械点検

コミットの名義を一覧する（全ブランチ・全履歴）。

```bash
git log --all --format='%an <%ae> | %cn <%ce>' | sort | uniq -c
```

**見方**: 各行の左が件数、`|` の左がauthor、右がcommitter。ユーザー本人の行は、どちらもnoreplyアドレスになっていれば問題ない。Claudeの関与はコミットメッセージの `Co-Authored-By` トレーラーで示す規約なので、名義の欄に `noreply@anthropic.com` が出ることは無い想定。noreply以外のアドレスが1件でもあれば、該当コミットを調べる。

追跡中のファイルから、メールアドレスの形の文字列を拾い、想定済みのもの（noreply・ダミーの `example.com`）を除く。

```bash
git grep -nIE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' | grep -vE 'users\.noreply\.github\.com|noreply@anthropic\.com|@example\.(com|org)|git@github\.com|@gz\.msgs\.'
```

同じことを全履歴（過去の版の本文）に対して行う。

```bash
git log --all -p | grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' | grep -vE 'users\.noreply\.github\.com|noreply@anthropic\.com|@example\.(com|org)|git@github\.com|@gz\.msgs\.' | sort | uniq -c
```

認証情報らしい語と、ユーザー名を含むパスを拾う。

```bash
git grep -nIiE 'api[_-]?key|secret|token|passw(or)?d|BEGIN [A-Z ]*PRIVATE KEY|ghp_[A-Za-z0-9]|github_pat_|sk-[A-Za-z0-9]{10}'

git grep -nIE '/home/[a-z][a-z0-9_-]*/|[A-Za-z]:\\+Users\\+|/mnt/c/Users/'
```

追跡してはいけないものが入っていないかを見る。

```bash
git ls-files | grep -E '^(notes|ws)/|\.(env|pem|key|tar|vhdx)$'
```

**見方**: メールアドレスの2つのコマンドと、最後のコマンドは、何も表示されなければ問題ない。認証情報とパスのコマンドは、説明文の中の一般的な語（「パスワードマネージャー」、`/home/<ユーザー名>` のような伏せ字）にも当たるので、1件ずつ読んで、実際の値や実在のユーザー名が書かれていないかを判断する。GitHubのアカウント名は、上の正規表現では拾えないので、アカウント名そのもので `git grep -n` して、出てきた箇所を目視で判断する。`LICENSE`・`README.md`・`CLAUDE.md` の著作権者の表示と、リポジトリのURLの中のアカウント名は、意図して載せているものなので問題ない（[`docs/qa_log.md`](qa_log.md) の過去の行に残っている件は、棚卸しで扱いを決める方針）。

### 1-3. 目視点検

- [`docs/idea_origin.md`](idea_origin.md) と [`CLAUDE.md`](../CLAUDE.md) に、PCの利用状況やスペックの制約、グローバルの安全ルールへの言及が、公開してよい書き方で残っているか。
- `docs/` の実測のPCスペック（RAM・ディスク容量・ビルド時間など）が、公開してよい範囲か。
- [`docs/qa_log.md`](qa_log.md) に新しく足した行に、アカウント名・実メール・PC固有の情報・個人の練習環境の状態（`notes/` に書くべき内容）が入っていないか。
- 前回の実施以降に足した画像があれば、1枚ずつ開いて写り込みを確かめる。

## 2. 権利侵害

### 2-1. 見る観点

| 観点 | 具体例 | 主な置き場所 |
|---|---|---|
| 公式ドキュメントの抜粋 | 抜粋した箇所に、帰属表示（出典・ライセンス（CC BY 4.0）へのリンク・改変の有無）が残っているか | [`docs/setup_wsl2_ros2.md`](setup_wsl2_ros2.md) の手順4 |
| 各冊の出典注記 | 公式チュートリアルと同等のコマンド・API利用パターンを含む冊に、出典（元にしたものとそのライセンス。公式ドキュメントならCC BY 4.0）と「逐語の転載でない」旨の注記があるか | `docs/phase*.md`、[`docs/interlude_components.md`](interlude_components.md)、[`docs/tips.md`](tips.md)、[`docs/reference_node_class.md`](reference_node_class.md) |
| 同等の値の明記 | コマンド例の値が公式の例と同等である旨の記述が残っているか | [`docs/phase1_cli_turtlesim.md`](phase1_cli_turtlesim.md) |
| 引用の範囲 | 雛形の `CMakeLists.txt`（Apache 2.0）の各行や、setuptoolsの警告文の引用が、説明に必要な範囲に留まり、出典があるか | [`docs/tips.md`](tips.md) |
| ソースの解説 | `rclpy`・`rclcpp` のソースを、丸ごと写さず自分の言葉で解説しているか | [`docs/reference_node_class.md`](reference_node_class.md) |
| 図 | `docs/img/` のSVGが自作のもので、第三者の図・ロゴ・埋め込み画像が混ざっていないか | `docs/img/` |
| サンプルコード | 公式チュートリアルや他のリポジトリのコードを、丸ごと転載していないか | 各手順書のコードブロック |
| 第三者のファイル | データシート・PDF・フォント・画像など、ライセンスが不明なファイルが追跡されていないか | `git ls-files` |
| リポジトリのライセンス | `LICENSE` の「第三者の著作物」の一覧が最新か（前回以降に、公式ドキュメントの抜粋や、雛形・ツールの出力の引用を足していないか）。サンプルコードが使うパッケージに、コピーレフト（GPL等）のものが加わっていないか | [`LICENSE`](../LICENSE)、`README.md` のライセンスの節 |

### 2-2. 機械点検

追跡しているファイルのうち、Markdown・SVG・Gitの設定以外のものを一覧する。

```bash
git ls-files | grep -vE '\.(md|svg)$|^\.git(attributes|ignore)$|^LICENSE(-APACHE-2\.0)?$'
```

SVGに、外部の画像や外部へのリンクが埋め込まれていないかを見る。

```bash
git grep -nE '<image|href="https?:' -- 'docs/img/*.svg'
```

出典・出どころの注記が無い文書を一覧する。

```bash
grep -LE '出典|出どころ|CC BY' docs/phase*.md docs/interlude_components.md docs/tips.md docs/reference_node_class.md docs/setup_wsl2_ros2.md
```

**見方**: 1つ目と2つ目は、何も表示されなければ問題ない。3つ目で表示された文書は、公式ドキュメント・ソース・記事に由来する内容を含まないかを確かめ、含むなら注記を足す。元にしたものによってライセンスが違う（ROS2の公式ドキュメントはCC BY 4.0、ROS2のソースや雛形はApache 2.0など）ので、注記の有無だけでなく、文面（元にしたもの・ライセンス・逐語の転載でない旨）が合っているかは2-3節で目視する。

### 2-3. 目視点検

- 前回の実施以降に足した・大きく直した節で、公式ドキュメントや記事の文章をほぼそのまま写していないか（要約・自分の言葉での再構成になっているか）。
- 足したサンプルコードが、公式チュートリアルのコードと一致しすぎていないか（仕様・変数名・構成まで同じなら、書き直すか出典を明記する）。
- 2-1節の表の「公式ドキュメントの抜粋」「同等の値の明記」の記述が、文書の書き換えで消えていないか。

## 3. リンク先が意図どおりか

### 3-1. 見る観点

| 観点 | 具体例 |
|---|---|
| 文書間のリンク | 相対パスのリンク（`[...](phase2_packages.md)` や画像の `img/...svg`）の先のファイルが実在するか |
| 参照の書き方 | 他の文書への参照が、パスを `` ` `` で囲むだけでなくリンクになっているか（GitHubで読む前提のため） |
| 読む順の案内 | 各冊の終盤の「次へ」のリンクが、[`docs/learning_plan.md`](learning_plan.md) の「手順書一覧」の順と合っているか |
| 節の参照 | 本文の「N節」「N-M節の④」などの参照が、現在の見出しの番号と中身に合っているか（節の追加・繰り下げの後は特に） |
| 外部リンクの実在 | 外部のURLがリンク切れになっていないか |
| 外部リンクの中身 | リンク先がJazzy版に固定されているか、リンクの文言と中身が合っているか |
| ROS2の標準の型 | 型の定義へのリンクが、GitHubの `ros2` の公式リポジトリ（`common_interfaces` 等）の `jazzy` ブランチの定義ファイルを指しているか |

### 3-2. 機械点検

文書間のリンクの先が実在するかを確かめる（`README.md` と `docs/` の文書の、リンクの `(...)` の中の、`http` で始まらないものを対象にする。`CLAUDE.md` は読者向けの文書ではなく、書き方の例としてのリンクを含むので対象にしない）。

```bash
python3 - <<'PY'
import pathlib, re
bad = 0
for md in [pathlib.Path('README.md'), *sorted(pathlib.Path('docs').glob('*.md'))]:
    for n, line in enumerate(md.read_text(encoding='utf-8').splitlines(), 1):
        for target in re.findall(r'\]\(([^)\s]+)\)', line):
            if re.match(r'[a-z]+:', target):
                continue
            path = (md.parent / target.split('#')[0]).resolve()
            if not path.exists():
                bad += 1
                print(f'{md}:{n}: {target}')
print(f'リンク切れ: {bad}件')
PY
```

パスを `` ` `` で囲んだだけで、リンクになっていない参照を拾う。

```bash
grep -nE '(^|[^[])`docs/[^`*]+\.md`($|[^]])' docs/*.md README.md | grep -v '^docs/qa_log.md'
```

ROS2の標準の型へのリンクが、`jazzy` 以外のブランチを指していないかを見る。

```bash
grep -ohE 'https://github\.com/ros2/(common_interfaces|example_interfaces|rcl_interfaces)/blob/[^/]+' docs/*.md | sort | uniq -c
```

**見方**: 1つ目は `リンク切れ: 0件` だけが表示されれば問題ない。2つ目は、表示された行がリンクにすべき参照かを判断する（コードブロックの中や、ファイルを新しく作る指示の文は、リンクにしなくてよい）。3つ目は、どの行も `/blob/jazzy` で終わっていれば問題ない。

外部リンクの実在は、この環境から外部へアクセスして確かめる必要がある。アクセスする前に、対象のドメインの一覧（下のコマンド）を見て、見慣れないドメインが無いかを確かめる。

```bash
grep -ohE '\]\(https?://[^/)]+' docs/*.md README.md | sort | uniq -c | sort -rn
```

**見方**: 件数の多い順にドメインが並ぶ。docs.ros.orgはボット対策でコマンドやツールからの取得に失敗するため、Web検索で実在を確かめる。ROS2の標準の型のリンクは、同じパスを `raw.githubusercontent.com` で開いて200が返るかで確かめる。それ以外のドメインは、ブラウザで開くか、1件ずつ確かめる。

### 3-3. 目視点検

- 節を足した・繰り下げた・消した文書について、その文書と、その文書を参照している他の文書の「N節」を読み直す。
- 「次へ」のリンクを、[`docs/learning_plan.md`](learning_plan.md) の「手順書一覧」と並べて読む。
- 外部リンクのうち、前回の実施以降に足したものは、リンク先を開いて、リンクの文言・版（Jazzy）・内容が本文の説明と合っているかを確かめる。

## 4. 実施記録

| 実施日 | 対象（範囲） | 結果の要約 | 対応 |
|---|---|---|---|
| 2026-09-26 | 機械点検のみ（文書の作成時の試行。外部リンクと目視点検は未実施） | 名義はすべてnoreply。メールアドレス・認証情報・ユーザー名入りのパス・追跡してはいけないファイル・SVGの埋め込みは該当なし（認証情報の語の2件は、説明文の一般的な語）。リンク切れ0件、型のリンクはすべて `jazzy`。出典・出どころの注記が無いのは `docs/phase0_overview.md` だけ。`LICENSE` は未設置 | フェーズ0の注記の要否と `LICENSE` の扱いは、Public化前の棚卸しで決める |
| 2026-09-26 | 上の行の追記 | 上の行の記録の後、同日に `LICENSE` を設置した（サンプルコードとコマンドはApache-2.0、文章と図はCC BY 4.0）。あわせて、2-1節の表の「リポジトリのライセンス」の行を、設置後の点検の観点に改めた | 残りはフェーズ0の出典の注記の要否 |
