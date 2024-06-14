# devcontainerの使い方

## コンテナ内で VSCode を起動する

1. .devcontainer/devcontainer.json を作成する

```json
{
  "name": "app", // 多分任意
  "dockerComposeFile": "../compose.yml",
  "service": "app", // compose.ymlのサービス名
  "workspaceFolder": "/usr/local/app", // マウントしたいコンテナ内のパス
  "customizations": {
    "vscode": {
      "extensions": [
        // extensionsは正式名称を調べる
        "ms-python.autopep8",
        "ms-python.python",
        "ms -python.flake8"
      ],
      "settings": {
        "flake8.args": ["--ignore=E111, E114, E402, E501"],
        "autopep8.args": ["--ignore=E114, E402, E501"],
        "[python]": {
          "editor.formatOnType": true,
          "editor.formatOnSave": true,
          "editor.defaultFormatter": "ms-python.autopep8"
        },
        "autoDocstring.guessTypes": false
      }
    }
  }
}
```

2. VSCode に Extensions，Dev Container を入れる。
3. コンテナを事前に立ち上げる。`docker compose up -d`
3. VSCode でコマンドパレットから`Reopen in container`

> [!NOTE]
> Extensions のアイコンから`Add to devcontainer.json`が使えそう。
> ![VSCode の画面](./fig/adddevcontainer.png)
