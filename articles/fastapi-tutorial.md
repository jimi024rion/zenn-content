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
    touch src/__init__.py src/database.py src/models.py
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

```python: models.py
from sqlalchemy import Column, Integer, String

from .database import Base

# Baseを継承することでORMモデルの作成できる
class User(Base):
    # テーブル名
    __tablename__ = "users"

    # カラム定義
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
    is_active = Column(Boolean, default=True)
```

## マイグレーション(Alembic)

Alembicで前ステップで定義したモデルのテーブルをマイグレーションする。

### Alembicとは？

Pythonのデータベースマイグレーションツール。
データベースのスキーマ変更を追跡し、バージョン管理ができる。
SQLAlchmeyと統合されており、簡単なマイグレーションスクリプトを生成できる。

### マイグレーション

[基本は公式Docのチュートリアル](https://alembic.sqlalchemy.org/en/latest/tutorial.html#tutorial)に沿って行えばOK

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
    + from src import models  # モデルの定義のために必要
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
    INFO  [alembic.ddl.postgresql] Detected sequence named 'users_id_seq' as owned by integer column 'users(id)', assuming SERIAL and omitting
    Generating /workspaces/app/migration/versions/6e9dbe72af7d_change_columns.py ...  done
    ```

    versions直下に定義したモデル情報でスクリプトが生成されてることを確認
    :::details migration/versions/直下の生成された中身

    ```python
    """init

    Revision ID: 7eb744555493
    Revises: 
    Create Date: 2024-04-30 16:02:57.275138

    """
    from typing import Sequence, Union

    from alembic import op
    import sqlalchemy as sa


    # revision identifiers, used by Alembic.
    revision: str = '7eb744555493'
    down_revision: Union[str, None] = None
    branch_labels: Union[str, Sequence[str], None] = None
    depends_on: Union[str, Sequence[str], None] = None


    def upgrade() -> None:
        # ### commands auto generated by Alembic - please adjust! ###
        op.add_column('users', sa.Column('hashed_password', sa.String(), nullable=True))
        op.add_column('users', sa.Column('is_active', sa.Boolean(), nullable=True))
        op.drop_index('ix_users_name', table_name='users')
        op.drop_column('users', 'name')
        # ### end Alembic commands ###


    def downgrade() -> None:
        # ### commands auto generated by Alembic - please adjust! ###
        op.add_column('users', sa.Column('name', sa.VARCHAR(), autoincrement=False, nullable=True))
        op.create_index('ix_users_name', 'users', ['name'], unique=True)
        op.drop_column('users', 'is_active')
        op.drop_column('users', 'hashed_password')
        # ### end Alembic commands ###
    ```

    :::

5. マイグレーション

    ```bash
    alembic upgrade head  # head: 最新のリビジョンでマイグレーション
    INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
    INFO  [alembic.runtime.migration] Will assume transactional DDL.
    INFO  [alembic.runtime.migration] Running upgrade  -> 7eb744555493, init
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
        Column      |       Type        | Collation | Nullable |              Default              
    -----------------+-------------------+-----------+----------+-----------------------------------
    id              | integer           |           | not null | nextval('users_id_seq'::regclass)
    email           | character varying |           |          | 
    hashed_password | character varying |           |          | 
    is_active       | boolean           |           |          | 
    Indexes:
        "users_pkey" PRIMARY KEY, btree (id)
        "ix_users_email" UNIQUE, btree (email)
    ```

定義したモデルのテーブルが生成できた。

## API実装

DBへのCRUD操作とAPIの処理を実装する。

### FastAPIとは？

APIを構築するための高速でモダンなWebフレームワーク。
ASGI（Asynchronous Server Gateway Interface）を使用しており非同期処理をサポート。また、型ヒントによるドキュメントの自動生成や、Pydanticとの統合による入力データのバリデーションやシリアライズが簡単に可能。

### Pydanticによる型定義

データの入出力時のPydanticモデルを定義する。

```bash
touch src/schmeas.py
```

```python: schamas.py
from pydantic import BaseModel

# 共通の属性を含むベースのモデル
class UserBase(BaseModel):
    email: str

# ユーザー作成リクエスト時のモデル
class UserCreate(UserBase):
    password: str

# ユーザー作成レスポンス時のモデル
class User(UserBase):
    id: int
    is_active: bool

    class Config:
        orm_mode = True
```

- **Pydanticモデルについて**
    BaseModelを継承したクラスがPydanticモデルとなる。
    更にそのクラスを継承した場合は子のクラスは親の属性も持つこととなる。(例えば、`UserCreate`クラスは、継承元の`email`も属性に持つ)
- **`orm_mode`について**
    Pydantic の `orm_mode` は Pydantic のモデルが dict ではなく ORM モデル（あるいは属性を持つ他の任意のオブジェクト）であってもデータを読み込むように指示する。
    これにより、PydanticモデルはORMと互換性があり、パス操作の`response_model`引数で宣言するだけでよくなる。
    データベースモデルを返すことができ、そこからデータを読み込んでくれる。

### CRUD実装

SQL AlchemyによるCRUD操作を実装する。

```bash
touch src/crud.py
```

```python: crud.py
from sqlalchemy.orm import Session  # dbパラメータの型。関数内で型チェックと補完を効かすために使う。
from . import models, schemas  # モデルとスキーマをインポート


def get_user(db: Session, user_id: int):
    """指定されたユーザーIDに対応するユーザーをデータベースから取得する関数"""
    return db.query(models.User).filter(models.User.id == user_id).first()


def get_user_by_email(db: Session, email: str):
    """指定されたメールアドレスに対応するユーザーをデータベースから取得する関数"""
    return db.query(models.User).filter(models.User.email == email).first()


def get_users(db: Session, skip: int = 0, limit: int = 100):
    """指定された範囲のユーザーをデータベースから取得する関数"""
    return db.query(models.User).offset(skip).limit(limit).all()


def create_user(db: Session, user: schemas.UserCreate):
    """新しいユーザーをデータベースに作成する関数"""
    fake_hashed_password = user.password + "notreallyhashed"  # パスワードをハッシュ化する（実際にはハッシュ化されない）
    db_user = models.User(email=user.email, hashed_password=fake_hashed_password)  # データベースモデルを作成
    db.add(db_user)  # データベースに追加
    db.commit()  # 変更をコミット
    db.refresh(db_user)  # データベースからユーザーをリフレッシュして返す
    return db_user  # 作成されたユーザーを返す


```

### FastAPIとSQLAlchemyの統合

## ローカル環境でのAPI動作確認

## おわりに

まとめ
参考文献
