
# AtCoder API

## 概要
目的の流れ:

1. コンテストを取得（一覧や特定コンテスト）
2. そのコンテストに参加したユーザーを取得（参加者一覧）
3. 各ユーザーの提出を取得
4. 必要に応じて提出ページへ移動（ブラウザで開く）

以下に、各ステップで利用する API（および代替手段）・指定方法（パラメータ）・取得できる情報（形式）をまとめます。

## コンテストの取得

- エンドポイント（例）:
	- `https://atcoder.jp/contests.json` （サイトにより公開されている JSON 一覧が存在する場合）
	- またはウェブページ `https://atcoder.jp/contests/` をスクレイピング

- 指定方法:
	- クエリパラメータはサイト実装依存。ページ取得→JSONパース、または HTML を解析。

- 取得情報:
  ```
  - `id` (string): コンテストID
  - `title` (string): コンテスト名
  - `start_epoch_second` (int): 開始時刻（UNIX タイムスタンプ）
  - `duration_second` (int): コンテスト時間（秒）
  - `rate_change` (string): レート変動対象区分
  - `is_rated` (boolean): レート対象か否か
  ```

- 形式: JSON 配列（あるいは HTML テーブル）

サンプル（簡易）:

```python
import requests
resp = requests.get("https://atcoder.jp/contests.json")
contests = resp.json()  # list
print(contests[0])
```

## コンテストに参加したユーザーの取得

AtCoder の公開 API に「参加者一覧」を直接返すエンドポイントがない場合があるため、代表的な取得方法は次の2つです。

- 方法 A: 提出一覧 API を走査して `user_id` を集める（API 利用）
	- エンドポイント: `GET https://atcoder.jp/api/v2/contests/{contestId}/submissions`
	- パラメータ例: `?user={user}`（特定ユーザーの提出を取得するためのオプション）
	- 取得情報（各提出要素）: `id`, `problem_id`, `user_id`, `language`, `point`, `result`, `epoch_second` など（JSON 配列）
	- 利点: API ベースで安定して取得可能。欠点: 提出がない参加者は検出できない（提出者のみ）。

	サンプル（ユーザー一覧の抽出）:
	```python
	resp = requests.get(f"https://atcoder.jp/api/v2/contests/{contest_id}/submissions")
	subs = resp.json()
	users = sorted({s['user_id'] for s in subs if s.get('user_id')})
	```

- 方法 B: スタンディング（順位表）ページをスクレイピング
	- URL: `https://atcoder.jp/contests/{contestId}/standings`
	- 取得情報: 順位、ユーザー名、合計得点など（HTML テーブル）
	- 利点: 提出の有無にかかわらず参加者を取得できる（コンテストによる）
	- 欠点: HTML 構造依存で変化に弱い

	サンプル（BeautifulSoup）:
	```python
	from bs4 import BeautifulSoup
	html = requests.get(f"https://atcoder.jp/contests/{contest_id}/standings").text
	soup = BeautifulSoup(html, "html.parser")
	# テーブルのユーザー名セルを探して抽出する（サイト構造に依存）
	users = [a.text.strip() for a in soup.select("table tr td a[href*='/users/']")]
	```

## 各ユーザーの提出取得

- エンドポイント: `GET https://atcoder.jp/api/v2/contests/{contestId}/submissions?user={userId}`
- 指定方法（パラメータ）:
	- `user` : ユーザー名（必須）
	- 他にページネーションや時間フィルタがある場合は API に依存（要確認）

- 取得情報（1 件あたり、JSON オブジェクト）:
  - `id` (int): 提出ID - 提出を一意に識別するための整数値。
    - `problem_id` (string): 問題ID - 提出された問題を識別するための文字列。
    - `user_id` (string): ユーザーID - 提出を行ったユーザーを識別するための文字列。
    - `language` (string): プログラミング言語 - 提出に使用されたプログラミング言語の名前。
    - `point` (float): 得点 - 提出に対して得られた得点（浮動小数点数）。
    - `result` (string): 提出結果 - 提出の結果を示す文字列（例: "AC" - 正解, "WA" - 不正解, "TLE" - 時間制限超過, "MLE" - メモリ制限超過, "CE" - コンパイルエラー）。
    - `epoch_second` (int): エポック秒 - 提出が行われた時刻を表す整数値（1970年1月1日からの秒数）。

サンプル:
```python
resp = requests.get(f"https://atcoder.jp/api/v2/contests/{contest_id}/submissions", params={"user": "username"})
resp.raise_for_status()
user_subs = resp.json()
```

## 提出ページに移動（ブラウザで確認）

- 提出ページ URL フォーマット:
	- `https://atcoder.jp/contests/{contestId}/submissions/{submissionId}`
- 確認方法（Python でブラウザを開く例）:
```python
import webbrowser
url = f"https://atcoder.jp/contests/{contest_id}/submissions/{submission_id}"
webbrowser.open(url)
```

## 実装例: フローをまとめた簡易スクリプト

```python
import requests
import time

contest_id = "abc001"
# 1) 提出一覧を取得して提出者を抽出
resp = requests.get(f"https://atcoder.jp/api/v2/contests/{contest_id}/submissions")
resp.raise_for_status()
subs = resp.json()
users = sorted({s['user_id'] for s in subs if s.get('user_id')})

for user in users:
		# 2) 各ユーザーの提出を取得
		r = requests.get(f"https://atcoder.jp/api/v2/contests/{contest_id}/submissions", params={"user": user})
		r.raise_for_status()
		user_subs = r.json()
		print(user, len(user_subs))
		time.sleep(0.5)  # 必要に応じて間隔を置く

		# 3) 任意の提出ページへ移動（例: 最初の提出）
		if user_subs:
				sid = user_subs[0]['id']
				print(f"https://atcoder.jp/contests/{contest_id}/submissions/{sid}")
```

## 注意点

- 提出のソースコードは API で取得できないことが多い（プライバシー）ので、ソースが必要な場合は提出ページへ遷移して確認する。
- アクセス頻度に注意（リクエスト間隔やページネーションの実装を検討する）。
