# 静的解析 uv+rye monorepo構成

## ryeインストール・アップデート

すでにインストール済みでもuvをonにするために実行してください。
uvをonにするだけなら `rye config ...` でもできます。

```console
$ curl -sSf https://rye-up.com/get | bash

? Select the preferred package installer ›
  pip-tools (slow but stable)
❯ uv (quick but experimental)

# uvを選んでください

? What should running `python` or `python3` do when you are not inside a Rye managed project? ›
❯ Run a Python installed and managed by Rye
  Run the old default Python (provided by your OS, pyenv, etc.)

# managed by Ryeを選んでください
```

## multi project workspace

複数のプロジェクトをmonorepoで管理したいのでryeのworkspace機能を使います。

https://rye-up.com/guide/workspaces/#declaring-workspaces

- ワークスペース内のプロジェクトで仮想環境が同期されます
- 静的解析の設定も楽になります
- 各プロジェクトはeditableなmoduleとしてvenvにinstallされます

### root settings

ルートのプロジェクトを作成します。

ここにworkspaceの設定を入れています。
デフォルトではpyproject.tomlがあるディレクトリ全てが認識されるようです。ここでは `./projects/*` に制限しています。

```console
$ rye init root
$ cd root
$ echo -e '\n\n[tool.rye.workspace]\nmembers = ["projects/*"]' >> pyproject.toml
```

### create new project

サブプロジェクトを作成します。
サブプロジェクトの `.python-version` は参照されないので削除します。
`rye sync` でサブプロジェクトがeditableなモジュールとして自動で追加されます。
他のプロジェクトからは `import <project.name>` （pyprojectで設定した名前）でインポートできます。

```console
$ rye init projects/project-a
$ (cd projects/project-a && rm -rf .git .python-version .gitignore)

$ rye init projects/project-b
$ (cd projects/project-b && rm -rf .git .python-version .gitignore)

$ rye sync
```

### run

`rye run python -m <project.name>` で `<root>/src/<project.name>/__main__.py` が実行されます。
working directoryはroot直下でなくても大丈夫です。

```console
$ echo -e 'print("run!")' > ./projects/project-a/src/project_a/__main__.py
$ rye run python -m project_a
run!
```

別プロジェクトからのインポート

```console
$ echo -e 'export_var = "Exported!"' > ./projects/project-b/src/project_b/lib.py
$ echo -e 'from project_b.lib import export_var\n\nprint(f"This value is {export_var}")' > ./src/root/__main__.py
$ rye run python -m root
This value is Exported!
```

### install dev dependencies

```console
$ rye add --dev ruff mypy
$ rye sync
```

```console
# 全体をcheck

$ rye lint
$ rye fmt --check
$ rye fmt
$ rye run mypy .

# サブプロジェクトのみを静的解析

$ rye lint projects/project-a
$ rye fmt --check projects/project-a
$ rye fmt projects/project-a
$ rye run mypy projects/project-a

# またはプロジェクトディレクトリから

$ cd projects/project-a
$ rye lint
$ rye fmt --check
$ rye fmt
$ rye run mypy .
```

`rye lint` は `rye run ruff check` です。

## ruffの設定

デフォルトだと厳しすぎるので適宜ignoreでルールを無視する

```toml
# root/pyproject.toml
[tool.ruff.lint]
select = [
    # pycodestyle
    "E",
    # Pyflakes
    "F",
    # pyupgrade
    "UP",
    # flake8-bugbear
    "B",
    # flake8-simplify
    "SIM",
    # isort
    "I",
]
ignore = ["F841"]
```

- project側に `tool.ruff.lint` があればproject内のコードはそちらが優先される
- 基本はrootでruffやmypyの設定をして、どうしても個別のプロジェクトで設定を変えたければそちらに書く
- mergeはされなさそう

## lambda

## docker

### 疑問点

- lockファイルがrootにしかないから1プロジェクトだけdocker buildとかはできるのか？
  - sync遅かったりしないか？
  - 不要なモジュールが追加されたりしないか？

