---
title: "FastAPIとSQLAlchemyを使ってPostgreSQLへCRUDしてみる"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["初心者", "fastapi", "sqlalchemy", "alembic", "postgres"]
published: false
---

## はじめに

ローカル環境でFastAPIとSQLAlchemyを使用してPostgreSQLへのCRUD操作を実際に試してみる。



:::message
実際に実装する際の流れについて理解することが目的です。
なので、各種ライブラリの詳細や使い方についてはほとんど触れません。
(ドキュメントにわかりやすく記載されているため)
:::

## 1. 開発環境セットアップ

1. dev containerによる開発環境のセットアップ
2. PostgreSQLへの接続確認


1. FastAPIの概要と基本的な使い方
FastAPIの紹介
FastAPIのインストールとプロジェクトのセットアップ
FastAPIでのAPIルーティングとリクエスト/レスポンスの処理
1. SQLAlchemyの概要と基本的な使い方
SQLAlchemyの紹介
SQLAlchemyのインストールと基本的な使い方
SQLAlchemyを使ったデータベースモデルの定義と操作
1. PostgreSQLの設定と接続
PostgreSQLの導入と設定
FastAPIとSQLAlchemyからPostgreSQLに接続する方法
1. FastAPIとSQLAlchemyの統合
FastAPIとSQLAlchemyを統合してAPIエンドポイントを作成する方法
ユーザーの作成、読み取り、更新、削除を行うCRUD操作の実装
1. Alembicを使ったデータベースマイグレーションの管理
Alembicの概要とインストール
データベースのスキーマ変更を行うマイグレーションファイルの作成と適用方法
1. ローカル環境での実装
実際のコード例を用いたCRUD操作の実装手順の解説
ローカル環境でのFastAPIアプリケーションの実行とテスト
1. おわりに
本記事のまとめと今後の学習について






## 2. SQL Alchemy(DBとの接続設定・CRUDの定義)


## 3. Alembic(マイグレーション)

## 4. FastAPI




1. SQL Alchemyによるテーブルのモデル定義
2. Alembicによるテーブルのマイグレーション
3. テーブル生成確認

SQL Alchemy & Alembic

