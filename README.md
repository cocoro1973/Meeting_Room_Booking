<div align="center">

# 🏢 簡単会議室予約システム (Meeting Room Booking System)
**Javascript / Node.js / Express.js / PostgreSQL で動作するスマート＆シンプルな会議室予約管理プラットフォーム**

![Node.js](https://img.shields.io/badge/Node.js-v18%2B-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Express.js](https://img.shields.io/badge/Express.js-v4.x-000000?style=for-the-badge&logo=express&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-v16%2B-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)

[概要](#-概要) • [主な機能](#-主な機能) • [アーキテクチャ](#-システムアーキテクチャ) • [PostgreSQLの設定](#-postgresqlの設定) • [セットアップ](#-セットアップ手順) • [技術的なこだわり](#-技術的なこだわり)

---

</div>

## 📌 概要

**簡単会議室予約システム** は、Node.js (Express.js) と PostgreSQL を利用して構築されたWebアプリケーションです。
社内や共有スペースにおける会議室の空き状況確認・予約申請・運用管理を直感的に行えます。

固定の会議名は `meeting.json` で管理し、データベース接続設定は `.env` ファイルに分離することで、高いメンテナンス性と安全性を両立しています。

---

## ✨ 主な機能

### 👤 一般ユーザー機能
* **予約状況閲覧**: 日付・時間帯（`HH:mm` 形式）ごとの予約状況を一覧で確認
* **新規予約作成**: 会議室・日時・会議名を指定して簡単予約
* **予約取り消し**: 自身の登録した予約をキャンセル可能

### 🔑 管理者機能 (`Role: admin`)
* **管理者権限 (`admin`)**: 新しい会議室の追加・編集機能
* **一元的な予約管理**: 全ユーザーの予約状況の閲覧および削除権限
* **管理画面ダッシュボード**: システム全体の利用管理

---

## 🏗 システムアーキテクチャ

本システムは、開発環境（Windows 11）から本番環境（Ubuntu Linux 24.04 + Apache 逆プロキシ）まで柔軟に対応できる構造になっています。

```
[ Client / Browser ]
         │ (HTTP / HTTPS)
         ▼
[ Apache Web Server ]  <-- Reverse Proxy (Port 80 / 443)
         │ (ProxyPass: localhost:3000)
         ▼
[ Express Application ]  <-- Node.js (Port 3000)
   ├── Auth Middleware (checkAuth)
   ├── Configuration (meeting.json / .env)
   ├── Template Engine (EJS)
   └── DB Client (node-postgres)
         │
         ▼
[ PostgreSQL Database ]  <-- Relational DB (Port 5432)
```

---

## 🛠 技術スタック

| 分類 | 技術 / ライブラリ | 用途 |
| :--- | :--- | :--- |
| **Server** | Node.js / Express.js | Webアプリケーションサーバー |
| **Database** | PostgreSQL | 予約データ・会議室マスタの永続化 |
| **View Engine** | EJS | 動的HTML描画 |
| **Config** | `dotenv` / `meeting.json` | 接続情報・固定パラメータの設定 |
| **Session** | `express-session` | 認証セッション管理 |

---

## 🗄 PostgreSQLの設定

ターミナルを開き、PostgreSQLにログインしてデータベース・ユーザー・テーブルを作成します。

```bash
sudo -u postgres psql
```

以下の SQL を順番に実行してください：

```sql
-- 1. 会議室予約用のデータベースを作成  
CREATE DATABASE meeting_room_db;  

-- 2. 専用のアプリケーションユーザーを作成（パスワードは任意）  
CREATE USER room_user WITH PASSWORD 'room_password';  
GRANT ALL PRIVILEGES ON DATABASE meeting_room_db TO room_user;  
-- ※ここで設定したユーザーとパスワードを .env に指定してください  

-- 3. 作成したデータベースに切り替え  
\c meeting_room_db  

-- 4. 会議室テーブルの作成  
CREATE TABLE rooms (  
    id SERIAL PRIMARY KEY,  
    name VARCHAR(255) NOT NULL UNIQUE  
);  

-- 5. 予約テーブルの作成（日付はDATE型、時間はTIME型）  
CREATE TABLE bookings (  
    id SERIAL PRIMARY KEY,  
    room_name VARCHAR(255) NOT NULL,  
    username VARCHAR(255) NOT NULL,  
    booking_date DATE NOT NULL,  
    start_time TIME NOT NULL,  
    end_time TIME NOT NULL  
);  

-- 初期データの投入  
INSERT INTO rooms (name) VALUES ('会議室A'), ('会議室B');
```

---

## 🚀 セットアップ手順

### 1. リポジトリのクローン & パッケージインストール
```bash
git clone https://github.com/your-username/meeting-room-booking.git
cd meeting-room-booking
npm install
```

### 2. 環境変数の設定 (`.env`)
`.env_sample` ファイルをコピーして `.env` を作成し、DB接続情報を指定してください。

```bash
cp .env_sample .env
```

**.env 設定例:**
```env
PORT=3000
SESSION_SECRET=your_secret_key_here
DB_HOST=localhost
DB_PORT=5432
DB_NAME=meeting_room_db
DB_USER=room_user
DB_PASSWORD=room_password
```

### 3. 固定会議名の設定 (`meeting.json`)
プロジェクト直下の `meeting.json` で固定の会議名リストを保持・変更できます。

```json
{
  "defaultMeetingNames": [
    "定例ミーティング",
    "プロジェクト進捗会議",
    "クライアント打ち合わせ",
    "社内研修"
  ]
}
```

### 4. アプリケーションの起動
```bash
# 通常起動
npm start

# 開発モード起動 (Nodemon等)
npm run dev
```

起動後、ブラウザで `http://localhost:3000` にアクセスしてください。

---

## 💡 技術的なこだわり

### 🛡 アクセス制御ミドルウェア (`checkAuth`) と `next()` バトンリレー

Express の高階関数パターンを用いて役割に応じたルート保護を実施しています。

```javascript
function checkAuth(role) {
    return function(req, res, next) {
        // 未ログイン時のリダイレクト
        if (!req.session.user) {
            return res.redirect('/');
        }
        // 管理者権限チェック (一般ユーザーのアクセスを拒否)
        if (role === 'admin' && req.session.user !== 'admin') {
            return res.status(403).send('閲覧権限がありません（管理者専用）');
        }
        // チェッククリア: 次の処理 (右隣の関数) へバトンを渡す
        next();
    };
}
```

---

## 📄 ライセンス

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
