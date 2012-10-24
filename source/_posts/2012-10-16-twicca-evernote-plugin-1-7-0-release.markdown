---
layout: post
title: "twicca - Evernote プラグイン 1.7.0 release"
date: 2012-10-16 23:46
comments: true
keywords: [ twicca, evernote, Android ]
categories: [ Android ]
---
[twicca - Evernote プラグイン](https://play.google.com/store/apps/details?id=jp.takuo.android.twicca.plugin.evernote) 1.7.0 をリリースしました。  
2012/11 よりパスワード認証でのサービス提供が終了することに伴い、Evernoteサービスの認証にOAuthを使うようになりました。  

<!-- more -->
認証手順で若干の不具合があり、ブラウザでOAuth認証した後設定画面に戻るのですが、そこからさらにブラウザが立ち上がってしまいます。 
認証自体は正常に終了していますのでそのままブラウザを終了して問題ありません。   

Evernote SDKでのintent処理に問題がある気がしていますが、試行錯誤したものの修正できなかったのでとりあえずこの状態でのリリースになっています。  
後ほどSDKを使わないでOAuthを処理する方法を試し修正していきたいと思います。  

※ SDKサイズが増えAPKのサイズも増大してしまいました。
