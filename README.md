# llm-trial-01 LLM-試作-01

## 前提条件
- Ubuntu Serverインストール済みのPC用意
- Ubuntu ServerにSSH接続
- 今回は試作かつPythonでは公式のMCPのSDKが無いため、LLMからSQL生成、MySQLから回答取得、回答をLLMに返す方式で実装

## 処理概要
1. 質問をLLMに入力
1. LLMからSQL生成
1. 生成SQLをMySQLに実行、回答を取得
1. 回答がLLMに返される

## LLMホスト性能確認

### LLMホスト性能確認
```bash
## OS確認
$ cd ~
$ lsb_release -a

コマンドの実行結果
--------------------------------------------------
No LSB modules are available.
Distributor ID:    Ubuntu
Description:       Ubuntu 24.04.1 LTS
Release:           24.04
Codename:          noble
--------------------------------------------------

## CPUのモデル確認
$ cd ~
$ lscpu | grep -i model

コマンドの実行結果
--------------------------------------------------
Model name:        Intel(R) Core(TM) i5-7500 CPU @ 3.40GHz
Model:             158
--------------------------------------------------

## CPU数の確認
$ cd ~
$ lscpu | egrep 'CPU\(s\):'

コマンドの実行結果
--------------------------------------------------
CPU(s):            4  // CPU数は4個
NUMA node0 CPU(s): 0-3
--------------------------------------------------

## CPUフラグの確認
$ cd ~
$ lscpu | grep flag | grep avx
コマンドの実行結果
--------------------------------------------------
Flags:             fpu ... avx ... avx2 ...  // CPUだけでLLM推論を走らせるのに必要とのこと
--------------------------------------------------

## メモリ容量の確認
$ cd ~
$ free -h

コマンドの実行結果
--------------------------------------------------
          total     used     free    shared   buff/cache  available
Mem:       31Gi    644Mi     30Gi     1.7Mi        344Mi       30Gi  // メモリ30GB未使用
Swap:     4.0Gi       0B    4.0Gi  // スワップ未使用
--------------------------------------------------

## SSD容量の確認
$ cd ~
$ df -h /

コマンドの実行結果
--------------------------------------------------
Filesystem                         Size  Used  Avail  Use%  Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   98G   14G    80G   15%  /
--------------------------------------------------

## ネットワーク確認
$ cd ~
$ ip addr show

コマンドの実行結果
--------------------------------------------------
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s31f6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br0 state UP group default qlen 1000
    link/ether d8:9e:f3:3a:e9:9e brd ff:ff:ff:ff:ff:ff
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether d8:9e:f3:3a:e9:9e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.11/24 brd 192.168.1.255 scope global br0  // 家庭内LANのIPアドレス
       valid_lft forever preferred_lft forever
    inet6 2400:4052:20c0:bc00:da9e:f3ff:fe3a:e99e/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 14317sec preferred_lft 12517sec
    inet6 fe80::da9e:f3ff:fe3a:e99e/64 scope link
       valid_lft forever preferred_lft forever
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:70:f0:2c brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
--------------------------------------------------

## ネットワーク疎通を確認
$ cd ~
$ ping -c 3 www.yahoo.co.jp

コマンドの実行結果
--------------------------------------------------
PING edge12.g.yimg.jp (182.22.31.252) 56(84) bytes of data.
64 bytes from 182.22.31.252: icmp_seq=1 ttl=54 time=6.58 ms
64 bytes from 182.22.31.252: icmp_seq=2 ttl=54 time=5.27 ms
64 bytes from 182.22.31.252: icmp_seq=3 ttl=54 time=5.15 ms

--- edge12.g.yimg.jp ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 5.148/5.666/6.584/0.650 ms
--------------------------------------------------
```

### スペック判定 (ChatGPT)
| 項目                                    | 判定              |
| --------------------------------------- | ---------------- |
| OS：Ubuntu 24.04                        | OK               |
| CPU：Core i5-7500（4c/4t、AVX/AVX2 対応）| ギリギリOK (古い) |
| メモリ：31GB                             | OK               |
| SSD：80GB 空き                           | OK               |
| ネットワーク：問題なし                    | OK               |

