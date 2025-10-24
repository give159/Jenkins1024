# PyCharm + Python + Jenkins 実践ガイド

## 📋 この記事でできるようになること

- PyCharmで開発したPythonプロジェクトをJenkinsで自動ビルド・テスト
- コードをGitHubにプッシュしたら自動でテスト実行
- テスト結果とカバレッジレポートの自動生成
- PyCharm上からJenkinsの状態を確認

## 🎯 前提条件

- PyCharm（Community版でもOK）がインストール済み
- Python 3.8以上がインストール済み
- Jenkinsが稼働している
- GitHubアカウントを持っている

---

## ステップ1: PyCharmでPythonプロジェクトを作成

### 1-1. 新規プロジェクトの作成

1. PyCharmを起動
2. 「New Project」をクリック
3. 以下を設定：
   - **Location**: `~/PycharmProjects/flask-jenkins-demo`
   - **Python Interpreter**: Python 3.8以上を選択
   - **Create a main.py welcome script**: チェックを外す
4. 「Create」をクリック

### 1-2. プロジェクト構造の作成

プロジェクトルートに以下のファイルを作成します：

**ファイル構造：**
```
flask-jenkins-demo/
├── app.py                 # メインアプリケーション
├── calculator.py          # テスト対象のモジュール
├── test_calculator.py     # ユニットテスト
├── requirements.txt       # 依存パッケージ
├── .gitignore            # Git除外設定
└── Jenkinsfile           # Jenkinsパイプライン定義
```

---

### 1-3. ソースコードの作成

**calculator.py** - 計算機クラス
```python
"""
簡単な計算機モジュール
"""

class Calculator:
    """基本的な計算機能を提供するクラス"""
    
    def add(self, a, b):
        """加算"""
        return a + b
    
    def subtract(self, a, b):
        """減算"""
        return a - b
    
    def multiply(self, a, b):
        """乗算"""
        return a * b
    
    def divide(self, a, b):
        """除算"""
        if b == 0:
            raise ValueError("0で割ることはできません")
        return a / b
    
    def power(self, base, exponent):
        """累乗"""
        return base ** exponent
```

**app.py** - Flaskアプリケーション
```python
"""
Flask Webアプリケーション
"""
from flask import Flask, jsonify, request
from calculator import Calculator

app = Flask(__name__)
calc = Calculator()

@app.route('/')
def home():
    return jsonify({
        "message": "Calculator API",
        "endpoints": {
            "/add": "GET ?a=1&b=2",
            "/subtract": "GET ?a=5&b=3",
            "/multiply": "GET ?a=4&b=3",
            "/divide": "GET ?a=10&b=2"
        }
    })

@app.route('/add')
def add():
    a = float(request.args.get('a', 0))
    b = float(request.args.get('b', 0))
    result = calc.add(a, b)
    return jsonify({"operation": "add", "result": result})

@app.route('/subtract')
def subtract():
    a = float(request.args.get('a', 0))
    b = float(request.args.get('b', 0))
    result = calc.subtract(a, b)
    return jsonify({"operation": "subtract", "result": result})

@app.route('/multiply')
def multiply():
    a = float(request.args.get('a', 1))
    b = float(request.args.get('b', 1))
    result = calc.multiply(a, b)
    return jsonify({"operation": "multiply", "result": result})

@app.route('/divide')
def divide():
    try:
        a = float(request.args.get('a', 0))
        b = float(request.args.get('b', 1))
        result = calc.divide(a, b)
        return jsonify({"operation": "divide", "result": result})
    except ValueError as e:
        return jsonify({"error": str(e)}), 400

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

**test_calculator.py** - ユニットテスト
```python
"""
Calculator クラスのユニットテスト
"""
import unittest
from calculator import Calculator

