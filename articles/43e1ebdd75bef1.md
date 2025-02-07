---
title: "【PWA】OneSignalを使ってiOSにWebプッシュ通知を送るRTA"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "pwa", "safari", "onesignal", "webpush", ]
published: true
---

![ちょっと株式会社 Advent Calendar 2023 4日目の記事](/images/43e1ebdd75bef1/eye_catching.png)
https://adventar.org/calendars/8910

## はじめに

2023年3月27日に公開された[iOS 16.4](https://webkit.org/blog/13966/webkit-features-in-safari-16-4/#web-push-on-ios-and-ipados)で、iOSおよびiPadOSでホーム画面に追加されたWebアプリにWeb Pushがサポートされました。

今回は超カンタンにWebプッシュ通知機能が使えることを証明するため、iOS SafariにWebプッシュ通知を送るまでのRTAに挑戦してみたいと思います。
**多分これが一番早いと思います。**

https://github.com/alpaca1231/next-pwa-onesignal-example

## レギュレーション

**達成条件**
Next.jsで新規作成したWebアプリをPWAにしてiPhoneのSafariでWebプッシュ通知を受け取る（Any%）

**使用技術**
Next.js v14.0.3 (App router)
TypeScript v5.3.2

**タイム計測方法**
開始点：Terminalで`create-next-app`を実行した瞬間
終了点：Webプッシュ通知を送信してiPhoneで通知を受け取った瞬間
タイマー：外部タイマーを使用し、ビデオ記録にタイマーを表示する

## OneSignalとは
https://onesignal.com/
プッシュ通知を簡単に送信することができるサービスです。
多様なプラットフォーム（iOS、Android、Web）に対応しており、セグメンテーション、A/Bテストなどの高度な機能も提供しています。無料プランもあります。

## 実装の流れ
1. [OneSignalに登録する](#onesignalに登録する)
2. [Next.jsにOneSignalを導入する](#nextjsにonesignalを導入する)
3. [manifest.jsonを作成してPWAにする](#manifestjsonを作成してpwaにする)
4. [デプロイする](#デプロイする)
5. [iPhoneのSafariからA2HSする](#iphoneのsafariからa2hsする)
6. [OneSignalからプッシュ通知を送る](#onesignalからプッシュ通知を送る)

## OneSignalに登録する
OneSignalの[SignUp](https://dashboard.onesignal.com/signup)からアカウントを作成します。

Create Appまで進めた後、`Settings`→`Platforms`に遷移して`App Settings`から`Web`を選択します。

`1.Choose Integration`は`Typical Site`を選択します。
Webアプリに最低限必要な設定は

| 項目 | 設定値 |
| --- | --- |
| SITE NAME | `Create Next App` |
| SITE URL | `http://localhost:3000/` |

です。
`SITE URL`には通常デプロイした環境のURLを設定しますが、ローカルで動作確認する必要があるため、現時点では`http://localhost:3000/`を設定しています。
![Webアプリの設定](/images/43e1ebdd75bef1/onesignal_create_app_web.png)

その他の設定はデフォルト値のままで問題ありません。`Save`を押下して次に進みます。

`6.Upload Files`でダウンロードしたファイルは解凍して後ほど`public`ディレクトリに格納します。
また`7.Add Code to Site`から`appId`だけメモしておきます。

![OneSignal SDK Fileをダウンロード](/images/43e1ebdd75bef1/onesignal_sdk_intro.png)

## Next.jsにOneSignalを導入する
`create-next-app`してOneSignalを導入していきます。
今回はApp routerを使用していますがpages routerでも同様です。

手順は以下の3つだけです。

### 1. `OneSignalSDKWorker.js`を`public`に格納する
先ほどダウンロードして解凍した`OneSignalSDKWorker.js`を`public`ディレクトリに格納します。

### 2. `react-onesignal`をインストールする
React用のモジュールですがTypeScriptをサポートしていて便利なので使用します。

https://www.npmjs.com/package/react-onesignal

```sh
npm i react-onesignal
# or
yarn add react-onesignal
# or
pnpm add react-onesignal
# or
bun add react-onesignal
```

### 3. OneSignalを初期化する
OneSignalはwindowオブジェクトを拡張して動作するようになっているため、クライアントコンポーネントにする必要があります。
また下記のコード例ではOneSignalを初期化した後すぐにプッシュ通知のプロンプトを表示するように呼び出しています。

```tsx:src/lib/OneSignalInitial.tsx
'use client'

import { useEffect } from 'react'
import OneSignal from 'react-onesignal'

export const OneSignalInitial = () => {
  useEffect(() => {
    const oneSignalInit = async () => {
      await OneSignal.init({
        appId: process.env.NEXT_PUBLIC_ONESIGNAL_APP_ID || '',
      }).then(() => {
        OneSignal.Slidedown.promptPush()
      })
    }
    oneSignalInit()
  }, [])
  return null
}
```

```diff tsx:src/app/layout.tsx
 import type { Metadata } from 'next'
 import { Inter } from 'next/font/google'
 import './globals.css'
+import { OneSignalInitial } from '@/lib/OneSignalInitial'

 // 省略

 export default function RootLayout({
   children,
 }: {
   children: React.ReactNode
 }) {
   return (
     <html lang="en">
       <body className={inter.className}>
+        <OneSignalInitial />
         {children}
       </body>
     </html>
   )
 }
```

ここまで設定して開発環境で確認するとプッシュ通知の購読許可するプロンプトが表示されます。
![プッシュ通知を購読するプロンプト](/images/43e1ebdd75bef1/subscribe.png)

この時点ですでにiOS Safari以外のブラウザではプッシュ通知を受け取ることができるようになっています。
下記の画像はMac chromeで`Subscribe`を押下した直後に届いたプッシュ通知です。
![Thanks for subscribing!](/images/43e1ebdd75bef1/thanks.png)


## manifest.jsonを作成してPWAにする
manifest.json generaterを使用して必要最低限のmanifest.jsonを生成します。
今回は下記のgeneratorを使用しました。
https://www.simicart.com/manifest-generator.html/

生成された`manifest.webmanifest`を`manifest.json`に変換してiconの画像ファイルと一緒に`public`に格納します。

```json:public/manifest.json
{
    "theme_color": "#f69435",
    "background_color": "#f69435",
    "display": "standalone",
    "scope": "/",
    "start_url": "/",
    "name": "next pwa webpush example",
    "short_name": "example",
    "icons": [
        {
            "src": "/icon-192x192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "/icon-256x256.png",
            "sizes": "256x256",
            "type": "image/png"
        },
        {
            "src": "/icon-384x384.png",
            "sizes": "384x384",
            "type": "image/png"
        },
        {
            "src": "/icon-512x512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

`manifest.json`を`src/app/layout.tsx`で読み込みます

```diff tsx:src/app/layout.tsx
 // 省略
 return (
     <html lang="en">
+      <head>
+        <link rel="manifest" href="/manifest.json" />
+        <link rel="apple-touch-icon" href="/icon-192x192.png"></link>
+        <meta name="theme-color" content="#f69435" />
+      </head>
      <body className={inter.className}>
        <OneSignalInitial />
        {children}
      </body>
     </html>
   )
```

これでPWAになりました。

## デプロイする

お好きなホスティング先にデプロイしてください。
今回はVercelにデプロイしました。
https://next-pwa-onesignal-example.vercel.app/

デプロイ後にはOneSignalの`App Settings`で`SITE URL`をデプロイした環境のURLに変更する必要があります。
![SITE URLを更新](/images/43e1ebdd75bef1/onesignal_update_site_url.png)


## iPhoneのSafariからA2HSする

iPhoneのSafariからA2HS(Add to Home Screen)します。
ホームに追加されたアプリを開くとプッシュ通知の購読許可するプロンプトが表示されます。
![PWAでプッシュ通知の購読を許可するプロンプト](/images/43e1ebdd75bef1/pwa_subscribe.jpg)

## OneSignalからプッシュ通知を送る
OneSignalの`Messages`→`Push`に遷移して`New Message`から新しいプッシュ通知を作成します。

Messageの内容は必須です。
それ以外はデフォルトのままでも構いません。

![新しいプッシュ通知を作成](/images/43e1ebdd75bef1/onesignal_new_push.png)

最後に`Review and Send`を押下してプッシュ通知を送信します。
無事iOS SafariにWebプッシュ通知を送ることができました。
![iOS SafariにWebプッシュ通知](/images/43e1ebdd75bef1/ios_push_notification.jpg)


## RTA 結果タイム
この記事を見ながら実装してタイムを計測してみました。
焦ってしまい正確に計測できませんでしたが、10分を切ることができました。
![ストップウォッチ 09:28.25](/images/43e1ebdd75bef1/rta_time.png)
![RTA 録画映像](/images/43e1ebdd75bef1/rta.gif)


10分程度で出来るので皆さんもぜひ挑戦してみてください。