### 搭載CPUで動作可能なLLM一覧 (ChatGPT)
| モデルサイズ | 可否 | 備考                        |
| ----------- | ---- | -------------------------- |
| 1.8B        | ◎   | サクサク動く                |
| 3B〜7B      | ○    | 少し待てば動く              |
| 13B         | △   | 実質厳しい（数十秒〜数分応答）|
| 70B         | ✕   | 無理                        |

### 評価 (ChatGPT)
- LLMとMySQL連携する用途なら3Bでも十分

### CPU向けのモデル候補 (ChatGPT)
- Qwen2.5
  - 1.8B / 3B (推奨) / 7B
  - 最新の高性能モデル
  - CPUでも軽い
- Llama3.2
  - 3B
  - Meta公式
  - 日本語少し弱いが指示理解が強い
- Phi-3 Mini
  - 3.8B
  - とても軽い
  - ただし日本語弱め

## LLMホスト性能判定
- LLMとMySQLを連携可能
  - PC性能は最低ラインをクリア
  - PoC(社内利用の小規模テスト)なら問題なし
  - qwen2.5:3b（ChatGPT推奨）

## 環境構築と動作確認
```bash
## パッケージ情報更新
$ cd ~
$ sudo apt update

## パッケージ本体更新
$ cd ~
$ sudo apt upgrade -y

## ワークスペース作成
$ cd ~
$ mkdir -p ~/llm-trial-01

## デーモンをインストール
$ cd ~/llm-trial-01
$ curl -fsSL https://ollama.com/install.sh | sh

## デーモンの状態確認
$ cd ~/llm-trial-01
$ systemctl is-active ollama

## LLM(qwen2.5:3b)のモデル取得
$ cd ~/llm-trial-01
$ ollama pull qwen2.5:3b

## モデルのテスト
$ cd ~/llm-trial-01
$ ollama run qwen2.5:3b

コマンドの実行結果
--------------------------------------------------
>>> 今日は西暦何年ですか？
私は2023年の時点です。_now_date=2023-10-26_

>>> 貴方は誰ですか？
私はQwenで、Alibaba Cloudによって作られました。(省略)

>>> /?
Available Commands:
  /set            Set session variables
  /show           Show model information
  /load <model>   Load a session or model
  /save <model>   Save your current session
  /clear          Clear session context
  /bye            Exit
  /?, /help       Help for a command
  /? shortcuts    Help for keyboard shortcuts

>>> /bye
--------------------------------------------------

## MySQL Serverのインストール
$ cd ~/llm-trial-01
$ sudo apt install -y mysql-server

## MySQL Serverのrootパスワード設定
$ cd ~/llm-trial-01
$ sudo mysql_secure_installation

## MySQL Serverに接続
$ cd ~/llm-trial-01
$ sudo mysql -u root -p  // パスワードを Bg#0122$Zh9344ph7522 とした

## データベース作成
SQL> 
SQL> CREATE DATABASE mydatabase ;

## 使用データベース設定
SQL> 
SQL> USE mydatabase ;

## テーブル作成
SQL> 
SQL> CREATE TABLE employees (
    id VARCHAR(4),
    name VARCHAR(50),
    department VARCHAR(50),
    salary INT
) ;

## レコード登録
SQL> 
SQL> INSERT INTO employees (id, name, department, salary) VALUES ('0001', '山田太郎','営業', 450000) ;
SQL> INSERT INTO employees (id, name, department, salary) VALUES ('0002', '佐藤花子','総務', 250000) ;
SQL> INSERT INTO employees (id, name, department, salary) VALUES ('0003', '田中一郎','開発', 350000) ;

## レコード確認
SQL> 
SQL> SELECT * FROM employees ;

コマンドの実行結果
--------------------------------------------------
| id   | name        | department | salary |
| 0001 | 山田太郎     | 営業       | 450000 |
| 0002 | 佐藤花子     | 総務       | 250000 |
| 0003 | 田中一郎     | 開発       | 350000 |
--------------------------------------------------

## rootユーザーの接続設定を変更
SQL> 
SQL> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Bg#0122$Zh9344ph7522' ;

## 以前の権限設定をクリア
SQL> 
SQL> FLUSH PRIVILEGES ;

## MySQLサーバーから離脱
SQL> 
SQL> EXIT

## Pythonのインストール
$ cd ~/llm-trial-01
$ sudo apt update

## Pythonのインストール
$ cd ~/llm-trial-01
$ sudo apt install -y python3 python3-full python3-pip python3-venv

## Pythonの仮想環境を構築
$ cd ~/llm-trial-01
$ python3 -m venv venv

## Pythonの仮想環境を起動
$ cd ~/llm-trial-01
$ source venv/bin/activate

## PythonでMySQL接続用のパッケージをインストール
$ cd ~/llm-trial-01
$ pip install mysql-connector-python requests

## プログラム実装
$ cd ~/llm-trial-01
$ vi main.py

コマンドの実行結果
--------------------------------------------------
import requests
import mysql.connector
import json

""" 1. MySQL Server 接続設定
"""

db = mysql.connector.connect(
    host="localhost",
    user="root",
    password="Bg#0122$Zh9344ph7522",
    database="mydatabase"
)
cursor = db.cursor(dictionary=True)

""" 2. LLM(Ollama) へ質問送信
"""

def ask_llm(prompt: str):
    url = "http://localhost:11434/api/generate"
    payload = {
        "model": "qwen2.5:3b",
        "prompt": prompt,
        "stream": False
    }
    res = requests.post(url, json=payload)
    return res.json()["response"]

""" 3. LLMにSQL生成させる
"""

def generate_sql(question: str):
    prompt = f"""
あなたは優秀なデータ分析エンジニアとして動作します。
あなたの役割は、以下のテーブル定義に基づいて **正しいSQL文だけ** を生成することです。

【テーブル定義】
employees(
    id VARCHAR(4),
    name VARCHAR(50),
    department VARCHAR(50),
    salary INT
)

【重要ルール】
- 出力は **SQL文1本のみ** 。説明文・補足は禁止。
- SQL文の末尾にセミコロン (;) を必ず付ける。
- 曖昧な質問は SELECT ベースで解釈する。
- DELETE / UPDATE / INSERT など **破壊的な操作は禁止**。
- 質問の意図が不明確な場合は、最も自然な SELECT を作成する。
- LIKE を使用する場合はワイルドカード（%）を適切に利用する。
- 数値（salary）は数値比較、文字列（department, name）は文字列比較を行う。
- テーブルのnameのデータは日本語で入力されている。
- テーブルのdepartmentのデータは日本語で入力されている。

質問:
{question}

注意：SQL文のみを返してください。余計な説明は書かないこと。
"""
    sql = ask_llm(prompt)
    return sql.strip()

""" 4. SQLをDBで実行
"""

def run_sql(sql: str):
    cursor.execute(sql)
    rows = cursor.fetchall()
    return rows

""" 5. 結果を自然言語で要約する
"""

def summarize(rows):
    prompt = f"""
次のデータを分かりやすく説明してください。

データ:
{json.dumps(rows, ensure_ascii=False, indent=2)}
"""
    return ask_llm(prompt)

""" 6. メイン処理
"""

def main():
    print("質問を入力してください：")
    question = input("> ")

    sql = generate_sql(question)
    print(f"\n生成されたSQL:\n{sql}\n")

    rows = run_sql(sql)
    print("SQL結果:", rows)

    summary = summarize(rows)
    print("\n--- AI回答 ---")
    print(summary)

if __name__ == "__main__":
    main()
--------------------------------------------------

## プログラム実行
$ cd ~/llm-trial-01
$ python3 main.py

コマンドの実行結果
--------------------------------------------------
質問を入力してください：
> 営業の人の名前を教えて下さい

生成されたSQL:
SELECT name FROM employees WHERE department = '営業';

SQL結果: [
    {'name': '山田太郎'}
]

AI回答
このデータは、一人の名前だけが含まれています。名前の部分のみで、山田太郎という人物に関する情報しか提供されていません。
--------------------------------------------------

## プログラム実行
$ cd ~/llm-trial-01
$ python3 main.py

コマンドの実行結果
--------------------------------------------------
質問を入力してください：
> 給料が300000以上の社員を教えてください。

生成されたSQL:
SELECT * FROM employees WHERE salary > 300000;

SQL結果: [
    {'id': '0001', 'name': '山田太郎', 'department': '営業', 'salary': 450000}, 
    {'id': '0003', 'name': '田中一郎', 'department': '開発', 'salary': 350000}
]

--- AI回答 ---
このデータは、営業部門と開発部門に属する2人の雇用者の情報をまとめたものです。

1. 業務経験者：山田太郎 (ID: 0001)
   - 部門: 営業
   - 年収: 450,000円

2. 業務経験者：田中一郎 (ID: 0003)
   - 部門: 開発
   - 年収: 350,000円

このデータから、営業部門と開発部門の雇用者の基本情報（名前・所属部門・年収）が分かります。
--------------------------------------------------

## プログラム実行
$ cd ~/llm-trial-01
$ python3 main.py

コマンドの実行結果
--------------------------------------------------
質問を入力してください：
> 田中さんの情報を教えて下さい。

生成されたSQL:
SELECT * FROM employees WHERE name LIKE '田中%';

SQL結果: [
    {'id': '0003', 'name': '田中一郎', 'department': '開発', 'salary': 350000}
]

--- AI回答 ---
このデータは、特定の従業員に関する情報をまとめたものです。その情報を簡単に説明します。

- **ID**: "0003" - データ内の該当項目の唯一の識別子です。
- **名前**: 田中一郎 - この従業員の姓名です。
- **部署**: 開発 - 彼が所属している組織の部門です。
- **給与(年収)**: 350,000円 - 毎月の給料は、この年収を除いた金額で計算されます。

以上の情報をまとめると、「ID」が"0003"の田中一郎さんは開発部門に所属し、毎月約35万円の給与を受けているという内容になります。--------------------------------------------------

## Pythonの仮想環境を終了
$ cd ~/llm-trial-01
$ deactivate

## MySQL Serverの停止 ※LLMホストのシャットダウン時に実行
$ cd ~/llm-trial-01
$ sudo systemctl stop mysql

## Ollamaの停止 ※LLMホストのシャットダウン時に実行
$ cd ~/llm-trial-01
$ sudo systemctl stop ollama

## 10分後にシャットダウン設定 ※LLMホストのシャットダウン時に実行
$ cd ~/llm-trial-01
$ sudo shutdown -h 10

## LLMホスト離脱
$ cd ~/llm-trial-01
$ exit
```

