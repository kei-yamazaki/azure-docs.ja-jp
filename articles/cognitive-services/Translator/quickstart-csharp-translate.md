---
title: 'クイック スタート: テキストを翻訳する - Translator Text、C#'
titleSuffix: Azure Cognitive Services
description: このクイック スタートでは、C# で Translator Text API を使ってテキストを別の言語に翻訳します。
services: cognitive-services
author: noellelacharite
manager: cgronlun
ms.service: cognitive-services
ms.component: translator-text
ms.topic: quickstart
ms.date: 06/15/2018
ms.author: nolachar
ms.openlocfilehash: e60b2a9ea7c56352a1b4dfb65537ba6dc57fcb68
ms.sourcegitcommit: ce526d13cd826b6f3e2d80558ea2e289d034d48f
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 09/19/2018
ms.locfileid: "46365963"
---
# <a name="quickstart-translate-text-with-c35"></a>クイック スタート: C&#35; を使ったテキストの翻訳

このクイック スタートでは、Translator Text API を使って、テキストを別の言語に翻訳します。

このサンプルのソース コードは、[GitHub](https://github.com/MicrosoftTranslator/Text-Translation-API-V3-C-Sharp) で入手できます。

## <a name="prerequisites"></a>前提条件

このコードを Windows 上で実行するには、[Visual Studio 2017](https://www.visualstudio.com/downloads/) が必要です。 (無料の Community Edition でかまいません。)

Translator Text API を使用するには、サブスクリプション キーも必要となります。「[Translator Text API にサインアップする方法](translator-text-how-to-signup.md)」を参照してください。

## <a name="translate-request"></a>翻訳要求

以下のコードは、[Translate](./reference/v3-0-translate.md) メソッドを使ってソース テキストを別の言語に翻訳します。

1. 適切な IDE で新しい C# プロジェクトを作成します。
2. 次に示すコードを追加します。
3. `key` の値を、お使いのサブスクリプションで有効なアクセス キーに置き換えます。
4. プログラムを実行します。

```csharp
using System;
using System.Net.Http;
using System.Text;
// NOTE: Install the Newtonsoft.Json NuGet package.
using Newtonsoft.Json;

namespace TranslatorTextQuickStart
{
    class Program
    {
        static string host = "https://api.cognitive.microsofttranslator.com";
        static string path = "/translate?api-version=3.0";
        // Translate to German and Italian.
        static string params_ = "&to=de&to=it";

        static string uri = host + path + params_;

        // NOTE: Replace this example key with a valid subscription key.
        static string key = "ENTER KEY HERE";

        static string text = "Hello world!";

        async static void Translate()
        {
            System.Object[] body = new System.Object[] { new { Text = text } };
            var requestBody = JsonConvert.SerializeObject(body);

            using (var client = new HttpClient())
            using (var request = new HttpRequestMessage())
            {
                request.Method = HttpMethod.Post;
                request.RequestUri = new Uri(uri);
                request.Content = new StringContent(requestBody, Encoding.UTF8, "application/json");
                request.Headers.Add("Ocp-Apim-Subscription-Key", key);

                var response = await client.SendAsync(request);
                var responseBody = await response.Content.ReadAsStringAsync();
                var result = JsonConvert.SerializeObject(JsonConvert.DeserializeObject(responseBody), Formatting.Indented);

                Console.OutputEncoding = UnicodeEncoding.UTF8;
                Console.WriteLine(result);
            }
        }

        static void Main(string[] args)
        {
            Translate();
            Console.ReadLine();
        }
    }
}
```

## <a name="translate-response"></a>翻訳応答

成功した応答は、次の例に示すように JSON で返されます。

```json
[
  {
    "detectedLanguage": {
      "language": "en",
      "score": 1.0
    },
    "translations": [
      {
        "text": "Hallo Welt!",
        "to": "de"
      },
      {
        "text": "Salve, mondo!",
        "to": "it"
      }
    ]
  }
]
```

## <a name="next-steps"></a>次の手順

このクイック スタートをはじめとする各種ドキュメントで表記変換と言語認識を含んだサンプル コードを詳しく見てみましょう。GitHub にある Translator Text の各種サンプル プロジェクトもご覧ください。

> [!div class="nextstepaction"]
> [GitHub で C# のコード例を詳しく見てみる](https://aka.ms/TranslatorGitHub?type=&language=c%23)
