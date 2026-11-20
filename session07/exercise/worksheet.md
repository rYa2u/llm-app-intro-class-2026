# 第7回 ワークシート：DB・APIを使うLLM

## 実行条件
- 実施日：2026年11月20日
- デモ / API：デモ、ローカル自動テスト、実API。デモは一時DBとFastAPIのTestClient、実APIは手元の架空の課題DBとask_assignments()で実行した。
- モデル（APIならgpt-6-luna）：gpt-6-luna、reasoning.effort=none。通信形式はSDKとhttpx.MockTransportによる模擬応答でも検証した。
- 変更したファイル：session07/exercise/assignment_tools.py、session07/exercise/worksheet.md、tests/test_course.py
- 使用データ：init_assignments()で登録する教材用の架空の課題3件。実在の学生の提出状況ではない。
- 検証の範囲：下記3質問は実APIで関数の選択・引数・取得行・回答内容を確認した。別の言い回しや多数の試行での安定性は未検証。

## 観察と比較
| 質問 | 方式・設定 | 結果 | 根拠・気づき |
|---|---|---|---|
| 未提出の課題を取得 | 関数を直接実行、status=pending | 2件：ID2「第5回 検索結果の比較」、ID3「第8回 評価シート」 | いずれもstatusはpending。 |
| 提出済みの課題を取得 | 関数を直接実行、status=done | 1件：ID1「第2回 API呼び出しレポート」 | statusはdone。 |
| すべての課題を取得 | 関数を直接実行、status=all | 3件：ID1・2・3 | ORDER BY idでID順に並ぶ。 |
| 未提出の課題を教えて | 課題DB・デモ | HTTP 200、ID2・3 | 固定のpendingを実行。入力・出力トークンは0。 |
| 提出済みの課題は？ | 課題DB・デモ | HTTP 200、ID2・3 | デモは質問に応じてdoneを選ばない。提出済みの結果が返らないことを確認した。 |
| すべての課題を一覧にして | 課題DB・デモ | HTTP 200、ID2・3 | デモではallも選ばれない。自然文からの関数選択の評価には使えない。 |
| delete_allを実行 | execute_tool('delete_all', '{}') | ValueError「許可していない関数です。」 | 実行前の関数名の検証で止まる。 |
| status=unknownを指定 | execute_tool('get_assignments', '{"status":"unknown"}') | ValueError「提出状況の指定が不正です。」 | 許可された3種類以外の値を拒否する。 |
| 余分なsql引数を指定 | status=allにsqlを追加 | ValueError「引数の形式が不正です。」 | モデルが提案したSQLを実行しない。 |

空の辞書、配列、数値・nullのstatus、SQLを含むstatus、不正なJSONも拒否されることをテストした。
拒否後もDBの全3件が元の内容のままであることを確認した。

## 後半実習：LLMとのやり取りと実行前の検証

2026年11月20日、実APIで次の3質問を実行した。各質問でget_assignmentsが1回選ばれ、引数のstatusと取得IDが期待値に一致した。回答文もDBのタイトルと提出状況に一致した。
入力・出力トークン数は、関数選択と回答生成の2回分を合計した実測値。

| 質問 | 選ばれた引数 | DB結果 | 回答の内容 | 入力トークン | 出力トークン |
|---|---|---|---|---|---|
| 未提出の課題を教えて | status=pending | ID2・3 | 未提出2件として「第5回 検索結果の比較」「第8回 評価シート」を列挙 | 309 | 55 |
| 提出済みの課題は？ | status=done | ID1 | 「第2回 API呼び出しレポート」が提出済みと回答 | 286 | 44 |
| すべての課題を一覧にして | status=all | ID1・2・3 | ID1は提出済み、ID2・3は未提出として全件を列挙 | 341 | 66 |

合計は入力936・出力165トークン。デモと異なり、質問に応じてstatusが切り替わることを確認できた。
第4回の既存.envからキーを検証プロセス内だけで読み込んだ。キーの値は記録せず、第7回への.envの複製や設定の永続的な変更も行っていない。
通常の実行環境ではAPIConnectionErrorになったが、ネットワーク制限のない実行で3質問とも成功した。

