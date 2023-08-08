---
title: "Angular と Firebase の開発環境を構築する"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Angular, Firebase]
published: true
---

いざ自分で一から作ってみると、ローカルで開発するまでも多少手間取るのでそのログです。

コード: https://github.com/dany1468/angular_firebase_playground

## プロジェクト作成

Angular って npm init してからの方法はないんですかね。`directory` オプションでカレントを指定したとしても、すでに `package.json` があるとコンフリクトで失敗してしまいます。（`force` を付けても同じ)

なので `ng` を使えるようにするために angular/cli をグローバルにインストールしておきます。

```bash
$ npm install -g @angular/cli
```

これで `ng` コマンドが使えるようになります。今回は以下のオプション付きで作成しました。

```bash
$ ng new angular-firebase-playground --standalone --style scss --routing false
```

## Firebase CLI

https://www.npmjs.com/package/firebase-tools

こちらもグローバルにインストールしておきます。

```bash
$ npm install -g firebase-tools
```

## Firebase プロジェクト作成

ここはあるものとして進めます。

## Firebase プロジェクトと Angular プロジェクトを紐付ける

Firebase に関わるパッケージを導入します。

```bash
$ npm i firebase @angular/fire
```

`@angular/fire` は `ng add` で入れると、そのまま firebase の初期化もしてくれるのですが、今回は手動でやります。

```bash
$ firebase init emulators
```

`emulators` だけでは初期化できないようで、結局実プロジェクトと紐付けする必要があります。

`firebase.json` に以下のように emulators の設定が入ればOKです。今回は以下の機能を入れています。（あれ、`database` は要らなかったのでは。。）

```json
{
  "emulators": {
    "auth": {
      "port": 9099
    },
    "functions": {
      "port": 5001
    },
    "firestore": {
      "port": 8080
    },
    "database": {
      "port": 9000
    },
    "storage": {
      "port": 9199
    },
    "ui": {
      "enabled": true
    },
    "singleProjectMode": true
  }
}
```

## セキュリティルールの設定

emulator を動作させるにあたり、セキュリティルールが必要になるので、firestore と Storage 用に作成します。

まずは動作させるために全て許可の設定をしておきます。

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```

```
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read, write: if true;
    }
  }
}
```

`firebase.json` に以下の設定を追加します。

```json
  "storage": {
    "rules": "storage.rules"
  },
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  }
```

> エミュレータは最初に firebase.json ファイルの firestore.rules フィールドで指定されたルールを読み込みます。
https://cloud.google.com/firestore/docs/security/test-rules-emulator?hl=ja
> Firebase CLI からルールにアクセスする場合は、firebase.json ファイルに指定されているルールファイルに移動します。
https://firebase.google.com/docs/cli?hl=ja#the_firebasejson_file

## Angular に emulator の設定を追加

この辺は基本は angularfire の samples を参考にしています。
https://github.com/angular/angularfire/tree/master/samples/modular

`environment` に以下のように `useEmulators` 設定を追加します。

```ts
export const environment = {
  production: false,
  firebase: {
    apiKey: 'xxx',
    authDomain: 'xxx',
    projectId: 'xxx',
    storageBucket: 'xxx',
    messagingSenderId: 'xxx',
    appId: 'xxx',
    measurementId: 'xxx',
  },
  useEmulators: true,
};
```

### Standalone で作成する

先ほど言及した angularfire のサンプルは、standalone ではないので、`app.module.ts` に設定を記述します。

ここに関しては、どうやら angularfire 自体が Standalone API の対応が完全ではないようです。

今回は以下の記事を参考にしています。angularfire と Standalone API についても同記事で知りました。ありがとうございます。
https://zenn.dev/akai/articles/a66df86caab520

Standalone API に変更することで、記述する場所は変わりますが、angularfile のサンプルで emulator の対応をしている部分と大きくやっていることは変わりません。

`app.config.ts` に以下の設定を追加します。

```ts
const provideFirebase = () => importProvidersFrom([
  provideFirebaseApp(() => initializeApp(environment.firebase)),
  provideAuth(() => {
    const auth = getAuth();
    if (environment.useEmulators) {
      connectAuthEmulator(auth, 'http://localhost:9099');
    }
    return auth;
  }),
  provideFirestore(() => {
    const firestore = getFirestore();
    if (environment.useEmulators) {
      connectFirestoreEmulator(firestore, 'localhost', 8080);
    }
    return firestore;
  }),
  provideFunctions(() => {
    const functions = getFunctions();
    if (environment.useEmulators) {
      connectFunctionsEmulator(functions, 'localhost', 5001);
    }
    return functions;
  }),
  provideStorage(() => {
    const storage = getStorage();
    if (environment.useEmulators) {
      connectStorageEmulator(storage, 'localhost', 9199);
    }
    return storage;
  }),
]);

export const appConfig: ApplicationConfig = {
  providers: [provideFirebase()]
};
```

angularfire のサンプルの場合は `app.module.ts` で以下のリンクのように書かれています。
https://github.com/angular/angularfire/blob/34e89a4625646ff446f2a49c8d8bc489fc917655/samples/modular/src/app/app.module.ts#L53-L99

## エミュレータを使ったログインの確認

以下の angularfire のサンプルの auth とほぼ同じコンポーネントを作成し、ログインの確認だけ行いました。
https://github.com/angular/angularfire/blob/master/samples/modular/src/app/auth/auth.component.ts

以下の画像のように、匿名ログインと、エミュレータに追加した Email/Pass でログインできることを確認できるフォームを作っています。（詳細は Github のコードを見てください）

![](https://storage.googleapis.com/zenn-user-upload/e511ca83e578-20230808.png)

## エミュレータの起動

データの保存を行うためのオプション付きのコマンドを `Taskfile` に登録しています。

```yaml
version: '3'

tasks:
  emulator:up:
    cmds:
      - firebase emulators:start --import data --export-on-exit
```

## まとめ

長くなってしまったので Functions はまた別の記事で書こうと思います。