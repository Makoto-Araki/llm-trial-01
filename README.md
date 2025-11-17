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
$ sudo apt install -y python3 python3-pip python3-venv

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
あなたはSQL生成AIです。
次の質問に対して適切なSQLクエリだけを返してください。
テーブル: employees(id, name, department, salary)

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

SQL結果: [{'name': '山田太郎'}]

AI回答
このデータは、一人の名前だけが含まれています。名前の部分のみで、山田太郎という人物に関する情報しか提供されていません。
--------------------------------------------------

## Pythonの仮想環境を終了
$ cd ~/llm-trial-01
$ deactivate

## MySQL Serverの停止
$ cd ~/llm-trial-01
$ sudo systemctl stop mysql

## Ollamaの停止
$ cd ~/llm-trial-01
$ sudo systemctl stop ollama

## LLMホスト離脱
$ cd ~/llm-trial-01
$ exit
```

