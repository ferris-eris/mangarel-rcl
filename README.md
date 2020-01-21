# 前提

- 2019年9月時点での情報をベースとしている
- Linux（Ubuntu 18.04LTSベース）環境
- [nvm](https://github.com/nvm-sh/nvm)はインストールされている前提
- Google Cloud Platformでのアカウントは取得済み
- コミック発売情報アプリ（サイト）を作成することで解説
  - プログラミング言語はTypeScriptを使用し、Reactでフロントエンドを構築する

# 環境構築

## Node.jsとyarnのインストール

- Cloud Functionsが12系をサポートしていないため10系もインストールする
  - 厳密には正式サポートは8系で10系はベータサポートだが、TypeScriptでのプログラミングを簡単にするため10系とする

```
nvm install v12.14.1
nvm install v10.18.0
nvm use v12.14.1
nvm alias default v12.14.1
npm install -g yarn
exec $SHELL -l  （ターミナルの再オープンでも構わない）
```

## create-react-appを実行

```
npx create-react-app mangarel-rcl --template typescript
cd mangarel-rcl
```

- `--typescript`オプションは次のバージョンアップで廃止予定のため、`--template typescript`を使用している
- `mangarel-rcl`はFirebaseサービスですでに使用済みなので唯一となりうる別の名前を指定すること

## tsconfig.json の変更

```
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react",
+   "baseUrl": "src",
+   "incremental": true
  },
  "include": ["src"],
+ "exclude": ["node_modules", "build", "scripts", "functions"]
}
```

- baseUrl
  - srcに設定しておくと、`import from myComponent from 'components/MyComponent';`のようにインポートをsrcをルートとした絶対パスで記述可能

- incremental
  - `true`にすることで再コンパイルが高速化

- exclude
  - 除外ディレクトリ
  - 特にfunctions以下はCloud Functionsのディレクトリでビルド設定を別にするため指定が必要

## LintとPrettier

```
yarn add -D stylelint prettier
yarn add -D eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-import eslint-plugin-jest eslint-plugin-prefer-arrow eslint-plugin-jsx-a11y eslint-plugin-prettier eslint-config-prettier eslint-config-airbnb
yarn add -D @typescript-eslint/parser @typescript-eslint/eslint-plugin
yarn add -D stylelint-config-prettier stylelint-config-standard stylelint-order stylelint-config-styled-components stylelint-processor-styled-components
yarn add -D prettier-stylelint
npm install -g typesync
exec $SHELL -l
typesync
yarn
```

- typesync
  - package.jsonを調べて、必要なTypeScriptの型ファイルがなければ自動でdevDependancies環境に追加

### その他のファイル

- 本リポジトリのものを利用
  - .eslintrc.js
  - stylelint.config.js
  - .gitignore
  - .eslintignore

## package.jsonの修正

npm scriptとしてESLintを実行できるようにするため

```
yarn add -D husky lint-staged
```

```
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "deploy:hosting": "npm run build && firebase deploy --only hosting",
    "deploy:rules": "firebase deploy --only firestore:rules",
    "eject": "react-scripts eject",
+   "lint": "eslint 'src/**/*.{js,jsx,ts,tsx}'; stylelint 'src/**/*.{css,jsx,tsx}'; cd functions/ && eslint 'src/**/*.{js,ts}'",
+   "precommit": "lint-staged",
    "test": "react-scripts test"
  },

+ "husky": {
+   "hooks": {
+     "pre-commit": "lint-staged"
+   }
+ },
+ "lint-staged": {
+   "src/**/*.{js,jsx,ts,tsx}": [
+     "eslint --fix",
+     "git add"
+   ],
+   "src/**/*.{css,jsx,tsx}": [
+     "stylelint --fix",
+     "git add"
+   ],
+   "functions/src/**/*.{js,ts}": [
+     "cd functions/ && eslint --fix",
+     "git add"
+   ]
+ }
}
```

# Firebaseプロジェクトの作成と初期化

https://firebase.google.com/?hl=ja にアクセス
  右上のアイコンで使用するアカウントを確認する
「使ってみる」をクリック
「プロジェクトを作成」をクリック
プロジェクト名を入力する
プロジェクト名はFirebasseサービス全体で一意である必要がある
スクリーンショットでは、プロジェクト名に「test4」と入力した時点で、その下のペンマークの横は「test4-9d330」となっていてこれは「test4」ですでに作成されているため重複を回避するため、「-9d330」が追加されている
「続行」ボタンをクリック
Googleアナリティックを使用するか聞かれる（有効のまま進むとする）
「Googleアナリティクスの構成」では以前にアカウントを作成していなければ新しいアカウントを作成することになる（はず）のでプロジェクトと同じ名前を指定
画面が拡張されるので
アナリティクスの地域を日本にし、その下のチェックボックスはすべて選択し「プロジェクトを作成」をクリック
ゆっくり処理が進む
「新しいプロジェクトの準備ができました」が表示されたら「続行」をクリックすると、以下を含む画面となる
今回はWebアプリを作成するため「</>」をクリック
アプリのニックネームにはプロジェクト名を指定
その下のチェックボックスをチェックすると、WebアプリとしてFirebaseのホスティング機能を使ってインターネットに公開することができる
具体的にはproject.firebase.comとproject.web.appにアクセスできるのだが、新しいサイトを作成を選択すると指定したサイト名も追加される
「アプリを登録」をクリック
Firebase SDKの追加ではWebpackを使うReactアプリには関係ないので無視し「次へ」をクリック
あとの3と4とは単なる説明なのでそのまま進め「コンソールに進む」をクリック

デフォルトのリソースロケーションの設定
左ペインの上の方にある歯車アイコンをクリックし、「プロジェクトの設定」を選択
「Google Cloud Platform（GCP）リソース ロケーション」が未選択になっているのでペンのアイコンをクリック
今回は東京に作成するので「asia-northeast1」を選択し「完了」をクリック

ターミナルに戻り「mangarel-rcl/」に入り直して以下を実行

```
npm install -g firebase-tools
exec $SHELL -l
firebase login
```

「Allow Firebase to collect CLI usage and error reporting information? (Y/n)」は会社的には`n`だと思う
場合によってはブラウザが立ち上がり、どのアカウントを使用するか聞いてくるので適切に選択する」

初期化

```
firebase init
```

FIREBASEのアスキーアートが表示され以下質問攻め
今回はRealtime DatabaseやCloud Storageは使用しないので、FirestoreとFunctionsとHostingのところにそれぞれ矢印キーで移動してスペースキーでチェックし、Enterキー

次（Project Setup）はそのままEnter
次は先に作成したプロジェクトを矢印キーで選択しEnter
「Firestore Setup]はそのままEnter、さらにEnter（2つデフォルトのファイル名）

