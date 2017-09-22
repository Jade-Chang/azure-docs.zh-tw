---
title: "Azure Active Directory 驗證 - Azure SQL (概觀) | Microsoft Docs"
description: "了解如何利用 SQL Database 和 SQL 資料倉儲使用 Azure Active Directory 來驗證"
services: sql-database
documentationcenter: 
author: BYHAM
manager: jhubbard
editor: 
tags: 
ms.assetid: 7e2508a1-347e-4f15-b060-d46602c5ce7e
ms.service: sql-database
ms.custom: security
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: data-management
ms.date: 08/11/2017
ms.author: rickbyh
ms.translationtype: HT
ms.sourcegitcommit: a9cfd6052b58fe7a800f1b58113aec47a74095e3
ms.openlocfilehash: c83d482eaf476388a174dfb02ae6ef53ebe921bc
ms.contentlocale: zh-tw
ms.lasthandoff: 08/12/2017

---
# <a name="use-azure-active-directory-authentication-for-authentication-with-sql-database-or-sql-data-warehouse"></a>利用 SQL Database 和 SQL 資料倉儲使用 Azure Active Directory 驗證來驗證
Azure Active Directory 驗證是 Azure Active Directory (Azure AD) 中使用身分識別連接到 Microsoft Azure SQL Database 和 [SQL 資料倉儲](../sql-data-warehouse/sql-data-warehouse-overview-what-is.md) 的機制。 您可以使用 Azure AD 驗證，在單一中央位置集中管理資料庫使用者和其他 Microsoft 服務的身分識別。 中央識別碼管理提供單一位置以管理資料庫使用者並簡化權限管理。 包括以下優點：

* 它提供 SQL Server 驗證的替代方案。
* 協助停止跨資料庫伺服器使用過多的使用者身分識別。
* 允許在單一位置的密碼替換
* 客戶可以管理使用外部 (AAD) 群組的資料庫權限。
* 它可以藉由啟用整合式 Windows 驗證和 Azure Active Directory 支援的其他形式驗證來避免儲存密碼。
* Azure AD 驗證會使用自主資料庫使用者，在資料庫層級驗證身分。
* Azure AD 針對連線到 SQL Database 的應用程式支援權杖型驗證。
* Azure AD 驗證本機 Azure Active Directory 的 ADFS (網域同盟) 或原生使用者/密碼驗證，而不需進行網域同步處理。  
* Azure AD 支援來自 SQL Server Management Studio 的連線，其中使用包含 Multi-Factor Authentication (MFA) 的 Active Directory 通用驗證。  MFA 包含增強式驗證功能，其中提供一系列簡易的驗證選項，例如電話、簡訊、含有 PIN 的智慧卡或行動應用程式通知。 如需詳細資訊，請參閱 [適用於與 SQL Database 和 SQL 資料倉儲搭配使用之 Azure AD MFA 的 SSMS 支援](sql-database-ssms-mfa-authentication.md)。  

>  [!NOTE]  
>  使用 Azure Active Directory 帳戶不支援連線到 Azure VM 上執行的 SQL Server。 請改用 Active Directory 網域帳戶。  

設定步驟包括以下設定和使用 Azure Active Directory 驗證的程序。

