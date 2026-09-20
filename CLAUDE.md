# ros2MinimalPhysicalAi

`brainstorming` リポジトリで育てたアイデア（`docs/idea_origin.md`）を実現するプロジェクト。フィジカルAI開発に必要な技術スタックを、ROS2最低限構成（WSL2 + Ubuntu 24.04 + Jazzy）で一通り身に着ける。

## Git運用

- コミットは `git-conventions` スキルに従う（author `ClaudeCode <noreply@anthropic.com>`、接頭辞 `[claude]`）。
- **`git push` は都度確認**（コミットとセットにしない）。GitHubリポジトリの作成・公開範囲の変更も実行前に承認を得る。
- リポジトリは当面 **private**。ただし **Public化を前提**に、次を常に意識する。
  - セキュリティ: 認証情報・APIキー・個人情報・PC固有の機微情報を含めない。push前に追跡ファイルの内容と `git log` のメタデータ（author名・email）を機密スキャンする。
  - 権利: 第三者の著作物（公式ドキュメントの転載、教科書の内容、サンプルコードの丸ごと転載、画像等）を含めない。参照はリンクと自分の言葉による要約に留める。ライセンス不明のものは含めない。
  - Public化の前には、上記を全履歴に対して精査し直す。
  - **committerの実メールアドレスが全コミットに残っている**（authorは`ClaudeCode`だが、committerはグローバルgit設定のまま。初回コミットはGitHubのprivateリポジトリへpush済み）。履歴の書き換えは禁止操作のため、Public化時の選択肢は「履歴なしで新規リポジトリに作り直す」か「許容する」のどちらかで、実施前にユーザーへ判断を仰ぐ。
  - `docs/`内の実測PCスペック（RAM・ディスク容量等）もPC固有情報に近いため、Public化前の精査対象に含める。
- 改行コードはLFに統一（`.gitattributes`）。

## 記録

- 判断・承認の経緯は `docs/qa_log.md` に追記する。
