---
title: Azure での調整エラーのトラブルシューティング | Microsoft Docs
description: 調整エラー、Azure コンピューティングでの再試行とバックオフ。
services: virtual-machines
documentationcenter: ''
author: changov
manager: jeconnoc
editor: ''
tags: azure-resource-manager,azure-service-management
ms.service: virtual-machines
ms.devlang: na
ms.topic: troubleshooting
ms.workload: infrastructure-services
ms.date: 09/18/2018
ms.author: vashan, rajraj, changov
ms.openlocfilehash: 53d94d8674a064960b3447374f68af0d3fdf6e0c
ms.sourcegitcommit: b7e5bbbabc21df9fe93b4c18cc825920a0ab6fab
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 09/27/2018
ms.locfileid: "47412564"
---
# <a name="troubleshooting-api-throttling-errors"></a>API の調整エラーのトラブルシューティング 

Azure コンピューティング要求は、サービスの全体的なパフォーマンスを向上させるために、サブスクリプションとリージョン ベースで調整される場合があります。 Microsoft.Compute 名前空間でリソースを管理する Azure コンピューティング リソース プロバイダー (CRP) に対するすべての呼び出しは、許容されている最大の API 要求レートを超えないことが保証されています。 このドキュメントでは、API の調整、調整に関する問題をトラブルシューティングする方法の詳細、および調整を回避するためのベスト プラクティスについて説明します。  

## <a name="throttling-by-azure-resource-manager-vs-resource-providers"></a>Azure Resource Manager による調整とリソース プロバイダーによる調整  

Azure への入口として、Azure Resource Manager は、認証と最初の検証を行って、すべての受信 API 要求を調整します。 Azure Resource Manager の呼び出しレート制限と、関連する診断応答 HTTP ヘッダーについては[こちら](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-request-limits)を参照してください。
 
Azure API クライアントで調整エラーが取得された場合、HTTP ステータスは "429 要求が多すぎます" になります。 要求の調整が Azure Resource Manager に行われているか CRP などの基になるリソース プロバイダーによって行われているかを判断するには、GET 要求の `x-ms-ratelimit-remaining-subscription-reads` と GET 以外の要求の `x-ms-ratelimit-remaining-subscription-writes` 応答ヘッダーを調べます。 残りの呼び出し回数が 0 に近づいていれば、Azure Resource Manager によって定義されているサブスクリプションの一般的な呼び出し制限に達しています。 すべてのサブスクリプション クライアントによるアクティビティがまとめてカウントされます。 それ以外の場合、調整はターゲット リソース プロバイダー (要求 URL の `/providers/<RP>` セグメントによってアドレス指定されているもの) によって行なわれています。 

## <a name="call-rate-informational-response-headers"></a>呼び出しレートの情報提供応答ヘッダー 

| ヘッダー                            | 値の書式                           | 例                               | 説明                                                                                                                                                                                               |
|-----------------------------------|----------------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| x-ms-ratelimit-remaining-resource |```<source RP>/<policy or bucket>;<count>```| Microsoft.Compute/HighCostGet3Min;159 | この要求のターゲットを含むリソース バケットまたは操作グループを対象とする調整ポリシーの残りの API 呼び出し数                                                                   |
| x-ms-request-charge               | ```<count>   ```                             | 1                                     | 適用可能なポリシーの制限でのこの HTTP 要求に対して "課金" される呼び出しカウントの数。 これは、ほとんどの場合 1 になります。 バッチ要求 (仮想マシン スケール セットのスケーリングなど) では、複数のカウントが課金される可能性があります。 |


API 要求は複数の調整ポリシーの対象になる可能性があることに注意してください。 ポリシーごとに個別の `x-ms-ratelimit-remaining-resource` ヘッダーが存在します。 

仮想マシン スケール セット要求内の VM の削除に対する応答の例を次に示します。

```
x-ms-ratelimit-remaining-resource: Microsoft.Compute/DeleteVMScaleSet3Min;107 
x-ms-ratelimit-remaining-resource: Microsoft.Compute/DeleteVMScaleSet30Min;587 
x-ms-ratelimit-remaining-resource: Microsoft.Compute/VMScaleSetBatchedVMRequests5Min;3704 
x-ms-ratelimit-remaining-resource: Microsoft.Compute/VmssQueuedVMOperations;4720 
```