class TestCalculator(unittest.TestCase):
    """計算機のテストケース"""
    
    def setUp(self):
        """各テスト前の準備"""
        self.calc = Calculator()
    
    def test_add(self):
        """加算のテスト"""
        self.assertEqual(self.calc.add(2, 3), 5)
        self.assertEqual(self.calc.add(-1, 1), 0)
        self.assertEqual(self.calc.add(0, 0), 0)
    
    def test_subtract(self):
        """減算のテスト"""
        self.assertEqual(self.calc.subtract(5, 3), 2)
        self.assertEqual(self.calc.subtract(0, 5), -5)
        self.assertEqual(self.calc.subtract(10, 10), 0)
    
    def test_multiply(self):
        """乗算のテスト"""
        self.assertEqual(self.calc.multiply(3, 4), 12)
        self.assertEqual(self.calc.multiply(-2, 3), -6)
        self.assertEqual(self.calc.multiply(0, 100), 0)
    
    def test_divide(self):
        """除算のテスト"""
        self.assertEqual(self.calc.divide(10, 2), 5)
        self.assertEqual(self.calc.divide(9, 3), 3)
        self.assertAlmostEqual(self.calc.divide(7, 2), 3.5)
    
    def test_divide_by_zero(self):
        """0除算のテスト"""
        with self.assertRaises(ValueError):
            self.calc.divide(10, 0)
    
    def test_power(self):
        """累乗のテスト"""
        self.assertEqual(self.calc.power(2, 3), 8)
        self.assertEqual(self.calc.power(5, 0), 1)
        self.assertEqual(self.calc.power(10, 2), 100)

if __name__ == '__main__':
    unittest.main()
```

**requirements.txt** - 依存パッケージ
```
flask==3.0.0
pytest==7.4.3
pytest-cov==4.1.0
pylint==3.0.3
```

**.gitignore** - Git除外設定
```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/
ENV/

# PyCharm
.idea/

# テストとカバレッジ
.pytest_cache/
.coverage
htmlcov/
*.xml
*.log

# OS
.DS_Store
Thumbs.db
```

---

## ステップ2: PyCharmでの開発環境設定

### 2-1. 仮想環境の作成

1. PyCharmのターミナルを開く（下部の「Terminal」タブ）
2. 以下のコマンドを実行：

```bash
# 仮想環境の作成
python -m venv venv

# 仮想環境の有効化（Windows）
venv\Scripts\activate

# 仮想環境の有効化（Mac/Linux）
source venv/bin/activate

# パッケージのインストール
pip install -r requirements.txt
```

### 2-2. テストの実行設定

1. メニューバーから「Run」→「Edit Configurations...」
2. 「+」ボタン→「Python tests」→「pytest」を選択
3. 以下を設定：
   - **Name**: All Tests
   - **Target**: Script path
   - **Path**: プロジェクトルートを選択
   - **Python interpreter**: 作成したvenvを選択
4. 「Apply」→「OK」

### 2-3. テストの実行

1. 左側のプロジェクトツリーで `test_calculator.py` を右クリック
2. 「Run 'pytest in test_calculator.py'」を選択
3. 下部のテスト結果ウィンドウで成功を確認

---

## ステップ3: Gitリポジトリの初期化とGitHub連携

### 3-1. Gitリポジトリの初期化

PyCharmのターミナルで：

```bash
# Gitリポジトリの初期化
git init

# ファイルの追加
git add .

# コミット
git commit -m "Initial commit: Flask calculator app with tests"
```

### 3-2. GitHubリポジトリの作成と連携

1. GitHubで新しいリポジトリ「flask-jenkins-demo」を作成
2. PyCharmのターミナルで：

```bash
# リモートリポジトリの追加
git remote add origin https://github.com/your-username/flask-jenkins-demo.git

# ブランチ名をmainに変更
git branch -M main

