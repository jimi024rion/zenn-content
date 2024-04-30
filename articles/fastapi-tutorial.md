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
    alembic
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

:::details ローカルのシェルを使いたい場合
devcontainer.jsonでポートフォワードの設定の箇所のコメントアウトを外してあげればよい

```diff json: devcontainer.json
	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	// This can be used to network with other containers or the host.
+ 	 "forwardPorts": [5000, 5432],  # <= ここ
```

:::

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

以上で、DevContainer上でPythonとPostgreSQLの実行環境の構築が完了。

## データベース設定・モデル定義(SQLAlchemy)

SQLAlchemyによるテーブルのモデル定義と設定を行う。

### SQLAlchemyとは？

Pythonで人気のORM(Object Relational Mapper)。
ざっくりいうと、テーブルとオブジェクト(クラス)を対応させて、クラスのメソッド経由でデータのやりとりをするためのもの。

### データベース設定

1. モデル定義と設定を書くファイルを作成

    ```bash
    mkdir src
    touch src/__init__.py src/database.py src/schemas.py
    ```

2. DB設定

    ```python: database.py
    from sqlalchemy import create_engine
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy.orm import sessionmaker

    # DBの接続情報を指定
    # dialect+driver://username:password@host:port/database
    SQLALCHEMY_DATABASE_URL = "postgresql://postgres:postgres@localhost/postgres"

    # DBとの接続を確立
    engine = create_engine(SQLALCHEMY_DATABASE_URL)

    # DBセッションを生成するファクトリを作成
    SessionLocal = sessionmaker(
        autocommit=False,  # セッションの自動コミットを無効化
        autoflush=False,  # 自動フラッシュを無効化
        bind=engine  # セッションが使用するエンジンを指定
    )

    # ベースとなるORMモデルのクラスを定義
    Base = declarative_base()
    ```

    `SessionLocal`について
    このクラス自体はまだDBのセッションではないが、`SessionLocal`のインスタンスを作成すると実際のDBのセッションとなる。
    SQLAlchemy からインポートする `Session`と区別するために、`SessionLocal`と命名。

### データベースモデルの定義

```python: schemas.py
from sqlalchemy import Column, Integer, String

from .database import Base

# Baseを継承することでORMモデルの作成できる
class User(Base):
    # テーブル名
    __tablename__ = "users"

    # カラム定義
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)
    name = Column(String, unique=True, index=True)
```

## マイグレーション(Alembic)

Alembicで前ステップで定義したモデルのテーブルをマイグレーションする。

### Alembicとは？

Pythonのデータベースマイグレーションツール。
データベースのスキーマ変更を追跡し、バージョン管理ができる。
SQLAlchmeyと統合されており、簡単なマイグレーションスクリプトを生成できる。

### マイグレーション

基本は公式Docのチュートリアルに沿って行えばOK