##<a name="throttling-error-details"></a>調整エラーの詳細

HTTP ステータス 429 は、通常は呼び出しレートの制限に達したという理由で要求を拒否するために使用されます。 コンピューティング リソース プロバイダーからの一般的な調整エラー応答は次の例のようになります (関係のあるヘッダーのみが示されています)。

```
HTTP/1.1 429 Too Many Requests
x-ms-ratelimit-remaining-resource: Microsoft.Compute/HighCostGet3Min;46
x-ms-ratelimit-remaining-resource: Microsoft.Compute/HighCostGet30Min;0
Retry-After: 1200
Content-Type: application/json; charset=utf-8
{
  "code": "OperationNotAllowed",
  "message": "The server rejected the request because too many requests have been received for this subscription.",
  "details": [
    {
      "code": "TooManyRequests",
      "target": "HighCostGet30Min",
      "message": "{\"operationGroup\":\"HighCostGet30Min\",\"startTime\":\"2018-06-29T19:54:21.0914017+00:00\",\"endTime\":\"2018-06-29T20:14:21.0914017+00:00\",\"allowedRequestCount\":800,\"measuredRequestCount\":1238}"
    }
  ]
}

```

残りの呼び出し回数が 0 というポリシーは、調整エラーが返されるためのポリシーの 1 つです。 ここでは、それは `HighCostGet30Min` です。 応答本文の全体的な形式は、一般的な Azure Resource Manager API エラー形式です (OData に準拠します)。 メインのエラー コードである `OperationNotAllowed` は、コンピューティング リソース プロバイダーが (その他の種類のクライアント エラーと共に) 調整エラーを報告するために使用するものです。 

上に示したように、すべての調整エラーには、クライアントが要求を再試行する前に待機する必要がある最少秒数を指定する `Retry-After` ヘッダーが含まれています。 

## <a name="best-practices"></a>ベスト プラクティス 

- Azure サービスの API エラーを無条件に再試行しないでください。 再試行できないエラーに遭遇したときにクライアント コードで一般的に発生するのは、急速な再試行ループに入ることです。 再試行のせいで最終的にターゲットの操作グループで許可される読み出し制限に達することで、サブスクリプションの他のクライアントに影響が出ます。 
- 大量の API の自動化では、ターゲットの操作グループで使用できる呼び出しカウントが下限のしきい値を下回った場合に、クライアント側でプロアクティブな自己調整を実装することを検討してください。 
- 非同期操作を追跡する場合は、Retry-After ヘッダーのヒントに従ってください。 
- クライアント コードが特定の仮想マシンに関する情報を必要とする場合は、すべての VM をリソース グループまたはサブスクリプション全体に含めた後でクライアント側で必要な VM を選択する代わりに、特定の VM に対するクエリを直接実行します。 
- クライアント コードで Azure の特定の場所にある VM、ディスク、およびスナップショットが必要な場合は、サブスクリプションのすべての VM をクエリした後でクライアント側で場所によってフィルター処理する代わりに、場所ベースの形式のクエリを使用します。`GET /subscriptions/<subId>/providers/Microsoft.Compute/locations/<location>/virtualMachines?api-version=2017-03-30` と `/subscriptions/<subId>/providers/Microsoft.Compute/virtualMachines` は、コンピューティング リソース プロバイダーに対して、リージョン エンドポイントのクエリを実行します。 特定の VM または仮想マシン スケール セットで API リソースの作成または更新を実行する場合は、(`provisioningState` に基づく) リソース URL そのものをポーリングするよりも、返された非同期操作の完了を追跡するほうがはるかに効率が良くなります。

## <a name="next-steps"></a>次の手順

Azure での他のサービスの再試行に関するガイダンスの詳細については、「[特定のサービスの再試行ガイダンス](https://docs.microsoft.com/en-us/azure/architecture/best-practices/retry-service-specific)」を参照してください。