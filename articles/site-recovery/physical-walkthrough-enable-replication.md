---
title: "使用 Azure Site Recovery 來啟用將實體伺服器複寫至 Azure 的複寫功能 | Microsoft Docs"
description: "摘要說明使用 Azure Site Recovery 服務來啟用將實體伺服器複寫至 Azure 的複寫功能時所需的步驟"
documentationcenter: 
author: rayne-wiselman
manager: carmonm
editor: 
ms.assetid: 519c5559-7032-4954-b8b5-f24f5242a954
ms.service: site-recovery
ms.workload: storage-backup-recovery
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 06/27/2017
ms.author: raynew
ms.translationtype: Human Translation
ms.sourcegitcommit: 138f04f8e9f0a9a4f71e43e73593b03386e7e5a9
ms.openlocfilehash: 42f35c53eec06a346281fd90c97aecfd2269307d
ms.contentlocale: zh-tw
ms.lasthandoff: 06/29/2017


---
# <a name="step-10-enable-replication-for-physical-servers-to-azure"></a>步驟 10：啟用將實體伺服器複寫至 Azure 的複寫功能


本文說明如何在 Azure 入口網站中使用 [Azure Site Recovery](site-recovery-overview.md) 服務，來啟用將內部部署 Windows/Linux 實體伺服器複寫至 Azure 的複寫功能。

請在本文下方或 [Azure 復原服務論壇](https://social.msdn.microsoft.com/forums/azure/home?forum=hypervrecovmgr)上張貼意見或問題。


## <a name="before-you-start"></a>開始之前

- 伺服器必須[已安裝行動服務元件](physical-walkthrough-install-mobility.md)。
- 如果已準備好 VM 進行推入安裝，當您啟用複寫時，處理序伺服器會自動安裝行動服務。
- 啟用機器的複寫功能時，您的 Azure 使用者帳戶必須具有特定[權限](site-recovery-role-based-linked-access-control.md#permissions-required-to-enable-replication-for-new-virtual-machines)，以確保您能夠使用 Azure 儲存體及建立 Azure VM。
- 當您新增或修改伺服器的 IP 位址時，可能需要 15 分鐘或更久，變更才會生效，也才會出現在入口網站中。


## <a name="exclude-disks-from-replication"></a>從複寫排除磁碟

依預設會複寫電腦上的所有磁碟。 您可以從複寫排除磁碟。 例如，您可能不想要複寫具有暫存資料的磁碟，或是每次機器或應用程式重新啟動時便重新整理的資料 (例如 pagefile.sys 或 SQL Server tempdb)。 [深入了解](site-recovery-exclude-disk.md)

## <a name="replicate-servers"></a>複寫伺服器

1. 按一下 [步驟 2: 複寫應用程式]  >  [來源]。
2. 在 [來源] 中，選取設定伺服器。
3. 在 [機器類型] 中，選取 [實體機器]。
4. 選取處理序伺服器。 如果您尚未建立任何額外的處理序伺服器，則這會成為設定伺服器。 然後按一下 [確定] 。
5. 在 [目標] 中，選取您想要在其中建立容錯移轉 VM 的訂用帳戶和資源群組。 選擇您想要在 Azure (傳統或資源管理) 中，針對容錯移轉 VM 使用的部署模型。
6. 選取您要用來複寫資料的 Azure 儲存體帳戶。 如果您不想要使用已設定的帳戶，您可以建立新的帳戶。
7. 選取 Azure VM 在容錯移轉後所要連線的 **Azure 網路**和**子網路**。 選取 [立即設定選取的機器]，將網路設定套用至您選取要進行保護的所有機器。 選取 [稍後設定] 以選取每部機器的 Azure 網路。 如果您不想使用現有的網路，您可以建立網路。

    ![啟用複寫](./media/physical-walkthrough-enable-replication/targetsettings.png)

8. 在**實體機器**中，按一下 [+實體機器]，然後輸入**名稱**和 **IP 位址**。 選擇您想要複寫之電腦的作業系統。 需要花費數分鐘的時間，才能探索到電腦並加以顯示於清單中。
9. 在 [名稱]  > 中，選取處理序伺服器將用來在機器上自動安裝行動服務的帳戶。
10. 依預設會複寫所有磁碟。 按一下 [所有磁碟]  ，然後將任何您不想要複寫的磁碟取消選取。 然後按一下 [確定] 。 您可以稍後再設定其他 VM 屬性。

    ![啟用複寫](./media/physical-walkthrough-enable-replication/enable-replication6.png)
11. 在 [複寫設定] > [設定複寫設定] 中，確認已選取正確的複寫原則。 如果您修改原則，變更會套用至複寫電腦及新的電腦。
12. 如果您想要將機器聚集成一個複寫群組，請啟用 [多部 VM 一致性]  ，並指定群組的名稱。 然後按一下 [確定] 。 請注意：

    * 複寫群組中的電腦會一起複寫，並且在容錯移轉時會有共用的損毀一致和應用程式一致的復原點。
    * 建議您將實體伺服器收集在一起，以便它們能夠反映您的工作負載。 啟用多部 VM 一致性可能會影響工作負載的效能，應該只用於機器正在執行相同工作負載，且您需要一致性的情況。

13. 按一下 [啟用複寫] 。 您可以在 [設定]  >  [作業]  >  [Site Recovery 作業] 中，追蹤 [啟用保護] 作業的進度。 執行 [完成保護]  作業之後，機器即準備好進行容錯移轉。

## <a name="next-steps"></a>後續步驟

移至[步驟 11：執行測試容錯移轉](physical-walkthrough-test-failover.md)

