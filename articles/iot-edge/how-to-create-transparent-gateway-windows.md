---
title: Azure IoT Edge で透過的なゲートウェイを作成する - Windows | Microsoft Docs
description: Azure IoT Edge を使用して、複数のデバイスの情報を処理できる透過的なゲートウェイを作成します
author: kgremban
manager: timlt
ms.author: kgremban
ms.date: 6/20/2018
ms.topic: conceptual
ms.service: iot-edge
services: iot-edge
ms.openlocfilehash: e9de037f886db7a48411959ef62e1e6687e54beb
ms.sourcegitcommit: 32d218f5bd74f1cd106f4248115985df631d0a8c
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 09/24/2018
ms.locfileid: "46984298"
---
# <a name="create-a-windows-iot-edge-device-that-acts-as-a-transparent-gateway"></a>透過的なゲートウェイとして動作する Windows IoT Edge デバイスを作成する

この記事では、IoT Edge デバイスを透過的なゲートウェイとして使用する場合の詳細な手順について説明します。 以降、この記事に記載される "*IoT Edge ゲートウェイ*" という用語は、透過的なゲートウェイとして使われる IoT Edge デバイスを指します。 概念に関する詳細については、「[IoT Edge デバイスをゲートウェイとして使用する方法][lnk-edge-as-gateway]」をご覧ください。

>[!NOTE]
>現時点では:
> * ゲートウェイが IoT Hub から切断された場合、ダウンストリーム デバイスはゲートウェイで認証を行うことができません。
> * また、Edge 対応デバイスは、IoT Edge ゲートウェイに接続できません。 
> * ダウンストリーム デバイスではファイル アップロードを使用できません。

透過的なゲートウェイの作成について困難な作業は、ゲートウェイをダウンストリーム デバイスに安全に接続することです。 Azure IoT Edge では、PKI インフラストラクチャを使用して、これらのデバイス間にセキュリティで保護された TLS 接続を設定することができます。 この場合、透過的なゲートウェイとして機能する IoT Edge デバイスにダウンストリーム デバイスが接続できるようにします。  合理的なセキュリティを維持するには、デバイスを自分のゲートウェイにのみ接続し、悪意のある可能性があるゲートウェイには接続しないようにする必要があるため、ダウンストリーム デバイスは Edge デバイスの ID を確認する必要があります。

デバイス ゲートウェイ トポロジに必要な信頼を有効にする証明書インフラストラクチャを作成できます。 この記事では、[X.509 CA セキュリティ][lnk-iothub-x509]を IoT Hub で有効にするために使用するのと同じ証明書のセットアップが前提となっています。これには、特定の IoT ハブ (IoT ハブ所有者の CA) に関連付けられている X.509 CA 証明書、およびこの CA と Edge デバイスの CA で署名された一連の証明書が含まれます。

![ゲートウェイの設定][1]

ゲートウェイは、接続の開始時に、ダウンストリーム デバイスに Edge デバイス CA 証明書を提示します。 ダウンストリーム デバイスは、Edge デバイス CA 証明書が、所有者 CA 証明書によって署名されていることを確認します。 このプロセスにより、ダウンストリーム デバイスは、ゲートウェイが信頼できるソースからのものであることを確認できます。

次の手順では、証明書を作成し、適切な場所にインストールするプロセスについて説明します。

## <a name="prerequisites"></a>前提条件
1.  透過的なゲートウェイとして使用する Windows デバイスに [Azure IoT Edge ランタイムをインストール][lnk-install-windows-x64]します。

