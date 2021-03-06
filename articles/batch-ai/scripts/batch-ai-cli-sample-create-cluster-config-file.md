---
title: Azure CLI のサンプル スクリプト - 構成ファイルを使用した Batch AI クラスターの作成 | Microsoft Docs
description: Azure CLI のサンプル スクリプト - JSON ファイルで構成設定を指定することで Batch AI クラスターを作成します。
services: batch-ai
documentationcenter: ''
author: dlepow
manager: jeconnoc
editor: ''
ms.assetid: ''
ms.service: batch-ai
ms.devlang: azurecli
ms.topic: sample
ms.tgt-pltfrm: multiple
ms.workload: na
ms.date: 08/16/2018
ms.author: danlep
ms.openlocfilehash: a1e472d237977d1948c69828d8ec391522896774
ms.sourcegitcommit: ebd06cee3e78674ba9e6764ddc889fc5948060c4
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 09/07/2018
ms.locfileid: "44058163"
---
# <a name="cli-example-create-a-batch-ai-cluster-using-a-cluster-configuration-file"></a>CLI の例: クラスター構成ファイルを使用した Batch AI クラスターの作成

このスクリプトは、JSON 構成ファイルを使用して Batch AI クラスターの設定を指定する方法を示しています。 `az batchai cluster create` の対応するコマンド ライン パラメーターの代わりに、これらの設定を使用します。 構成ファイルは、複数のファイル システムをクラスター ノードにマウントするときや、複数のクラスターで同じ構成を使用する必要があるときに便利です。

[!INCLUDE [cloud-shell-try-it.md](../../../includes/cloud-shell-try-it.md)]

CLI をローカルにインストールして使用することを選択する場合、このクイック スタートでは、Azure CLI バージョン 2.0.38 以降を実行している必要があります。 バージョンを確認するには、`az --version` を実行します。 インストールまたはアップグレードする必要がある場合は、「[Azure CLI 2.0 のインストール](/cli/azure/install-azure-cli)」を参照してください。 

## <a name="example-script"></a>サンプル スクリプト

[!code-azurecli-interactive[main](../../../cli_scripts/batch-ai/create-cluster/create-cluster-config-file.sh "Create Batch AI cluster - configuration file")]

## <a name="clean-up-deployment"></a>デプロイのクリーンアップ

次のコマンドを実行して、リソース グループと、それらに関連付けられているすべてのリソースを削除します。

```azurecli-interactive
# Remove resource group for the cluster.
az group delete --name myResourceGroup
```

## <a name="script-explanation"></a>スクリプトの説明

このスクリプトでは、次のコマンドを使用します。 表内の各コマンドは、それぞれのドキュメントにリンクされています。

| コマンド | メモ |
|---|---|
| [az group create](/cli/azure/group#az-group-create) | すべてのリソースを格納するリソース グループを作成します。 |
| [az storage account create](/cli/azure/storage/account#az-storage-account-create) | ストレージ アカウントを作成します。 |
| [az storage share create](/cli/azure/storage/share#az-storage-share-create) | ストレージ アカウントにファイル共有を作成します。 |
| [az batchai workspace create](/cli/azure/batchai/workspace#az-batchai-workspace-create) | Batch AI ワークスペースを作成します。 |
| [az batchai cluster create](/cli/azure/batchai/cluster#az-batchai-cluster-create) | Batch AI クラスターを作成します。 |
| [az batchai cluster show](/cli/azure/batchai/cluster#az-batchai-cluster-show) | Batch AI クラスターに関する情報が表示されます。 |
| [az group delete](/cli/azure/group#az-group-delete) | 入れ子になったリソースすべてを含むリソース グループを削除します。 |

## <a name="next-steps"></a>次の手順

Azure CLI の詳細については、[Azure CLI のドキュメント](https://docs.microsoft.com/cli/azure)のページをご覧ください。
