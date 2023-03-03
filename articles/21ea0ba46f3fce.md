---
title: "【React】GitHub ActionsでFirebase Hostingに自動デプロイしてみた"
emoji: "🤙"
type: "tech"
topics:
  - "firebase"
  - "react"
  - "githubactions"
  - "deploy"
published: true
published_at: "2021-12-07 03:09"
---

GitHub ActionsでFirebase Hostingに自動デプロイする記事はたくさんあります。
ですが、多くの記事では自分でGitHubのsecret変数にトークンを登録したり、ymlファイルを作成したり、と初学者にはなかなか難しい内容です。
この記事では超簡単に出来る方法を書きました。いくつかの注意すべき点を気をつければ誰でもできます。

# 環境
macOS : Monterey 12.0.1
Node : 14.18.1
npm : 6.14.15
Firebase CLI : 9.23.0

[Firebase CLI]((https://firebase.google.com/docs/cli?hl=ja#setup_update_cli))のインストールはこちら
```shell:npm
npm install -g firebase-tools
```
```shell:yarn
yarn global add firebase-tools
```


# 1.プロジェクトを用意する

## CreateReactApp
まずはReactのプロジェクトを用意しましょう。
```sh
npx create-react-app firebase-hosting-deploy
```

## Firebaseのインストール
プロジェクトができたら作成したプロジェクトのディレクトリに移動してFirebaseをインストールしておきましょう。
```shell:npm
npm install firebase
```
```shell:yarn
yarn add firebase
```

# 2.Firebaseのプロジェクトを作成する

## 新しいプロジェクトを作成
[FIrebaseの公式サイト](https://firebase.google.com/?hl=ja)からプロジェクトを作成します。

![FirebaseNewProject](https://storage.googleapis.com/zenn-user-upload/d78df4e7c9a0-20211206.png)
今回は`firebase-hosting-deploy`というプロジェクトにしました。

![Googleアナリティクス](https://storage.googleapis.com/zenn-user-upload/079582eca18b-20211206.png)
Googleアナリティクスが必要な人は有効にしてプロジェクトを作成してください。

## アプリの追加
![FirebaseProject](https://storage.googleapis.com/zenn-user-upload/55e46b9ff386-20211206.png)
プロジェクトが作成できたらアプリにFirebaseを追加します。
今回はReactなので**ウェブ**を選択します。（**赤枠のボタンをクリック**）

![newApp](https://storage.googleapis.com/zenn-user-upload/c79079166048-20211206.png)
アプリ名を入力して**アプリを登録をクリック**します。
「このアプリの**Firebase Hosting**も設定します。」のチェックも入れておいてください。

![FirebaseSDK](https://storage.googleapis.com/zenn-user-upload/d89be936f28e-20211206.png)
続いてFirebaseをインストールするように言われますが、これはもう実施したので飛ばします。
FireStore等の機能を使用する人はfirebaseConfig（赤枠の部分）をコピーしておいてください。
今回のHostingでは使用しません。

![FirebaseCLI](https://storage.googleapis.com/zenn-user-upload/0c4e88b0c0dd-20211206.png)
まだFirebase CLIをインストールしていない人はインストールしてください。

![FirebaseHosting](https://storage.googleapis.com/zenn-user-upload/b71e5f8232f6-20211206.png)
最後にFirebase Hostingへのデプロイのやり方が書いています。
こちらは次の項目で詳しくやっていきます。


# 3.Firebase CLIを使ってGitHub Actionsを設定する

## Firebaseに接続する
まずはプロジェクトのディレクトリをFirebaseプロジェクトに接続します。
```shell
firebase login
```
を実行すると次のように聞かれます。
> Firebase optionally collects CLI usage and error reporting information to help improve our products. Data is collected in accordance with Google's privacy policy (https://policies.google.com/privacy) and is not used to identify you.
> 訳：FirebaseがCLI使用状況やエラー報告などの情報を勝手に集めて改善に役立てるんやけど、Googleのプライバシーポリシー的なやつに従って集めるし、ユーザー特定とかせんから安心して

> Allow Firebase to collect CLI usage and error reporting information? (Y/n) 
> 訳：FirebaseがCLIの使用状況とエラー報告を収集できるようにするけどええ？

あまりにもGoogleに不信感がある人以外は`y`を入力しましょう。
また、そのまま**Enterを押す**とYesが選択されます。

:::message
Enterを押すと大文字になっているほうが選択されるようになっています。
今回は(Y/n)なのでYesが選択されます。
:::

![Googleログイン](https://storage.googleapis.com/zenn-user-upload/4aeee1b07261-20211206.png)
ブラウザが立ち上がると**Googleアカウントでログイン**する画面が表示されます。
ログインしたら**Firebase CLIのアクセスを許可**してください。

![Successful](https://storage.googleapis.com/zenn-user-upload/3c3f52f27a02-20211206.png)
この画面が出てくれば成功です。


## プロジェクトの初期化

プロジェクトのディレクトリから**次のコマンドを実行**します。
```shell
firebase init
```
するとデカデカと書かれた**FIREBASE**に続いて色々と質問されるので順番に答えていきます。

> Which Firebase features do you want to set up for this directory? Press Space to select features, then Enter to confirm your choices. 
> 訳：このディレクトリにはどのFirebase機能を使うんか？ **スペースを押して機能を選択**し、**Enterキーを押して確定**できるで。

>  ◯ Realtime Database: Configure a security rules file for Realtime Database and (optionally) provision default instance
 ◯ Firestore: Configure security rules and indexes files for Firestore
 ◯ Functions: Configure a Cloud Functions directory and its files
❯◉ Hosting: Configure files for Firebase Hosting and (optionally) set up GitHub Action deploys
 ◯ Hosting: Set up GitHub Action deploys
 ◯ Storage: Configure a security rules file for Cloud Storage
 ◯ Emulators: Set up local emulators for Firebase products
(Move up and down to reveal more choices)

今回はFirebase Hostingを目的にしているので**Hostingを選びます**。
２つあるHostingの違いは以下の通りです。

> Hosting: Configure files for Firebase Hosting and (optionally) set up GitHub Action deploys
> 訳：Firebase Hostingのファイル構成するし、ついでにGitHub Actionのデプロイも設定しとくで

> Hosting: Set up GitHub Action deploys
> 訳：GitHubActionsのデプロイ設定しかせぇへんよ

とりあえず今回は**前者のHosting**を選択して**Enter**を押しましょう。

> First, let's associate this project directory with a Firebase project.You can create multiple project aliases by running firebase use --add, but for now we'll just set up a default project.
Please select an option: (Use arrow keys)
❯ Use an existing project 
  Create a new project 
  Add Firebase to an existing Google Cloud Platform project 
  Don't set up a default project
> 訳：最初はこのプロジェクトディレクトリをFirebaseプロジェクトに関連付けるとこからやで。
firebase use --addを実行したら複数のプロジェクトを作成できるけど、とりあえずデフォルトのプロジェクトを設定するだけでええよ。
オプションを選択してくれへん:(矢印キーを使用）
❯既存のプロジェクトを使用する
   新しいプロジェクトを作成する
   Firebaseを既存のGoogleCloudPlatformプロジェクトに追加します
   デフォルトのプロジェクトを設定しないでください

すでにFirebaseのプロジェクトを作成済みなので**既存のプロジェクトを使用する**を選択します。

> Select a default Firebase project for this directory: (Use arrow keys)
> 訳：このディレクトリのデフォルトのFirebaseプロジェクトを選択してくれへん:(矢印キーを使用）

先程作成した**Firebaseのプロジェクトを選択**します。

> What do you want to use as your public directory?(public)
> 訳：パブリックディレクトリとして何使う？

デフォルトではpublicフォルダを選択しようとするのですが、Reactなどの事前にビルドが必要なアプリではbuildフォルダに設定しておかないと画面が真っ白になります。
必ず**build**を入力してEnterを押してください。

> Configure as a single-page app (rewrite all urls to /index.html)? (y/N) 
> 訳：シングルページアプリとして構成する（すべてのURLを/index.htmlに書き換える）？

Reactの場合、SPAとして構成するので`y`と答えます。
**そのままEnterするとNoが選択されるので注意してください。**

> Set up automatic builds and deploys with GitHub? (y/N) 
> 訳：GitHubを使って自動ビルドとデプロイを設定する？

GitHub Actionsの設定をするかどうか聞かれているのでもちろん`y`と答えます。
**そのままEnterするとNoが選択されるので注意してください。**

> File public/index.html already exists. Overwrite? (y/N) 
> 訳：public/index.htmlはすでに存在してるけど上書きしてええんか？

この質問で上書きしてしまうとindex.htmlがFirebase仕様になり、Reactには欠かせない`<div id="root"></div>`が削除されてしまいます。
これは大問題なので`n`を選択します。**そのままEnter**でも大丈夫です。

:::details もしYesを選択してしまった場合の対処法
public/index.htmlの`<body>`タグ内に以下のコードを記述し、それ以外の`<body>`タグ内のコードを全て削除(orコメントアウト)する。
```html:index.html
<body>
  <div id="root"></div>
</body>
```
参考記事　[React+Firebaseで「Error: Target container is not a DOM element.」](https://qiita.com/KOBA-RYOTA/items/eb3e7f745109a0ab1387)
:::

> Detected a .git folder at /users/hoge/hoge/firebase-hosting-deploy
Authorizing with GitHub to upload your service account to a GitHub repository's secrets store.
> 訳:/Users/hoge/hoge/firebase-hosting-deployで.gitフォルダーを見つけたんやけど。
GitHubを使ってアカウントをGitHubリポジトリのシークレットストアにアップロードすることを承認するで。

よくわからないですが、**GitHubのアカウント認証**を要求されますので指示に従い認証を行ってください。

![Successful](https://storage.googleapis.com/zenn-user-upload/6de0b09a8380-20211207.png)
この画面が出てくれば成功です。

> For which GitHub repository would you like to set up a GitHub workflow? (format: user/repository) 
> 訳：どのGitHubリポジトリをワークフローに設定するねん

フォーマットに合わせて`[GitHubのアカウント名]/[リポジトリ名]`を入力してEnterを押します。

> Created service account github-action-********* with Firebase Hosting admin permissions.
> Uploaded service account JSON to GitHub as secret FIREBASE_SERVICE_ACCOUNT_FIR_HOSTING_DEPLOY.
> You can manage your secrets at https://github.com/*****/firebase-hosting-deploy/settings/secrets.
> 訳：FirebaseHostingの管理者権限でgithub-action-*******を勝手に作ったで。
> ついでにFIREBASE_SERVICE_ACCOUNT_FIR_HOSTING_DEPLOYとしてGitHubのsecretsにアップロードもしといたわ。
> URLでsecrets確認できるで。

この一瞬で結構すごいことをしてくれています。
要約すると「GitHub Actionsがいい感じに動くようにして、さらにGitHubのsecrets（環境変数）にFirebaseとの接続に必要なキー情報を登録したからもう面倒なことは全部終わったで。」って感じです。すごい。

> Set up the workflow to run a build script before every deploy? (y/N) 
> 訳：デプロイする前にビルドするように設定しとく？

Reactなどの事前にビルドが必要な場合は`y`を選択します。
**そのままEnterするとNoが選択されるので注意してください。**

> What script should be run before every deploy? (npm ci && npm run build) 
> 訳：デプロイする前にどのスクリプトを実行したらええん？

npmを使用する場合は**そのままEnter**を押してください。
yarnを使用する場合は`yarn install && yarn run build`を**入力してEnter**を押してください
:::message
`npm ci`に代わるyarnのコマンドは`yarn install --frozen-lockfile`ですが、このコマンドで設定したらエラーが出ました。誰か教えて下さい。
:::

すると自動で`.github/workflows/firebase-hosting-pull-request.yml`が作成されます。すごい。

> Set up automatic deployment to your site's live channel when a PR is merged? (Y/n) 
> 訳：プルリクエストがマージされたときに本番用のサイトにマージすんのかい？

これは好きなように設定してください。今回は**そのままEnter**を押して`y`を選択します。

> What is the name of the GitHub branch associated with your site's live channel? (main) 
> 訳：本番用のサイトにマージする場合の対象となるブランチはどれやねん

mainの人もいればmasterの人もいると思いますが、**本番環境のブランチを入力してEnter**を押してください。

> Action required: Visit this URL to revoke authorization for the Firebase CLI GitHub OAuth App:
https://github.com/settings/connections/applications/******
> 訳：このURLにアクセスしたらFirebaseCLIとGitHubの認証を解除できるで。
> Action required: Push any new workflow file(s) to your repo
> 訳：新しいワークフローファイルをリポジトリにアップしてや。
> Writing configuration info to firebase.json...
> 訳：`firebase.json`に色々書いたで。
> Writing project information to .firebaserc...
> 訳：プロジェクトの情報は`.firebaserc`に書いたで。

> Firebase initialization complete!
> 訳：Firebaseの初期化が完了したで。お疲れさん。

### これでやっとFirebaseの初期化が終わりました。

# 3.firebase deployしてみる
早くGitHub Actionsを使って自動デプロイしたい気持ちを抑えて、まずは`firebase deploy`コマンドを使って手動でデプロイしましょう。

## その前にbuildしてみよう
```shell:npm
npm run build
```
```shell:yarn
yarn build
```
`build`をするとbuildファイルが作成されますが、ファイル内の`index.html`をブラウザで見てみると真っ白な画面になっていると思います。
![white](https://storage.googleapis.com/zenn-user-upload/589a974ac9c2-20211207.png)

これは`package.json`に`"homepage": ".",`を追加することで解決します。
```diff json:package.json
{
  "name": "firebase-hosting-deploy",
  "version": "0.1.0",
+ "homepage": ".",
  "private": true,
  // ... 略
}
```
一度作成したbuildファイルを削除して、もう一度`build`してみましょう。
今度はちゃんと表示されるはずです。

:::message alert
この時点で画面が真っ白になる人は他の問題がある場合があるので次に進む前に解決しておきましょう。
:::

## firebase deploy
buildファイルの`index.html`で表示されることを確認したら実際にデプロイしてみましょう。
```shell
firebase deploy
```
デプロイできたらサイトを確認してみます。
ちゃんと表示されていたら喜んでください。やったー。
:::message
画面が真っ白になる人は`firebase.json`の"public"が"build"になっているかを確認しましょう。
```json:firebase.json
{
  "hosting": {
    "public": "build",
    "ignore": [
  // ...略
}
```
また何度もデプロイをしていて画面が真っ白になる場合はキャッシュが残っている場合があるので`command + shift + R`で更新しましょう。
:::

## GitHubにpushしておく
ここまで出来たらすべての変更履歴をGitHubにpushしておきましょう。

# 4.GitHub ActionsでFirebase Hostingに自動デプロイしてみる
最後にGitHub ActionsでFirebase Hostingに自動デプロイされるか確認しましょう。
手順は以下の通りです。
1. mainブランチからdevelopブランチを切る
2. HTMLファイルを一部修正する
3. GitHubにpushする
4. developブランチからmainブランチへプルリクを送る
5. mainブランチにプルリクをマージする

今回は`App.js`に`<h1>`タグを追加しました。
```diff jsx:src/App.js
// ...略
  <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
+       <h1>GitHub ActionsでFirebase Hostingに自動デプロイしてみた</h1>
        <p>
          Edit <code>src/App.js</code> and save to reload.
        </p>
// ...略
```

![Pull requests](https://storage.googleapis.com/zenn-user-upload/d662c846fda3-20211207.png)
![React](https://storage.googleapis.com/zenn-user-upload/ff75cbbfd305-20211207.png)
おー！できました！！！
これでdevelopブランチからmainブランチへプルリクを送ったときにプレビューが確認できます。

![merge deploy](https://storage.googleapis.com/zenn-user-upload/cd6b736e81e2-20211207.png)
さらにmainブランチにプルリクをマージしたら本番環境へデプロイされることが確認できました。

https://github.com/alpaca1231/firebase-hosting-deploy

# 5.おまけ
ESLint使ってるとよくわからないけど、`.env`ファイルにこれ書かないとエラーが出るときありますよね。
```env:.env
SKIP_PREFLIGHT_CHECK=true
```
これもGitHubのsecretsに登録してあげないとGitHub Actionsでエラー出ちゃうので忘れないように登録しておきましょうね。
（本当はESLintのバージョンに合わせてあげるのが一番いいんだけどね）


# 参考文献
https://firebase.google.com/docs/hosting/github-integration?hl=ja#:~:text=GitHub%E3%83%97%E3%83%AB%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%82%92%E4%BB%8B%E3%81%97%E3%81%A6%E3%83%A9%E3%82%A4%E3%83%96%E3%83%81%E3%83%A3%E3%83%B3%E3%83%8D%E3%83%AB%E3%81%A8%E3%83%97%E3%83%AC%E3%83%93%E3%83%A5%E3%83%BC%E3%83%81%E3%83%A3%E3%83%B3%E3%83%8D%E3%83%AB%E3%81%AB%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%99%E3%82%8B
https://classic.yarnpkg.com/en/docs/cli/install#:~:text=yarn%20install%20%2D%2Dfrozen%2Dlockfile
https://zenn.dev/watarukun/articles/8f3e318bacf97cabf879
