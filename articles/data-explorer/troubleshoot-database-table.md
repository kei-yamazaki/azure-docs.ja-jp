---
title: Azure データ エクスプローラーでデータベースまたはテーブルの作成または削除が失敗する
description: この記事では、Azure データ エクスプローラーでのデータベースおよびテーブルの作成と削除に関するトラブルシューティング手順について説明します。
author: orspod
ms.author: v-orspod
ms.reviewer: mblythe
ms.service: data-explorer
services: data-explorer
ms.topic: conceptual
ms.date: 09/24/2018
ms.openlocfilehash: 6ec81d6154f15d1c49428b50f0e65eed8edcedad
ms.sourcegitcommit: 32d218f5bd74f1cd106f4248115985df631d0a8c
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 09/24/2018
ms.locfileid: "46986517"
---
# <a name="troubleshoot-failure-to-create-or-delete-a-database-or-table-in-azure-data-explorer"></a>トラブルシューティング: Azure データ エクスプローラーでデータベースまたはテーブルの作成または削除が失敗する

Azure データ エクスプローラーでは、データベースやテーブルの作業をよく行います。 この記事では、発生する可能性のある問題のトラブルシューティング手順を示します。

## <a name="creating-a-database"></a>データベースを作成する

1. 適切なアクセス許可を持っていることを確認します。 データベースを作成するには、Azure サブスクリプションの "*共同作成者*" または "*所有者*" ロールのメンバーである必要があります。 必要であれば、サブスクリプション管理者に問い合わせて、適切なロールに追加してもらいます。

1. データベース名に関して名前の検証エラーがないことを確認します。 名前は英数字で、260 文字以下である必要があります。

1. データベースの保有期間とキャッシュ値が許容範囲内であることを確認します。 保有期間は 1 ～ 36,500 日 (100 年) にする必要があります。 キャッシュは、1 から保有期間に対して設定されている最大値の範囲にする必要があります。

## <a name="deleting-or-renaming-a-database"></a>データベースの削除または名前変更

適切なアクセス許可を持っていることを確認します。 データベースを削除または名前変更するには、Azure サブスクリプションの "*共同作成者*" または "*所有者*" ロールのメンバーである必要があります。 必要であれば、サブスクリプション管理者に問い合わせて、適切なロールに追加してもらいます。

## <a name="creating-a-table"></a>テーブルの作成

1. 適切なアクセス許可を持っていることを確認します。 テーブルを作成するには、データベースの "*データベース管理者*" または "*データベース ユーザー*" ロールのメンバー、または Azure サブスクリプションの "*共同作成者*" または "*所有者*" ロールのメンバーである必要があります。 必要であれば、サブスクリプションまたはクラスターの管理者に問い合わせて、適切なロールに追加してもらいます。

    アクセス許可について詳しくは、「[データベースのアクセス許可を管理する](manage-database-permissions.md)」をご覧ください。

1. 同じ名前のテーブルが既に存在しないことを確認します。 存在する場合は、別の名前でテーブルを作成するか、既存のテーブルの名前を変更するか ("*テーブル管理者*" ロールが必要)、既存のテーブルを削除する ("*データベース管理者*" ロールが必要) ことができます。 次のコマンドを使用します。

    ```Kusto
    .drop table <TableName>

   .rename table <OldTableName> to <NewTableName>
    ```

## <a name="deleting-or-renaming-a-table"></a>テーブルの削除または名前変更

適切なアクセス許可を持っていることを確認します。 テーブルの削除または名前変更を行うには、データベースの "*データベース管理者*" または "*テーブル管理者*" ロールのメンバーである必要があります。 必要であれば、サブスクリプションまたはクラスターの管理者に問い合わせて、適切なロールに追加してもらいます。

アクセス許可について詳しくは、「[データベースのアクセス許可を管理する](manage-database-permissions.md)」をご覧ください。

## <a name="general-guidance"></a>一般的なガイダンス

1. [Azure サービス正常性ダッシュボード](https://azure.microsoft.com/status/>)を確認します。 データベースまたはテーブルの作業を行おうとしているリージョンでの Azure データ エクスプローラーの状態を探します。

    状態が **[良好]** (緑色のチェック マーク) でない場合は、状態が改善されてからもう一度試します。

1. 問題解決にさらに手助けが必要な場合は、[Azure portal](https://portal.azure.com) でサポート リクエストを作成してください。

## <a name="next-steps"></a>次の手順

[クラスターの正常性を確認する](check-cluster-health.md)