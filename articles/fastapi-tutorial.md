---
title: "FastAPIとSQLAlchemyを使ってPostgreSQLへCRUDしてみる"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["初心者", "fastapi", "sqlalchemy", "alembic", "postgres"]
published: false
---

## はじめに

ローカル環境でPythonとPostgreSQL実行環境を構築し、Alembicによるマイグレーション、FastAPIとSQLAlchemyを使用したCRUD操作を実際に試してみる。
:::message
実際に動かして流れを理解することが目的のため、各種ライブラリの詳細や使い方についてはあまり触れません。
:::

## 記事の対象者

以下をざっくり理解したい方向け

- DevContainerによるPython, PostgreSQLローカル実行環境構築
- FastAPIの使い方とローカル実行する流れ
- SQLAlchemyの設定とCRUDする流れ
- AlembicによるDBのマイグレーションする流れ
- FastAPIとSQLAlchemyを組み合わせて使用する際の構成

## 開発環境セットアップ

VSCodeの[DevContainer](https://containers.dev/)を用いてPythonとPostgreSQLが実行可能な環境をつくる。

### DevContainerとは？

VSCodeでDockerコンテナを使用して統一された開発環境を構築し、管理するための機能
https://containers.dev/

### 手順

1. 作業するディレクトリを作成し移動

    ```bash
    mkdir app && cd app
    ```

2. VSCode左下の`><`のマークを選択
3. 開発コンテナーセクションの「コンテナーで再度開く」を選択
4. 開発コンテナー構成ファイルの追加で「Python3 & PostgreSQL devcontainers」を選択
5. 機能の選択で「PostgreSQL Client」を選択
6. 以下のファイルが生成されDevContainerが立ち上がる

    ```bash
    .
    ├── .devcontainer
    │   ├── Dockerfile          # Pythonのコンテナ定義
    │   ├── devcontainer.json   # DevContainerの設定
    │   └── docker-compose.yml  # アプリとPostgreSQLのコンテナ定義
    └── .github
        └── dependabot.yml
    ```

7. 使用するPythonライブラリの設定ファイルを追加

    ```bash
    touch requirements.txt
    ```

    ```txt: requirements.txt
    fastapi
    psycopg2-binary
    pydantic
    sqlalchemy
    uvicorn[standard]
    ```

8. Dockerfileの以下箇所のコメントアウトを外す

    ```diff Dockerfile
    FROM mcr.microsoft.com/devcontainers/python:1-3.11-bullseye

    ENV PYTHONUNBUFFERED 1

    # [Optional] If your requirements rarely change, uncomment this section to add them to the image.
    + COPY requirements.txt /tmp/pip-tmp/
    + RUN pip3 --disable-pip-version-check --no-cache-dir install -r /tmp/pip-tmp/requirements.txt \
    +    && rm -rf /tmp/pip-tmp

    # [Optional] Uncomment this section to install additional OS packages.
    # RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    #     && apt-get -y install --no-install-recommends <your-package-list-here>
    ```

9. VSCode左下の`><`のマークを選択
10. 開発コンテナーセクションの「コンテナーのリビルド」を選択

### 確認

VSCodeのターミナル上で、PythonとPostgreSQLの起動・接続できることを確認

```bash
# Pythonの起動確認
python
Python 3.11.8 (main, Mar 12 2024, 12:00:36) [GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> quit()

# postgreSQL接続確認
psql -h localhost -U postgres -d postgres
psql (13.14 (Debian 13.14-1.pgdg110+2), server 16.2 (Debian 16.2-1.pgdg120+2))
WARNING: psql major version 13, server major version 16.
         Some psql features might not work.
Type "help" for help.

postgres=# \q
```

以上で、DevContainer上でPythonとPostgreSQLの実行環境が整った。

## データベースモデル定義と操作(SQLAlchemy)

SQLAlchemyとは？
データベース設定
データベースモデルの定義

## マイグレーション(Alembic)

Alembicとは？
マイグレーション

## FastAPI

FastAPIとは？
FastAPIとSQLAlchemyの統合

## ローカル環境でのAPI動作確認

## おわりに

まとめ
参考文献