Functions SetupはTypeScriptを選択し、次のTSLintの環境作成は**No**（近い将来非推奨になるため）、最後のnpmパッケージのインストールも**No**

Hosting Setupでは公開ディレクトリにbuildを指定、次のSPA設定は**Yes**

`firebase init`で行った作業（選択・指定）はfirebase.jsonと.firebasercという2つの設定ファイルを作成・・・後で変更可能

ここでアプリをデプロイしてみる

```
yarn build
firebase deploy --only hosting
```

Hosting URL : https://project名.firebaseapp.com と表示されるのでブラウザでアクセスすると、いつものCreate React Appの画面が表示される

## Cloud Functionsの環境整備

以下を実行

```
cd functions/
nvm use v10.18.0
```

package.jsonの以下の部分を書き換える

```
"engines": {
- "node": "8"
+ "node": "10"
},
```

`yarn`を実行してパッケージをインストールする
ここでyarnがないというメッセージが表示されたら、以下を実行しておく

```
npm install -g yarn
npm install -g typesync
npm install -g firebase-tools
```

tsconfig.jsonの修正

```
{
  "compilerOptions": {
+   "esModuleInterop": true,
+   "lib": ["dom", "esnext"],
    "module": "commonjs",
+   "moduleResolution": "node",
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "outDir": "lib",
    "rootDir": "src",
+   "removeComments": true,
+   "resolveJsonModule": true,
+   "sourceMap": true,
    "strict": true,
    "target": "es2017",
+   "types": ["jest", "node"],
+   "typeRoots": ["node_modules/@types", "../node_modules/@types"]
  },
  "compileOnSave": true,
  "include": ["src"],
+ "exclude": ["node_modules"]
}
```

修正しないと、import/exportが使えなかったり、JSONが読み込めなかったり、Jestが使えなかったり、外部サイトをクローリングしたときにDOMの解釈ができなかったりするため

テスト実行環境
Jestのインストール

```
yarn add -D jest ts-jest @types/jest
```

さらに

```
yarn add -D eslint prettier
yarn add -D eslint-plugin-import eslint-plugin-jest eslint-plugin-prefer-arrow eslint-plugin-prettier eslint-config-prettier eslint-config-airbnb-base
yarn add -D @typescript-eslint/parser @typescript-eslint/eslint-plugin
typesync
yarn
```

その他、リポジトリ内のファイル（.eslintrc.js、.eslintignore）は設定済みのためそのまま使用