1. 初期化

    ```bash
    alembic init migration
    ```

    以下ファイルが生成される
    生成されるファイルの詳細は [ここ(公式Doc)](https://alembic.sqlalchemy.org/en/latest/tutorial.html#the-migration-environment) にのっている。

    ```bash
    tree
    .
    ├── alembic.ini
    └── migration
        ├── README
        ├── env.py
        ├── script.py.mako
        └── versions
    ```

    重要なのは以下
    - **alembic.ini**
    alembicスクリプトが呼び出されたときに探すファイル。
    - **migration/env.py**
    マイグレーションツールが起動するたびに実行されるPythonスクリプト。少なくとも、SQLAlchemy エンジンを設定・生成し、トランザクショ ンとともにそのエンジンから接続を取得し、接続をデータベース接続のソースとして使って移行エン ジンを起動するための指示が含まれる。
    - **migration/versions**
    マイグレーションのバージョンごとのスクリプトが格納される。

2. iniファイルにDBのURLを設定

    ```diff ini
    - sqlalchemy.url = driver://user:pass@localhost/dbname
    + sqlalchemy.url = postgresql://postgres:postgres@localhost/postgres
    ```

3. env.pyにマイグレーションするモデルの情報を設定

    ```diff python: env.py
    from logging.config import fileConfig

    from sqlalchemy import engine_from_config
    from sqlalchemy import pool

    from alembic import context
    + from src import schemas  # モデルの定義のために必要
    + from src.database import Base, engine

    # this is the Alembic Config object, which provides
    # access to the values within the .ini file in use.
    config = context.config

    # Interpret the config file for Python logging.
    # This line sets up loggers basically.
    if config.config_file_name is not None:
        fileConfig(config.config_file_name)

    # add your model's MetaData object here
    # for 'autogenerate' support
    # from myapp import mymodel
    # target_metadata = mymodel.Base.metadata
    - target_metadata = None
    + target_metadata = Base.metadata

    # other values from the config, defined by the needs of env.py,
    # can be acquired:
    # my_important_option = config.get_main_option("my_important_option")
    # ... etc.


    def run_migrations_offline() -> None:
        """Run migrations in 'offline' mode.

        This configures the context with just a URL
        and not an Engine, though an Engine is acceptable
        here as well.  By skipping the Engine creation
        we don't even need a DBAPI to be available.

        Calls to context.execute() here emit the given string to the
        script output.

        """
        url = config.get_main_option("sqlalchemy.url")
        context.configure(
            url=url,
            target_metadata=target_metadata,
            literal_binds=True,
            dialect_opts={"paramstyle": "named"},
        )

        with context.begin_transaction():
            context.run_migrations()


    def run_migrations_online() -> None:
        """Run migrations in 'online' mode.

        In this scenario we need to create an Engine
        and associate a connection with the context.

        """
    +    url = config.get_main_option("sqlalchemy.url")
    +    connectable = engine

        with connectable.connect() as connection:
            context.configure(
    +            url=url,
                connection=connection,
                target_metadata=target_metadata
            )

            with context.begin_transaction():
                context.run_migrations()


    if context.is_offline_mode():
        run_migrations_offline()
    else:
        run_migrations_online()
    ```

4. マイグレーションスクリプトの生成

    ```bash
    alembic revision --autogenerate -m "init"
    INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
    INFO  [alembic.runtime.migration] Will assume transactional DDL.
    INFO  [alembic.autogenerate.compare] Detected added table 'users'
    INFO  [alembic.autogenerate.compare] Detected added index ''ix_users_email'' on '('email',)'
    INFO  [alembic.autogenerate.compare] Detected added index ''ix_users_name'' on '('name',)'
    Generating /workspaces/app/migration/versions/e5c17bc53035_init.py ...  done
    ```

    versions直下に定義したモデル情報でスクリプトが生成されてることを確認
    :::details migration/versions/直下の生成された中身

    ```python
    """init

    Revision ID: e5c17bc53035
    Revises: 
    Create Date: 2024-04-30 16:02:57.275138

    """
    from typing import Sequence, Union

    from alembic import op
    import sqlalchemy as sa


    # revision identifiers, used by Alembic.
    revision: str = 'e5c17bc53035'
    down_revision: Union[str, None] = None
    branch_labels: Union[str, Sequence[str], None] = None
    depends_on: Union[str, Sequence[str], None] = None


    def upgrade() -> None:
        # ### commands auto generated by Alembic - please adjust! ###
        op.create_table('users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('email', sa.String(), nullable=True),
        sa.Column('name', sa.String(), nullable=True),
        sa.PrimaryKeyConstraint('id')
        )
        op.create_index(op.f('ix_users_email'), 'users', ['email'], unique=True)
        op.create_index(op.f('ix_users_name'), 'users', ['name'], unique=True)
        # ### end Alembic commands ###


    def downgrade() -> None:
        # ### commands auto generated by Alembic - please adjust! ###
        op.drop_index(op.f('ix_users_name'), table_name='users')
        op.drop_index(op.f('ix_users_email'), table_name='users')
        op.drop_table('users')
        # ### end Alembic commands ###
    ```

    :::

5. マイグレーション

    ```bash
    alembic upgrade head  # head: 最新のリビジョンでマイグレーション
    INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
    INFO  [alembic.runtime.migration] Will assume transactional DDL.
    INFO  [alembic.runtime.migration] Running upgrade  -> e5c17bc53035, init
    ```

6. テーブル生成確認

    ```bash
    psql -h localhost -U postgres -d postgres
    psql (13.14 (Debian 13.14-1.pgdg110+2), server 16.2 (Debian 16.2-1.pgdg120+2))
    WARNING: psql major version 13, server major version 16.
            Some psql features might not work.
    Type "help" for help.

    postgres=# \d
                List of relations
    Schema |      Name       |   Type   |  Owner   
    --------+-----------------+----------+----------
    public | alembic_version | table    | postgres
    public | users           | table    | postgres
    public | users_id_seq    | sequence | postgres
    (3 rows)

    postgres=# \d users
                                    Table "public.users"
    Column |       Type        | Collation | Nullable |              Default              
    --------+-------------------+-----------+----------+-----------------------------------
    id     | integer           |           | not null | nextval('users_id_seq'::regclass)
    email  | character varying |           |          | 
    name   | character varying |           |          | 
    Indexes:
        "users_pkey" PRIMARY KEY, btree (id)
        "ix_users_email" UNIQUE, btree (email)
        "ix_users_name" UNIQUE, btree (name)
    ```

定義したモデルのテーブルが生成できた。

## API実装

FastAPIとは？
Pydanticによる型定義
CRUD実装
FastAPIとSQLAlchemyの統合

## ローカル環境でのAPI動作確認

## おわりに

まとめ
参考文献
