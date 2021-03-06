---
title: インクルード ファイル
description: インクルード ファイル
services: active-directory
documentationcenter: dev-center-name
author: brandwe
manager: mtillman
editor: ''
ms.service: active-directory
ms.devlang: na
ms.topic: include
ms.tgt_pltfrm: ios
ms.workload: identity
ms.date: 09/19/2018
ms.author: brandwe
ms.custom: include file
ms.openlocfilehash: 331d16df55e26df5d49555c636b307499dd052af
ms.sourcegitcommit: 6f59cdc679924e7bfa53c25f820d33be242cea28
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 10/05/2018
ms.locfileid: "48842947"
---
## <a name="register-your-application"></a>アプリケーションの登録
次の 2 つのセクションで説明する方法のいずれかを使用して、アプリケーションを登録できます。

### <a name="option-1-express-mode"></a>オプション 1: 簡易モード
次の手順に従って *Microsoft アプリケーション登録ポータル*でアプリケーションを登録する必要があります。
1. [Microsoft アプリケーション登録ポータル](https://apps.dev.microsoft.com/portal/register-app?appType=mobileAndDesktopApp&appTech=ios&step=configure)でアプリケーションを登録します。
2.  アプリケーションの名前とお使いのメール アドレスを入力します
3.  ガイド付きセットアップのオプションがオンになっていることを確認します。
4.  手順に従ってアプリケーション ID を取得し、それをコードに貼り付けます。

### <a name="option-2-advanced-mode"></a>オプション 2: 詳細モード

1.  [Microsoft アプリケーション登録ポータル](https://apps.dev.microsoft.com/portal/register-app)に進みます
2.  アプリケーションの名前を入力します
3.  ガイド付きセットアップのオプションがオフになっていることを確認します
4.  [`Add Platform`] をクリックし、[`Native Application`] を選択し、[`Save`] をクリックします。
5.  Xcode に戻ります。 `ViewController.swift` で、'`let kClientID`' で始まる行をさきほど登録したアプリケーション ID に変更します。

```swift
let kClientID = "Your_Application_Id_Here"
```

<!-- Workaround for Docs conversion bug -->
<ol start="6">
<li>
Ctrl キーを押しながら [<code>Info.plist</code>] をクリックしてコンテキスト メニューを表示し、<code>Open As</code>> <code>Source Code</code>
 をクリックします</li>
<li>
ルート ノード <code>dict</code> の下に次の内容を追加します。
</li>
</ol>

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLName</key>
        <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>msal[Your_Application_Id_Here]</string>
        </array>
    </dict>
</array>
```
<ol start="8">
<li>
<i><code>[Your_Application_Id_Here]</code></i> を、さきほど登録したアプリケーション ID に置き換えます
</li>
</ol>