関数呼び出しの提案を実行し、call_idで対応する結果をLLMへ返す流れは[OpenAI公式Function Callingガイド](https://developers.openai.com/api/docs/guides/function-calling)で確認した。
実行前の検証はexecute_tool()が担当し、不正な関数・引数の拒否はローカルテストで確認した。実APIで不正な呼び出しを生成させる試験は行っていない。

## 講師スライド「実習2：get_tasks()のSQLを埋める」との対応

手元の教材にはprogress_db.pyとget_tasks()はなく、対応する実装はassignment_tools.pyのget_assignments()。
画像のtodoは、この教材ではpending（未提出）に当たる。SQLの穴埋めは実装済み。
2026年11月20日に手元のDBで次の4通りを実行して確認した。

| スライドの指定 | 手元で実行した呼び出し | 実行結果 |
|---|---|---|
| todo | get_assignments('pending') | 2件：ID2・3 |
| all | get_assignments('all') | 3件：ID1・2・3 |
| done | get_assignments('done') | 1件：ID1 |
| x | get_assignments('x') | ValueError「提出状況の指定が不正です。」 |

## コードの説明
- get_assignments()のSQLを復元し、仮のraiseを削除する。
- 自分が変更した行の意味：statusがallなら全件取得用のSELECT、それ以外ならWHERE status = ?で絞り込むSELECTを実行する。値は(status,)という1要素のタプルで別に渡す。SQL文字列と入力値を直接連結しない。仮のraiseを削除した。
- 検証：execute_tool()は関数名、JSONの形式、キーがstatusだけか、値が文字列かを調べる。get_assignments()は値がall・pending・doneのどれかかを調べてからDBへ接続する。
- データが通る場所（APIモード）：画面で課題DBを選択 → POST /api/ask → ask_assignments() → 1回目のAPIへTOOLSと質問を送る → LLMが関数名・引数を提案 → execute_tool()で検証 → Pythonのget_assignments()がSQLiteを検索 → call_id付きのfunction_call_outputを履歴へ追加 → 2回目のAPIが結果を文章にする → 回答とtool_resultsを画面へ返す。
- データが通る場所（デモ）：ask_assignments() → 固定のget_assignments(status=pending) → DB結果をJSONのまま表示する。LLMの関数選択や文章生成は行わない。
- 文書検索との違い：説明や手順は資料本文を検索する。現在の提出状況はDBのstatusから条件検索する。今回の画面では使う方式を利用者が選ぶ。
- 模擬APIの検証：関数呼び出し出力を履歴へ引き継ぎ、call_id=call_testの結果としてpendingの2件を返すことを確認した。2回目にtoolsを渡さず、2回分の入力トークン12+12=24、出力トークン6+6=12を合計することも確認した。この数値はテスト用の模擬値で、実利用量ではない。

## 確認問題への回答
1. 実際に関数を実行するのはPython。LLMは使いたい関数名と引数を提案し、Pythonが検証して実行する。
2. Python側の検証も必要。toolsの形式指定だけに頼らず、許可された関数名・引数のキー・型・値を実行前に確認する。
3. LLMが提案した関数呼び出しと、その呼び出しに対する実行結果を対応させる。
4. 1回目だけでは足りない。結果を使って回答を作る2回目も含め、実際に呼び出した分の入力・出力トークンを合計する。

## 失敗と改善
- 失敗した操作・入力：デモに「提出済みの課題は？」と質問しても未提出が返った。delete_allや不正なstatusを指定するとValueErrorで止まった。
- 原因の予想：前者はデモが固定でpendingを使う仕様。後者は実行可能な操作を読み取り関数と3種類のstatusに制限しているためで、想定どおりの拒否。
- 試した変更：自然文の選択処理とDB取得処理を切り分け、pending・done・allを直接指定して正しい件数を確認した。模擬APIで関数の提案から実行結果の受け渡しまでを検証し、その後、実APIでも3質問すべてで正しい選択と回答を確認した。
- 実装後の検証：既存の全体テストに第7回exerciseを追加し、未実装エラーを期待する対象から外した。第7回の件数・引数の拒否・DB内容の維持・デモの固定動作を検証した。
- まだ確かめていないこと：別の言い回しや曖昧な質問での関数選択の安定性、実APIが不正な呼び出しを提案した場合や、関数を選ばず回答した場合の品質。今回の3質問の結果だけで一般的な精度は判断しない。
