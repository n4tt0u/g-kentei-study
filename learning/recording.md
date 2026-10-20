# 個人記録と再開

追加ランタイムやフックを使わず、Codexが通常のファイル操作で保存する。自動同期サービスや会話全体の収集は行わない。

## 初期化

`learner/` がなければ作成し、存在しないものだけ `templates/state.json`、`templates/answers.jsonl`、`templates/preferences.md` を同名でコピーする。既存の記録は維持する。ひな型は共有用の初期値なので授業中に編集しない。

開始時にファイルの構文と `schema_version` を確認する。破損・未知の版なら自動リセットせず元を保持し、復旧が必要と伝える。複数の授業チャットで同じ記録へ同時に書かない。書込み前に再読取りし、変更があれば統合してから保存する。

## 出題時

1. 前の未回答問題がないか確認。あれば同じID・本文・選択肢を再提示できるようにする。
2. 次のID `G-0001` 等を割り当てる。IDは保存済みの問題と重複させない。
3. 問題ごとに `learner/questions/<ID>.json` を作る。必須項目は id、topic_id、concept、kind（initial/review）、original_question_id（復習元かnull）、stem、options（A〜D）、correct_option、explanations（A〜D）、sources、created_at。sourcesはURL・資料名・版・実際の確認日を含む。作成日時は取得可能な実時刻を使い、分からなければnullとする。
4. `state.json` の pending_question_id と next_question_number、last_explained_concept、status を更新してから問題を提示する。正解はファイルに保存するが回答前のチャットには出さない。これは厳密な試験用の解答秘匿機構ではない。

## 回答時

保存済み問題と本人の回答を照合し、採点イベントを `answers.jsonl` に1行のJSONで追記する。既存の行は変更しない。項目は event_id（重複しないID）、event_type（answer/correction/skip）、question_id、selected_option、correct_option、result（correct/incorrect/skipped/invalid）、reason（本人の申告かnull）、confidence（本人の申告かnull）、recorded_at、supersedes_event_id（訂正元かnull）、note。

同じ回答を再送された場合は同じイベントを二重追加しない。追記後、stateの未回答を解除し、current_topic_id・next_concept・review_candidates・status（awaiting_next等）を更新する。解答イベントの追記後に中断した場合は、次回にイベントと問題を読んでstateを復旧する。保存済みのイベントがある問題を新規回答として二重採点しない。

一つの概念の説明・初回確認を終えただけで単元全体を習得済みにしない。集計はイベント履歴から計算し、訂正元を二重に数えない。説明への質問やオモチの誤解を本人の誤答として記録しない。

## 終了・再開

休憩時は現在地を保存し、次回の候補を短く案内する。再開時はstateだけでなく対応する問題・回答イベントも確認する。未回答なら前の問題、採点済みなら次の小テーマや復習候補から本人の希望に応じ再開する。

学習ルールの版はstateのmethod_versionへ記録する。版が変わっても既存履歴を消さない。形式移行が必要なら原本を同じ非公開領域へバックアップし、明示的な移行を行う。単なる配布更新でtemplatesを既存learnerへコピーし直さない。

保存に失敗したらその旨を伝え、問題ID・回答・再開位置をチャットへ示す。保存完了や次回の確実な復元を装わない。
