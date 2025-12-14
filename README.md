# ホストに直接 Ansible/ansible-lint を入れて使う手順（Windows 前提、WSL でも同様）

## 1) 前提
- Python 3.9+ が入っていることを確認。ない場合は公式インストーラ、または WSL で `sudo apt-get install python3 python3-pip python3-venv` を実行。

## 2) インストール（推奨: venv）
- 仮想環境を作成・有効化:
  - Windows（PowerShell）:  
    `python -m venv .venv; .\.venv\Scripts\Activate.ps1`
  - WSL/Linux/macOS:  
    `python -m venv .venv && source .venv/bin/activate`
- Ansible と ansible-lint をインストール（venv 有効化後）:
  - `pip install --upgrade pip`
  - `pip install ansible ansible-lint`

## 3) 動作確認
- `ansible --version`
- `ansible-lint --version`

## 4) Lint の実行
- Playbook を lint:  
  `ansible-lint path/to/playbook.yml`
- 設定ファイルを使う場合: Playbook ルートに `./.ansible-lint` または `./.config/ansible-lint.yml` を置けば自動読み込み。

## 5) コレクション依存がある場合
- `requirements.yml` を用意し依存をローカルに導入:  
  `ansible-galaxy collection install -r requirements.yml -p ./collections`
- Lint は Playbook と同じディレクトリで実行。

## 補足
- グローバル導入したい場合は venv を省いて `pip install --user ansible ansible-lint` でも可（PATH 設定を確認）。
- Windows で `Activate.ps1` がブロックされたら、管理者 PowerShell で  
  `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` などを実行して許可。