1. OpenSSL for Windows を入手します。 OpenSSL をインストールする方法はたくさんあります。

   >[!NOTE]
   >OpenSSL をWindows デバイスに既にインストールしている場合は、この手順をスキップできますが、`openssl.exe` が `%PATH%` 環境変数で使用可能であることを確認してください。

   * [サードパーティの OpenSSL バイナリ](https://wiki.openssl.org/index.php/Binaries) (たとえば、[SourceForge のこちらのプロジェクトのバイナリ](https://sourceforge.net/projects/openssl/)) をダウンロードしてインストールします。
   
   * OpenSSL ソース コードをダウンロードし、ローカル コンピューター上でバイナリをビルドします。または [vcpkg](https://github.com/Microsoft/vcpkg) 経由でこれを実行します。 以下に示す指示は、vcpkg を使用してソース コードのダウンロード、コンパイル、および Windows コンピューターへの OpenSSL のインストールを、非常に簡単に使用できる手順で実行します。

      1. vcpkg をインストールするディレクトリに移動します。 ここでは、それを $VCPKGDIR と呼びます。 指示に従って [vcpkg](https://github.com/Microsoft/vcpkg) インストーラーをダウンロードし、実行します。
   
      1. vcpkg がインストールされたら、Powershell プロンプトから次のコマンドを実行して、Windows x64 用の OpenSSL パッケージをインストールします。 これは、通常は、完了まで約 5 分かかります。

         ```PowerShell
         .\vcpkg install openssl:x64-windows
         ```
      1. `$VCPKGDIR\installed\x64-windows\tools\openssl` を `PATH` 環境変数に追加して、`openssl.exe` ファイルを呼び出すことができるようにします。

1. 作業するディレクトリに移動します。 ここでは、これを $WRKDIR と呼びます。  すべてのファイルがこのディレクトリに作成されます。
   
   cd $WRKDIR

1.  次のコマンドで、必要な非運用環境の証明書を生成するスクリプトを取得します。 これらのスクリプトでは、透過的なゲートウェイの設定に必要な証明書を作成できます。

      ```PowerShell
      git clone https://github.com/Azure/azure-iot-sdk-c.git
      ```

1. 構成ファイルとスクリプト ファイルを作業ディレクトリにコピーします。 さらに、openssl_root_ca.cnf 構成ファイルを使用するように OPENSSL_CONF 環境変数を設定します。

   ```PowerShell
   copy azure-iot-sdk-c\tools\CACertificates\*.cnf .
   copy azure-iot-sdk-c\tools\CACertificates\ca-certs.ps1 .
   $env:OPENSSL_CONF = "$PWD\openssl_root_ca.cnf"
   ```

1. 次のコマンドを実行して、PowerShell がスクリプトを実行できるようにします

   ```PowerShell
   Set-ExecutionPolicy -ExecutionPolicy Unrestricted
   ```

1. 次のコマンドを使用してドット ソース形式で読み込み、スクリプトで使用される関数を PowerShell のグローバル名前空間に取り込みます
   
   ```PowerShell
   . .\ca-certs.ps1
   ```

1. OpenSSL が正しくインストールされていることを確認し、次のコマンドを実行して既存の証明書の名前の競合がないことを確認します。 問題がある場合は、スクリプトによって、システム上でそれらを修正する方法が説明されます。

   ```PowerShell
   Test-CACertsPrerequisites
   ```

## <a name="certificate-creation"></a>証明書の作成
1.  所有者 CA 証明書と 1 つの中間証明書を作成します。 これらはすべて `$WRKDIR` にあります。

      ```PowerShell
      New-CACertsCertChain rsa
      ```

1.  次のコマンドで、Edge デバイス CA 証明書と秘密キーを作成します。

   >[!NOTE]
   > ゲートウェイの DNS ホスト名と同じ名前を使用**しない**でください。 そのようにすると、これらの証明書に対するクライアント証明が失敗します。

   ```PowerShell
   New-CACertsEdgeDevice "<gateway device name>"
   ```

## <a name="certificate-chain-creation"></a>証明書チェーンの作成
次のコマンドを使用して、所有者 CA 証明書、中間証明書、Edge デバイス CA 証明書から証明書チェーンを作成します。 チェーン ファイルに配置すると、透過的なゲートウェイとして機能する Edge デバイスに簡単にインストールできます。

   ```PowerShell
   Write-CACertsCertificatesForEdgeDevice "<gateway device name>"
   ```

   スクリプトの実行の出力は、次の証明書とキーです。
   * `$WRKDIR\certs\new-edge-device.*`
   * `$WRKDIR\private\new-edge-device.key.pem`
   * `$WRKDIR\certs\azure-iot-test-only.root.ca.cert.pem`

## <a name="installation-on-the-gateway"></a>ゲートウェイへのインストール
1.  次のファイルを $WRKDIR から Edge デバイス上の任意の場所にコピーします。この場所を $CERTDIR と呼びます。 Edge デバイスで証明書を生成した場合は、この手順をスキップします。

   * デバイス CA 証明書 - `$WRKDIR\certs\new-edge-device-full-chain.cert.pem`
   * デバイス CA 秘密キー - `$WRKDIR\private\new-edge-device.key.pem`
   * 所有者 CA - `$WRKDIR\certs\azure-iot-test-only.root.ca.cert.pem`

2.  セキュリティ デーモン構成 yaml ファイル内の `certificate` プロパティを、証明書とキー ファイルを配置した場所のパスに設定します。

```yaml
certificates:
  device_ca_cert: "$CERTDIR\\certs\\new-edge-device-full-chain.cert.pem"
  device_ca_pk: "$CERTDIR\\private\\new-edge-device.key.pem"
  trusted_ca_certs: "$CERTDIR\\certs\\azure-iot-test-only.root.ca.cert.pem"
```
## <a name="deploy-edgehub-to-the-gateway"></a>ゲートウェイへの Edge ハブのデプロイ
Azure IoT Edge の主要な機能の 1 つは、クラウドから IoT Edge デバイスにモジュールをデプロイできることです。 このセクションでは、空に見えるデプロイを作成します。ただし、他のモジュールがない場合でも、Edge ハブはすべてのデプロイに自動的に追加されます。 Edge ハブは、透過的なゲートウェイとして動作させる Edge デバイスに必要な唯一のモジュールであるため、空のデプロイを作成すれば十分です。 
1. Azure Portal で、お使いの IoT ハブに移動します。
2. **IoT Edge** に移動し、ゲートウェイとして使用する IoT Edge デバイスを選択します。
3. **[Set Modules] \(モジュールの設定)** を選択します。
4. **[次へ]** を選択します。
5. **[ルートの指定]** ステップで、すべてのモジュールから IoT Hub にすべてのメッセージを送信する既定のルートを用意しておく必要があります。 そうしていない場合は、次のコードを追加し、**[次へ]** を選択します。
   ```JSON
   {
       "routes": {
           "route": "FROM /* INTO $upstream"
       }
   }
   ```
6. [Review template]\(テンプレートのレビュー\) ステップで、**[送信]** を選びます。

## <a name="installation-on-the-downstream-device"></a>ダウンストリーム デバイスへのインストール
ダウンストリーム デバイスには、「[.NET を使用してデバイスを IoT Hub にデバイスを接続する][lnk-iothub-getstarted]」で説明されている単純なアプリケーションなど、[Azure IoT device SDK][lnk-devicesdk] を使用する任意のアプリケーションを指定できます。 ダウンストリーム デバイス アプリケーションは、ゲートウェイ デバイスへの TLS 接続を検証するために**所有者 CA** 証明書を信頼する必要があります。 この手順は通常、OS レベル、または (特定の言語の場合) アプリケーション レベルの 2 つの方法で実行できます。

### <a name="os-level"></a>OS レベル
OS 証明書ストアにこの証明書をインストールすると、すべてのアプリケーションが所有者 CA 証明書を信頼された証明書として使用できます。

* Ubuntu - Ubuntu ホストに CA 証明書をインストールする方法の例を次に示します。

   ```cmd
   sudo cp $CERTDIR/certs/azure-iot-test-only.root.ca.cert.pem  /usr/local/share/ca-certificates/azure-iot-test-only.root.ca.cert.pem.crt
   sudo update-ca-certificates
   ```
 
    "Updating certificates in /etc/ssl/certs...1 added, 0 removed; done." (/etc/ssl/certs の証明書を更新しています... 1 個追加、0 個削除されました。完了しました) というメッセージが表示されます。

* Windows - Windows ホストに CA 証明書をインストールする方法の例を次に示します。
  * [スタート] メニューで、[コンピューター証明書の管理] を実行します。 `certlm` という名前のユーティリティが起動します。
  * [証明書 - ローカル コンピューター] --> [信頼されたルート証明書] --> [証明書] --> 右クリック --> [すべてのタスク] --> [インポート] に移動して、証明書インポート ウィザードを起動します。
  * 指示に従って次の手順を実行し、証明書ファイル $CERTDIR/certs/azure-iot-test-only.root.ca.cert.pem をインポートします。
  * 完了すると、「正常にインポートされました」メッセージが表示されます。

### <a name="application-level"></a>アプリケーション レベル
.NET アプリケーションの場合、次のスニペットを追加して PEM 形式の証明書を信頼することができます。 変数 `certPath` を `$CERTDIR\certs\azure-iot-test-only.root.ca.cert.pem` で初期化します。

   ```
   using System.Security.Cryptography.X509Certificates;

   ...

   X509Store store = new X509Store(StoreName.Root, StoreLocation.CurrentUser);
   store.Open(OpenFlags.ReadWrite);
   store.Add(new X509Certificate2(X509Certificate2.CreateFromCertFile(certPath)));
   store.Close();
   ```

## <a name="connect-the-downstream-device-to-the-gateway"></a>ゲートウェイへのダウンストリーム デバイスの接続
ゲートウェイ デバイスのホスト名を参照する接続文字列を使用して、IoT Hub デバイス SDK を初期化する必要があります。 これを行うには、`GatewayHostName` プロパティをデバイスの接続文字列を追加します。 デバイスの接続文字列の例を次に示します。これには `GatewayHostName` プロパティが追加されています。

   ```
   HostName=yourHub.azure-devices.net;DeviceId=yourDevice;SharedAccessKey=XXXYYYZZZ=;GatewayHostName=mygateway.contoso.com
   ```

   >[!NOTE]
   >これは、すべて正しく設定されていることをテストするサンプル コマンドです。 "verified OK" (検証 OK) というメッセージが表示されます。
   >
   >openssl s_client -connect mygateway.contoso.com:8883 -CAfile $CERTDIR/certs/azure-iot-test-only.root.ca.cert.pem -showcerts

## <a name="routing-messages-from-downstream-devices"></a>ダウンストリーム デバイスからのメッセージのルーティング
IoT Edge ランタイムでは、モジュールによって送信されたメッセージと同じように、ダウンストリーム デバイスから送信されたメッセージをルーティングできます。 これにより、クラウドにデータを送信する前に、ゲートウェイで実行されているモジュールの分析を実行することができます。 次のルートは、`sensor` という名前のダウンストリーム デバイスから `ai_insights` という名前のモジュールにメッセージを送信するために使用されます。

   ```json
   { "routes":{ "sensorToAIInsightsInput1":"FROM /messages/* WHERE NOT IS_DEFINED($connectionModuleId) INTO BrokeredEndpoint(\"/modules/ai_insights/inputs/input1\")", "AIInsightsToIoTHub":"FROM /messages/modules/ai_insights/outputs/output1 INTO $upstream" } }
   ```

メッセージ ルーティングについて詳しくは、[モジュールの構成に関する記事][lnk-module-composition]をご覧ください。

[!INCLUDE [](../../includes/iot-edge-extended-offline-preview.md)]

## <a name="next-steps"></a>次の手順
[IoT Edge モジュールを開発するための要件と ツールについて理解します][lnk-module-dev]。

<!-- Images -->
[1]: ./media/how-to-create-transparent-gateway/gateway-setup.png

<!-- Links -->
[lnk-install-windows-x64]: ./how-to-install-iot-edge-windows-with-windows.md
[lnk-module-composition]: ./module-composition.md
[lnk-devicesdk]: ../iot-hub/iot-hub-devguide-sdks.md
[lnk-tutorial1-win]: tutorial-simulate-device-windows.md
[lnk-tutorial1-lin]: tutorial-simulate-device-linux.md
[lnk-edge-as-gateway]: ./iot-edge-as-gateway.md
[lnk-module-dev]: module-development.md
[lnk-iothub-getstarted]: ../iot-hub/quickstart-send-telemetry-dotnet.md
[lnk-iothub-x509]: ../iot-hub/iot-hub-x509ca-overview.md
[lnk-iothub-secure-deployment]: ../iot-hub/iot-hub-security-deployment.md
[lnk-iothub-tokens]: ../iot-hub/iot-hub-devguide-security.md#security-tokens
[lnk-iothub-throttles-quotas]: ../iot-hub/iot-hub-devguide-quotas-throttling.md
[lnk-iothub-devicetwins]: ../iot-hub/iot-hub-devguide-device-twins.md
[lnk-iothub-c2d]: ../iot-hub/iot-hub-devguide-messages-c2d.md
[lnk-ca-scripts]: https://github.com/Azure/azure-iot-sdk-c/blob/master/tools/CACertificates/CACertificateOverview.md
[lnk-modbus-module]: https://github.com/Azure/iot-edge-modbus