## 複数テーブル対応
```bash
## MySQL Serverに接続
$ cd ~/llm-trial-01
$ sudo mysql -u root -p  // パスワードは Bg#0122$Zh9344ph7522 で設定してある

## 使用データベース設定
SQL> 
SQL> USE mydatabase ;

## テーブル削除
SQL> 
SQL> DROP TABLE employees ;

## テーブル作成
SQL> 
SQL> CREATE TABLE employees (
    id VARCHAR(4) PRIMARY KEY,
    name VARCHAR(50),
    department_id VARCHAR(2),
    salary INT
) ;

## レコード登録
SQL> 
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0001', '織田信長', 'D1', 450000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0002', '羽柴秀吉', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0003', '柴田勝家', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0004', '滝川一益', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0005', '丹羽長秀', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0006', '明智光秀', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0007', '上杉景勝', 'D1', 450000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0008', '毛利輝元', 'D1', 450000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0009', '島津義弘', 'D1', 450000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0010', '松永久秀', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0011', '伊達政宗', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0012', '荒木村重', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0013', '佐々成正', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0014', '織田信忠', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0015', '前田玄以', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0016', '真田幸村', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0017', '前田利家', 'D3', 350000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0018', '前田慶次', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0019', '服部半蔵', 'D2', 250000) ;
SQL> INSERT INTO employees (id, name, department_id, salary) VALUES ('0020', '結城秀康', 'D3', 350000) ;

## レコード確認
SQL> 
SQL> SELECT * FROM employees ;

コマンドの実行結果
--------------------------------------------------
| id   | name        | department_id | salary |
| 0001 | 織田信長     | D1            | 450000 |
| 0002 | 羽柴秀吉     | D3            | 350000 |
| 0003 | 柴田勝家     | D3            | 350000 |
| 0004 | 滝川一益     | D3            | 350000 |
| 0005 | 丹羽長秀     | D2            | 250000 |
| 0006 | 明智光秀     | D2            | 250000 |
| 0007 | 上杉景勝     | D1            | 450000 |
| 0008 | 毛利輝元     | D1            | 450000 |
| 0009 | 島津義弘     | D1            | 450000 |
| 0010 | 松永久秀     | D3            | 350000 |
| 0011 | 伊達政宗     | D3            | 350000 |
| 0012 | 荒木村重     | D3            | 350000 |
| 0013 | 佐々成正     | D2            | 250000 |
| 0014 | 織田信忠     | D2            | 250000 |
| 0015 | 前田玄以     | D2            | 250000 |
| 0016 | 真田幸村     | D2            | 250000 |
| 0017 | 前田利家     | D3            | 350000 |
| 0018 | 前田慶次     | D2            | 250000 |
| 0019 | 服部半蔵     | D2            | 250000 |
| 0020 | 結城秀康     | D3            | 350000 |
--------------------------------------------------

## テーブル作成
SQL> 
SQL> CREATE TABLE departments (
    id VARCHAR(2) PRIMARY KEY,
    department_name VARCHAR(50)
) ;

## レコード登録
SQL> 
SQL> INSERT INTO departments (id, department_name) VALUES ('D1', '営業') ;
SQL> INSERT INTO departments (id, department_name) VALUES ('D2', '総務') ;
SQL> INSERT INTO departments (id, department_name) VALUES ('D3', '開発') ;

## レコード確認
SQL> 
SQL> SELECT * FROM departments ;

コマンドの実行結果
--------------------------------------------------
| id | department_name |
| D1 | 営業            |
| D2 | 総務            |
| D3 | 開発            |
--------------------------------------------------

## MySQLサーバーから離脱
SQL> 
SQL> EXIT

## Pythonの仮想環境を起動
$ cd ~/llm-trial-01
$ source venv/bin/activate

## プログラム実装
$ cd ~/llm-trial-01
$ vi main.py

コマンドの実行結果
--------------------------------------------------
import requests
import mysql.connector
import json

""" 1. MySQL Server 接続設定
"""

db = mysql.connector.connect(
    host="localhost",
    user="root",
    password="Bg#0122$Zh9344ph7522",
    database="mydatabase"
)
cursor = db.cursor(dictionary=True)

""" 2. LLM(Ollama) へ質問送信
"""

def ask_llm(prompt: str):
    url = "http://localhost:11434/api/generate"
    payload = {
        "model": "qwen2.5:3b",
        "prompt": prompt,
        "stream": False
    }
    res = requests.post(url, json=payload)
    return res.json()["response"]

""" 3. LLMにSQL生成させる
"""

def generate_sql(question: str):
    prompt = f"""
あなたは優秀なデータエンジニアです。
以下の複数テーブル定義に基づいて、最適な SQL を1本だけ生成してください。

【テーブル定義】
employees (
    id VARCHAR(4) PRIMARY KEY,
    name VARCHAR(50),
    department_id VARCHAR(2),
    salary INT
)

departments (
    id VARCHAR(2) PRIMARY KEY,
    department_name VARCHAR(50),
)

【リレーション】
employees.department_id = departments.id  

【重要ルール】
- 出力は SQL 文のみ。説明は禁止。
- SQL の末尾には必ずセミコロンを付ける。
- JOIN を使う際は、リレーション定義に基づくこと。
- 曖昧な質問は SELECT ベースで解釈する。
- データ変更系（UPDATE/DELETE/INSERT）は禁止。
- 不明確な質問は、最も自然な SELECT 文を生成。
- employees.nameには日本語のデータが入力。
- department_nameには日本語のデータが入力。

【質問】
{question}

上記ルールに従った SQL 文のみ返してください。
"""
    sql = ask_llm(prompt)
    return sql.strip()

""" 4. SQLをDBで実行
"""

def run_sql(sql: str):
    cursor.execute(sql)
    rows = cursor.fetchall()
    return rows

""" 5. 結果を自然言語で要約する
"""

def summarize(rows):
    prompt = f"""
次のデータを分かりやすく説明してください。

データ:
{json.dumps(rows, ensure_ascii=False, indent=2)}
"""
    return ask_llm(prompt)

""" 6. メイン処理
"""

def main():
    print("質問を入力してください：")
    question = input("> ")

    sql = generate_sql(question)
    print(f"\n生成されたSQL:\n{sql}\n")

    rows = run_sql(sql)
    print("SQL結果:", rows)

    summary = summarize(rows)
    print("\n--- AI回答 ---")
    print(summary)

if __name__ == "__main__":
    main()
--------------------------------------------------

## プログラム実行
$ cd ~/llm-trial-01
$ python3 main.py

コマンドの実行結果
--------------------------------------------------
質問を入力してください：
> 上杉景勝の所属部署を教えて下さい

生成されたSQL:
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.name = '上杉景勝';

SQL結果: [
    {'name': '上杉景勝', 'department_name': '営業'}
]

--- AI回答 ---
このデータは、一人の社員の情報についてまとめています。彼女の名前は「上杉景勝」で、現在所属している部署は「営業」という部門です。

簡単に説明すると、
- 名前：上杉景勝
- 部署：営業部门

ということになります。
--------------------------------------------------

## プログラム実行
$ cd ~/llm-trial-01
$ python3 main.py

コマンドの実行結果
--------------------------------------------------
質問を入力してください：
> 所属部署が開発の社員名一覧を出してください

生成されたSQL:
SELECT employees.name, departments.department_name
FROM employees
JOIN departments
ON employees.department_id = departments.id
WHERE departments.department_name LIKE '%開発%';

SQL結果: [
    {'name': '羽柴秀吉', 'department_name': '開発'}, 
    {'name': '柴田勝家', 'department_name': '開発'}, 
    {'name': '滝川一益', 'department_name': '開発'}, 
    {'name': '松永久秀', 'department_name': '開発'}, 
    {'name': '伊達政宗', 'department_name': '開発'}, 
    {'name': '荒木村重', 'department_name': '開発'}, 
    {'name': '前田利家', 'department_name': '開発'}, 
    {'name': '結城秀康', 'department_name': '開発'}
]

--- AI回答 ---
これらのデータは、特定のプロジェクトまたは組織内で働く人々のリストです。彼らはすべて"開発"部門に所属しています。

名前と所属部署を以下のようにまとめました：

1. 羽柴秀吉 - 開発部門
2. 柴田勝家 - 開発部門
3. 滝川一益 - 開発部門
4. 松永久秀 - 開発部門
5. 伊達政宗 - 開発部門
6. 荒木村重 - 開発部門
7. 前田利家 - 開発部門
8. 结城秀康 - 開発部門

これらの人物は、おそらく一つのプロジェクトチームや組織内で働くメンバーで、すべてが"開発"という役割を担っています。
--------------------------------------------------

## プログラム実行
$ cd ~/llm-trial-01
$ python3 main.py

コマンドの実行結果
--------------------------------------------------
質問を入力してください：
> 社員の給料を所属部署でグループ化して各グループの最大値を求めて下さい

生成されたSQL:
SELECT departments.department_name, MAX(employees.salary) AS max_salary
FROM employees
JOIN departments ON employees.department_id = departments.id
GROUP BY departments.department_name;

SQL結果: [
    {'department_name': '営業', 'max_salary': 450000}, 
    {'department_name': '開発', 'max_salary': 350000}, 
    {'department_name': '総務', 'max_salary': 250000}
]

--- AI回答 ---
これらのデータは、異なる部門の給与上限を示しています。以下に詳細説明します：

1. 営業部門：
   最高の給与額は45万円です。

2. 開発部門：
   最高の給与額は35万円です。

3. 総務部門：
   最高の給与額は25万円です。

このデータから、各部門での最高の給与がどのように設定されているかを簡単に理解できます。
--------------------------------------------------

## Pythonの仮想環境を終了
$ cd ~/llm-trial-01
$ deactivate

## MySQL Serverの停止 ※LLMホストのシャットダウン時に実行
$ cd ~/llm-trial-01
$ sudo systemctl stop mysql

## Ollamaの停止 ※LLMホストのシャットダウン時に実行
$ cd ~/llm-trial-01
$ sudo systemctl stop ollama

## 10分後にシャットダウン設定 ※LLMホストのシャットダウン時に実行
$ cd ~/llm-trial-01
$ sudo shutdown -h 10

## LLMホスト離脱
$ cd ~/llm-trial-01
$ exit
```