package.jsonの設定追加（リポジトリ内のファイル参照）
package.jsonを修正し、設定したテストやLintを実行できるようにする（リポジトリ内のファイル参照）

functions/配下は同じリポジトリ内にあるが、上の階層のReactアプリとはTypeScript やLintの設定が別々
  VSCode でファイルを開く際、フロントエンドを編集するときは開くポイントを mangarel-demo/、  Cloud Functions を編集するときは mangarel-demo/functions/
  ビルドプロセスも別々に走らせる必要がある
  フロントエンドを編集するために開いた VSCode のウィンドウから functions/ ディレクトリを開いて中を触らないほうがいい

# 独自ドメインの設定

Firebaseコンソールから「Hosting」を選択

![ホスティング](images/ss04.png)


「カスタムドメインを追加」をクリック
次の画面で、ドメインを入力し、「<ドメイン>を既存のウェブサイトにリダイレクトする」はチェックせず「次へ」をクリック

次の画面では、ドメインの所有権を証明するための「google-site-verification」が提示される
ドメインを登録してるサービスの設定ツールに、以下のように記述する

```
txt @ google-site-verification=*************
```

伝播に時間がかかるので、5分ほど待ってから『確認』ボタンをクリック

所有権が確認されたら、今度はAレコードに設定するべき値が表示されるので、先程のTXTレコードに追記する形で設定

```
a @ 151.101.??.???
a @ 151.101.??.???
a www 151.101.??.???
a www 151.101.??.???
txt @ google-site-verification=*************
```

ドメインの伝播は5分ぐらいだが、SSL証明書のプロビジョニングには最大24時間かかる

※上記を自社ドメイン（サブドメイン：mangarel.ricoh.comとか）で実施する場合のTXTレコードの設定方法は別途調査が必要

# Seedデータ投入スクリプトの作成

## データベースの作成とAdmin環境の整備

主要なモデルは以下の3つ

- books （本）
- authors （著者）
- publishers （出版社）
  - 出版社については、functions/seeds/publishers.tsv に格納済み
  
出版社データをSeedデータとして最初にデータベースに格納する

Firebaseには（TypeScriptには）Railsのrake db:seedのようなコマンドは用意されていないのでスクリプトの作成が必要

### firestoreの有効化

Firebaseコンソールにアクセスして左ペインの「Database」をクリック

![Cloud Firestore](documents/ss05.png)

「データベースの作成」をクリック

※下の方に「またはRealtime　Databaseを選択」とあるが、FirebaseのデータベースにはFirestoreとRealtime Databaseとがあり、後者は古く機能的にも劣るので使用しない<br>
さらに、FirestoreにはネイティブモードとDatastoreモードとがあり、後者はCloud Datastoreとの互換モードなためネイティブモードで使用する<br>
作業中にDatastoreモードになっていたケースがあり、その場合、上の画像でその旨を示すエラーメッセージが表示されるので、そのメッセージに従ってGoogle Cloud Platformのコンソールからネイティブモードの変更する

Cloud Firestoreのセキュリティ保護ルールは「テストモードで開始」を選択
次のロケーションはasia-northeast1に固定されているはず
「完了」をクリック

SeedスクリプトはFirebaseのAdmin SDKを使用するのでその認証のために秘密鍵が必
要
左の歯車アイコンから、
「プロジェクトの設定>サービスアカウント」の画面に移動
「新しい秘密鍵の生成」をクリック
`mangarel-demo-firebase-adminsdk-*****-**********.json`ファイルがダウンロードされるので、`mangarel-rcl-firebase-adminsdk.json`にリネームしfunctions/src/に保存する
このファイルをGitHub等で公開しないように注意すること

### データ投入スクリプトの作成

Node でコマンドラインインターフェースを実現するライブラリとして、Commander.jsを使用する（TypeScriptの型も公式が提供）
CSV ファイルをパースするため、csv-parseも入れる

```
cd functions/
yarn add firebase commander csv-parse
yarn add -D @types/node
```

services/mangarel/models/publisher.ts
```
import { firestore } from 'firebase/app';

export type Publisher = {
  id?: string;
  name: string;
  nameReading: string | null;
  website: string | null;
  createdAt: firestore.Timestamp | null;
  updatedAt: firestore.Timestamp | null;
};
```

`id`を省略可能にしている理由
リレーショナル・データベースのレコードIDはレコードの中にあるが、Firestoreのドキュメント（レコードに相当）のIDはドキュメントの外にある
Firestoreから取得したドキュメントデータはそのままだとIDはの値を持たないので、必要に応じて後から入れられるようにしている