1. 建立和填入 Azure AD。
2. 選用：和目前與您的 Azure 訂用帳戶相關聯的 Active Directory 產生關聯並加以變更
3. 建立 Azure SQL 伺服器或 [Azure SQL 資料倉儲](https://azure.microsoft.com/services/sql-data-warehouse/)的 Azure Active Directory 系統管理員。
4. 設定用戶端電腦。
5. 在對應至 Azure AD 身分識別的資料庫中建立自主資料庫使用者。
6. 使用 Azure AD 身分識別連接到您的資料庫。

> [!NOTE]
> 若要了解如何建立和填入 Azure AD，然後使用 Azure SQL Database 和 SQL 資料倉儲設定 Azure AD，請參閱[使用 Azure SQL Database 設定 Azure AD](sql-database-aad-authentication-configure.md)。
>

## <a name="trust-architecture"></a>信認架構
下列高階圖表摘要說明搭配使用 Azure AD 驗證與 Azure SQL Database 的解決方案架構。 相同的概念適用於 SQL 資料倉儲。 若要支援 Azure AD 原生使用者密碼，只會考慮雲端部分和 Azure AD/Azure SQL Database。 若要支援同盟驗證 (或 Windows 認證的使用者/密碼)，需要與 ADFS 區塊進行通訊。 箭頭表示通訊路徑。

![aad 驗證圖表][1]

下圖表示允許用戶端藉由提交權杖連線到資料庫的同盟、信任和主控關聯性。 此權杖是由 Azure AD 所驗證，並受到資料庫信任。 客戶 1 可以代表具有原生使用者的 Azure AD 或具有同盟使用者的 Azure Active Directory。 客戶 2 代表包含已匯入使用者的可能解決方案；在此範例中，來自同盟 Azure Active Directory 且 ADFS 正與 Azure Active Directory 進行同步處理。 請務必了解使用 Azure AD 驗證存取資料庫的必要條件是裝載訂用帳戶要與 Azure AD 相關聯。 您必須使用相同的訂用帳戶來建立裝載 Azure SQL Database 或 SQL 資料倉儲的 SQL Server。

![訂用帳戶關聯性][2]

## <a name="administrator-structure"></a>系統管理員結構
使用 Azure AD 驗證時，SQL Database 伺服器會有兩個系統管理員帳戶：原始的 SQL Server 系統管理員和 Azure AD 系統管理員。 相同的概念適用於 SQL 資料倉儲。 只有以 Azure AD 帳戶為基礎的系統管理員可以在使用者資料庫中建立第一個 Azure AD 自主資料庫使用者。 Azure AD 系統管理員登入可以是 Azure AD 使用者或 Azure AD 群組。 當系統管理員是群組帳戶時，它可以供任何群組成員使用，啟用 SQL Server 執行個體的多個 Azure AD 系統管理員。 以系統管理員的身分使用群組帳戶，可讓您集中新增和移除 Azure AD 中的群組成員，而不需要變更 SQL Database 中的使用者或權限，藉以增強管理性。 一律只能設定一個 Azure AD 系統管理員 (使用者或群組)。

![系統管理員結構][3]

## <a name="permissions"></a>權限
若要建立新的使用者，您必須具有資料庫中的 `ALTER ANY USER` 權限。 任何資料庫使用者皆可授與 `ALTER ANY USER` 權限。 `ALTER ANY USER` 權限的擁有者還包括伺服器系統管理員帳戶、具備資料庫之 `CONTROL ON DATABASE` 或 `ALTER ON DATABASE` 權限的資料庫使用者，以及 `db_owner` 資料庫角色的成員。

若要在 Azure SQL Database 或 SQL 資料倉儲中建立自主資料庫使用者，您必須使用 Azure AD 身分識別連線到資料庫。 若要建立第一個自主資料庫使用者，您必須使用 Azure AD 系統管理員 (資料庫的擁有者) 連接到資料庫。 如以下步驟 4 和 5 所示。 只有針對 Azure SQL Database 或 SQL 資料倉儲伺服器建立 Azure AD 系統管理員後，才可能進行任何 Azure AD 驗證。 如果已從伺服器移除 Azure Active Directory 系統管理員，則先前在 SQL Server 內建立的現有 Azure Active Directory 使用者便無法再使用其 Azure Active Directory 認證連線到資料庫。

## <a name="azure-ad-features-and-limitations"></a>Azure AD 功能和限制
下列 Azure AD 的成員可在 Azure SQL 伺服器或 SQL 資料倉儲中佈建：

* 原生成員：在受管理網域或客戶網域的 Azure AD 中建立的成員。 如需詳細資訊，請參閱 [將您自己的網域名稱新增至 Azure AD](../active-directory/active-directory-add-domain.md)。
* 同盟網域成員：利用同盟網域在 Azure AD 中建立的成員。 如需詳細資訊，請參閱 [Microsoft Azure 現在支援 Windows Server Active Directory 的同盟](https://azure.microsoft.com/blog/2012/11/28/windows-azure-now-supports-federation-with-windows-server-active-directory/)。
* 從其他 Azure AD 匯入，且為原生網域或同盟網域成員者。
* 建立 Active Directory 群組作為安全性群組。

不支援 Microsoft 帳戶 (例如 outlook.com、hotmail.com、live.com) 或其他來賓帳戶 (例如 gmail.com、yahoo.com)。 如果您可以使用帳戶和密碼登入 [https://login.live.com](https://login.live.com) ，則您就能使用 Azure SQL Database 或 Azure SQL 資料倉儲的 Azure AD 驗證不支援的 Microsoft 帳戶。

## <a name="connecting-using-azure-ad-identities"></a>使用 Azure AD 身分識別連接

Azure Active Directory 驗證支援下列方法，使用 Azure AD 身分識別連接至資料庫：

* 使用整合式 Windows 驗證
* 使用 Azure AD 主體名稱和密碼
* 使用應用程式權杖驗證

### <a name="additional-considerations"></a>其他考量

* 若要增強管理性，建議您以系統管理員身分佈建專用的 Azure AD 群組。   
* 一個 Azure SQL 伺服器或 Azure SQL 資料倉儲一律只能設定一個 Azure AD 系統管理員 (使用者或群組)。   
* 只有 SQL Server 的 Azure AD 系統管理員可以一開始就使用 Azure Active Directory 帳戶連接到 Azure SQL 伺服器或 Azure SQL 資料倉儲。 Active Directory 系統管理員可以設定後續的 Azure AD 資料庫使用者。   
* 建議將連接逾時設定為 30 秒。   
* SQL Server 2016 Management Studio 和 SQL Server Data Tools for Visual Studio 2015 (版本 14.0.60311.1 (2016 年 4 月) 或更新版本) 支援 Azure Active Directory 驗證。 (**.NET Framework Data Provider for SqlServer** 支援 Azure AD 驗證，最低版本 .NET Framework 4.6)。 因此，這些工具和資料層應用程式 (DAC 和 .bacpac) 的最新版本可以使用 Azure AD 驗證。   
* [ODBC 13.1 版](https://www.microsoft.com/download/details.aspx?id=53339)支援 Azure Active Directory 驗證，不過，`bcp.exe` 無法使用 Azure Active Directory 驗證進行連線，因為它們使用較舊的 ODBC 提供者。   
* `sqlcmd` 從 13.1 版 (可從 [下載中心](http://go.microsoft.com/fwlink/?LinkID=825643)取得) 開始即支援 Azure Active Directory 驗證。   
* SQL Server Data Tools for Visual Studio 2015 至少需要 2016 年 4 月版本的 Data Tools (版本 14.0.60311.1)。 Azure AD 使用者目前不會顯示在 SSDT 物件總管中。 解決方法是在 [sys.database_principals](https://msdn.microsoft.com/library/ms187328.aspx) 中檢視使用者。   
* [Microsoft JDBC Driver 6.0 for SQL Server](https://www.microsoft.com/download/details.aspx?id=11774) 支援 Azure AD 驗證。 此外，請參閱 [設定連接屬性](https://msdn.microsoft.com/library/ms378988.aspx)。   
* PolyBase 無法使用 Azure AD 驗證進行驗證。   
* Azure 入口網站的 [匯入資料庫] 和 [匯出資料庫] 刀鋒視窗支援 SQL Database 的 AD 驗證。 PowerShell 命令也支援使用 Azure AD 驗證的匯入和匯出。   
* SQL Database 和 SQL 資料倉儲 (使用 CLI) 支援 Azure AD 驗證。 如需詳細資訊，請參閱[使用 SQL Database 或 SQL 資料倉儲設定和管理 Azure Active Directory 驗證](sql-database-aad-authentication-configure.md)和 [SQL Server - az sql server](https://docs.microsoft.com/en-us/cli/azure/sql/server)。

## <a name="next-steps"></a>後續步驟
- 若要了解如何建立和填入 Azure AD，然後使用 Azure SQL Database 或 Azure SQL 資料倉儲來設定 Azure AD，請參閱[使用 SQL Database 或 SQL 資料倉儲設定和管理 Azure Active Directory 驗證](sql-database-aad-authentication-configure.md)。
- 如需 SQL Database 中存取權和控制權的概觀，請參閱 [SQL Database 的存取權和控制權](sql-database-control-access.md)。
- 如需 SQL Database 中登入、使用者和資料庫角色的概觀，請參閱[登入、使用者和資料庫角色](sql-database-manage-logins.md)。
- 如需資料庫主體的詳細資訊，請參閱[主體](https://msdn.microsoft.com/library/ms181127.aspx)。
- 如需資料庫角色的詳細資訊，請參閱[資料庫角色](https://msdn.microsoft.com/library/ms189121.aspx)。
- 如需 SQL Database 中防火牆規則的詳細資訊，請參閱 [SQL Database 防火牆規則](sql-database-firewall-configure.md)。
- 如需技術影片，請參閱 [深入研究 Windows Server 2016 Active Directory] (https://channel9.msdn.com/Series/Microsoft-IT-Friday/Windows-Server-2016-Active-Directory1)。

<!--Image references-->

[1]: ./media/sql-database-aad-authentication/1aad-auth-diagram.png
[2]: ./media/sql-database-aad-authentication/2subscription-relationship.png
[3]: ./media/sql-database-aad-authentication/3admin-structure.png
[4]: ./media/sql-database-aad-authentication/4select-subscription.png
[5]: ./media/sql-database-aad-authentication/5ad-settings-portal.png
[6]: ./media/sql-database-aad-authentication/6edit-directory-select.png
[7]: ./media/sql-database-aad-authentication/7edit-directory-confirm.png
[8]: ./media/sql-database-aad-authentication/8choose-ad.png
[9]: ./media/sql-database-aad-authentication/9ad-settings.png
[10]: ./media/sql-database-aad-authentication/10choose-admin.png
[11]: ./media/sql-database-aad-authentication/11connect-using-int-auth.png
[12]: ./media/sql-database-aad-authentication/12connect-using-pw-auth.png
[13]: ./media/sql-database-aad-authentication/13connect-to-db.png