# プッシュ
git push -u origin main
```

**PyCharmのGit統合を使う場合：**
1. メニューバー「VCS」→「Share Project on GitHub」
2. リポジトリ名を入力して「Share」

---

## ステップ4: Jenkinsfileの作成

プロジェクトルートに **Jenkinsfile** を作成：

```groovy
pipeline {
    agent any
    
    environment {
        PYTHON_VERSION = '3.9'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '=== ソースコードのチェックアウト ==='
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                echo '=== Python環境のセットアップ ==='
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('Lint') {
            steps {
                echo '=== コード品質チェック（Pylint） ==='
                sh '''
                    . venv/bin/activate
                    pylint calculator.py app.py || true
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo '=== ユニットテストの実行 ==='
                sh '''
                    . venv/bin/activate
                    pytest test_calculator.py -v --junitxml=test-results.xml
                '''
            }
        }
        
        stage('Coverage') {
            steps {
                echo '=== コードカバレッジの測定 ==='
                sh '''
                    . venv/bin/activate
                    pytest --cov=calculator --cov=app --cov-report=xml --cov-report=html
                '''
            }
        }
        
        stage('Archive') {
            steps {
                echo '=== テスト結果とレポートの保存 ==='
                junit 'test-results.xml'
                publishHTML(target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'htmlcov',
                    reportFiles: 'index.html',
                    reportName: 'Coverage Report'
                ])
            }
        }
    }
    
    post {
        success {
            echo '✅ ビルド成功！'
        }
        failure {
            echo '❌ ビルド失敗'
        }
        always {
            echo '=== クリーンアップ ==='
            cleanWs()
        }
    }
}
```

このJenkinsfileをコミット：

```bash
git add Jenkinsfile
git commit -m "Add Jenkinsfile for CI/CD pipeline"
git push origin main
```

---

## ステップ5: Jenkinsジョブの作成

### 5-1. パイプラインジョブの作成

1. Jenkinsダッシュボードで「新規ジョブ作成」
2. ジョブ名を入力：`flask-jenkins-demo`
3. 「パイプライン」を選択
4. 「OK」をクリック

### 5-2. ジョブの設定

**General セクション：**
- 「GitHub project」にチェック
- Project url: `https://github.com/your-username/flask-jenkins-demo`

**Build Triggers セクション：**
- 「GitHub hook trigger for GITScm polling」にチェック

**Pipeline セクション：**
- **Definition**: Pipeline script from SCM
- **SCM**: Git
- **Repository URL**: `https://github.com/your-username/flask-jenkins-demo.git`
- **Credentials**: 必要に応じて追加
- **Branch Specifier**: `*/main`
- **Script Path**: `Jenkinsfile`

「保存」をクリック

### 5-3. 必要なプラグインのインストール

1. 「Jenkinsの管理」→「プラグインの管理」
2. 以下をインストール：
   - **HTML Publisher Plugin** - カバレッジレポート表示用
   - **JUnit Plugin** - テスト結果表示用
   - **Pipeline Plugin** - パイプライン実行用

---

## ステップ6: PyCharm Jenkins プラグインの設定

### 6-1. Jenkins Pluginのインストール

1. PyCharmで「File」→「Settings」（Mac: 「PyCharm」→「Preferences」）
2. 「Plugins」を選択
3. 検索バーで「Jenkins」と入力
4. 「Jenkins Control Plugin」をインストール
5. PyCharmを再起動

### 6-2. Jenkins接続の設定

1. 「Settings」→「Tools」→「Jenkins」
2. 以下を設定：
   - **Jenkins URL**: `http://localhost:8080`（Jenkinsのアドレス）
   - **Username**: Jenkinsのユーザー名
   - **Password/Token**: APIトークン（後述）
3. 「Test Connection」で接続確認
4. 「OK」をクリック

### 6-3. Jenkins APIトークンの取得

1. Jenkinsにログイン
2. 右上のユーザー名→「設定」
3. 「API Token」セクション→「Add new Token」
4. トークン名を入力（例: PyCharm）
5. 「生成」をクリック
6. **生成されたトークンをコピー**（再表示不可）
7. PyCharmの設定でパスワード欄に貼り付け

---

## ステップ7: 動作確認

### 7-1. 手動ビルドのテスト

1. Jenkinsのジョブページで「ビルド実行」
2. ビルド進行状況を確認
3. 「Console Output」でログを確認
4. 完了後、以下を確認：
   - テスト結果レポート
   - カバレッジレポート

### 7-2. PyCharmからの確認

1. PyCharmの下部ツールバーから「Jenkins」タブを開く
2. ジョブ一覧から「flask-jenkins-demo」を確認
3. ビルド状況（緑=成功、赤=失敗）を確認

### 7-3. 自動ビルドのテスト

PyCharmでコードを修正：

**calculator.py に新しいメソッドを追加：**
```python
def square(self, n):
    """平方を計算"""
    return n * n
```

**test_calculator.py に対応するテストを追加：**
```python
def test_square(self):
    """平方のテスト"""
    self.assertEqual(self.calc.square(5), 25)
    self.assertEqual(self.calc.square(0), 0)
    self.assertEqual(self.calc.square(-3), 9)
```

コミット＆プッシュ：

```bash
git add .
git commit -m "Add square method with tests"
git push origin main
```

数秒後、Jenkinsが自動的にビルドを開始します！

---

## ステップ8: カバレッジレポートの確認

### 8-1. Jenkinsでのカバレッジ確認

1. ビルドが完了したら、ジョブページの左側メニューから「Coverage Report」をクリック
2. 各ファイルのカバレッジ率を確認
3. ファイル名をクリックすると、行ごとの実行状況が表示される

### 8-2. ローカルでのカバレッジ確認

PyCharmのターミナルで：

```bash
# カバレッジレポートの生成
pytest --cov=calculator --cov=app --cov-report=html

# HTMLレポートを開く（ブラウザで表示）
# Windows
start htmlcov/index.html

# Mac
open htmlcov/index.html

# Linux
xdg-open htmlcov/index.html
```

---

## ステップ9: コード品質の自動チェック

### 9-1. Pylintの設定ファイル作成

プロジェクトルートに **.pylintrc** を作成：

```ini
[MASTER]
disable=
    C0111,  # missing-docstring
    R0903,  # too-few-public-methods

[FORMAT]
max-line-length=100

[MESSAGES CONTROL]
confidence=HIGH
```

### 9-2. PyCharmでのPylint統合

1. 「Settings」→「Tools」→「External Tools」
2. 「+」をクリック
3. 以下を設定：
   - **Name**: Pylint
   - **Program**: `$PyInterpreterDirectory$/pylint`
   - **Arguments**: `$FilePath$`
   - **Working directory**: `$ProjectFileDir$`
4. 「OK」をクリック

使用方法：
- ファイルを右クリック→「External Tools」→「Pylint」

---

## ステップ10: デバッグとトラブルシューティング

### 10-1. よくあるエラーと解決法

**エラー1: "pytest: command not found"**

```bash
# 解決法：仮想環境の有効化を確認
source venv/bin/activate  # Mac/Linux
venv\Scripts\activate     # Windows

# または、フルパスで実行
venv/bin/pytest
```

**エラー2: "ModuleNotFoundError: No module named 'flask'"**

```bash
# 解決法：requirements.txtから再インストール
pip install -r requirements.txt
```

**エラー3: Jenkins ビルドが "No such file or directory" で失敗**

Jenkinsfileを修正：
```groovy
sh '''
    #!/bin/bash
    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt
'''
```

### 10-2. PyCharmデバッガーの使用

1. テストファイルで行番号の左をクリックしてブレークポイント設定
2. テストを右クリック→「Debug 'pytest in test_calculator.py'」
3. デバッグコンソールで変数を確認

---

## 📊 実践課題

### 課題1: 新機能の追加

以下の機能を追加して、テスト→Jenkins自動ビルドまで実施：

1. `calculator.py`に`factorial(n)`メソッドを追加（階乗計算）
2. 対応するテストケースを3つ以上作成
3. コミット＆プッシュで自動ビルドを確認

**ヒント：**
```python
def factorial(self, n):
    """階乗を計算"""
    if n < 0:
        raise ValueError("負の数の階乗は定義されていません")
    if n == 0 or n == 1:
        return 1
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

### 課題2: APIエンドポイントの追加

1. `app.py`に新しいエンドポイント`/square`を追加
2. FlaskのテストケースをPyTestで作成
3. カバレッジ80%以上を目指す

### 課題3: Slack通知の追加

前回の記事を参考に、Jenkinsfileに以下を追加：

```groovy
post {
    success {
        echo '✅ ビルド成功！'
        // Slack通知を追加
    }
    failure {
        echo '❌ ビルド失敗'
        // Slack通知を追加
    }
}
```

---

## 🎓 さらに学ぶために

### PyCharm便利機能

1. **Gitコミット履歴**: `Alt+9`（Mac: `⌘+9`）
2. **最近のファイル**: `Ctrl+E`（Mac: `⌘+E`）
3. **実行**: `Shift+F10`（Mac: `⌃+R`）
4. **デバッグ**: `Shift+F9`（Mac: `⌃+D`）
5. **ターミナル**: `Alt+F12`（Mac: `⌥+F12`）

### pytest高度な使い方

```bash
# 特定のテストのみ実行
pytest test_calculator.py::TestCalculator::test_add

# 失敗したテストのみ再実行
pytest --lf

# デバッグ情報付きで実行
pytest -vv

# 並列実行（高速化）
pip install pytest-xdist
pytest -n 4  # 4プロセスで並列実行
```

---

## 📚 まとめ

これで以下のワークフローが完成しました：

```
PyCharmで開発
    ↓
ローカルでテスト（pytest）
    ↓
Gitコミット＆プッシュ
    ↓
Jenkins自動ビルド
    ↓
テスト結果・カバレッジレポート
    ↓
PyCharmで結果確認
```

**ポイント：**
- ✅ PyCharmとJenkinsのシームレスな連携
- ✅ 自動テストとカバレッジ測定
- ✅ コード品質の継続的なチェック
- ✅ チーム開発に即応用可能

この環境があれば、高品質なPythonアプリケーション開発ができます！
