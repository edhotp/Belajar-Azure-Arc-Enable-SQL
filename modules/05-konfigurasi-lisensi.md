# Modul 05 — Konfigurasi & Lisensi

> 📚 Sumber utama: [Configure SQL Server enabled by Azure Arc](https://learn.microsoft.com/sql/sql-server/azure-arc/manage-configuration) · [Manage licensing and billing](https://learn.microsoft.com/sql/sql-server/azure-arc/manage-license-billing) · [Feature availability by license type](https://learn.microsoft.com/sql/sql-server/azure-arc/overview#feature-availability-depending-on-license-type)

Setelah SQL Server tersambung ke Arc, langkah berikutnya adalah **mengatur license type** dan opsi konfigurasi.

## 5.1 Tipe Lisensi

| License Type | Deskripsi | Fitur premium aktif? |
|--------------|-----------|----------------------|
| **Paid** | Anda punya lisensi SQL Server **dengan Software Assurance (SA)** | Ya |
| **PAYG** (Pay-as-you-go) | Bayar per core/jam ditagih melalui Azure | Ya |
| **LicenseOnly** | Lisensi tanpa SA (mis. perpetual) | Tidak (fitur terbatas) |
| **Configuration needed** | Belum diisi onboarding | — (perlu dilengkapi) |

> Fitur seperti **Backup otomatis, Best Practices Assessment, Defender for Cloud, Microsoft Entra Auth, Migration to Azure SQL MI** umumnya **butuh Paid atau PAYG**.

### Set License Type via Azure CLI

```azurecli
az connectedmachine extension update \
  --machine-name "SRV-SQL01" \
  --resource-group "rg-arc-sql" \
  --name "WindowsAgent.SqlServer" \
  --type "WindowsAgent.SqlServer" \
  --publisher "Microsoft.AzureData" \
  --settings '{"LicenseType":"PAYG", "SqlManagement": {"IsEnabled":true}}'
```

> ⚠️ **Penting**: perintah `update` **menimpa seluruh settings**. Jika sebelumnya ada `ExcludedSqlInstances`, sertakan kembali dalam payload.

### Set License Type via PowerShell

```powershell
$rg = "rg-arc-sql"; $machine = "SRV-SQL01"
Set-AzConnectedMachineExtension -ResourceGroupName $rg `
  -MachineName $machine -Name "WindowsAgent.SqlServer" `
  -Location "southeastasia" `
  -Publisher "Microsoft.AzureData" `
  -ExtensionType "WindowsAgent.SqlServer" `
  -Setting @{ "LicenseType" = "PAYG"; "SqlManagement" = @{ "IsEnabled" = $true } }
```

> Lihat juga [script `modify-license-type` di GitHub](https://github.com/microsoft/sql-server-samples/tree/master/samples/manage/azure-arc-enabled-sql-server/modify-license-type) untuk update massal.

### Resource Graph: cek license type semua instance

```kusto
resources
| where type == 'microsoft.hybridcompute/machines/extensions'
| where name endswith 'SqlServer'
| project machine = tostring(split(id,'/')[8]),
          licenseType = tostring(properties.settings.LicenseType)
```

## 5.2 Pay-as-you-go (PAYG)

Konsep PAYG:

```mermaid
flowchart LR
    SQL[SQL Instance Active] -- usage data --> Ext[SQL Extension]
    Ext -- per jam --> DPS
    DPS --> Bill[Azure Commerce]
    Bill --> Invoice[Tagihan Azure]
```

- Tersedia untuk SQL Server 2012–2025.
- **Linux**: tidak ada fitur passive instance detection & connected user verification — semua instance ditagih sebagai active.
- Cocok untuk workload variabel, lab, atau pemakaian jangka pendek.

## 5.3 Extended Security Updates (ESU)

Jika SQL Server Anda sudah **end-of-support** (mis. SQL 2012 setelah 2022), Anda bisa berlangganan ESU **gratis** selama instance ke-Arc-kan.

- Setelah upgrade SQL → ESU otomatis berhenti.
- Setelah migrasi ke Azure SQL → biaya ESU berhenti, tapi Anda tetap dapat update.

Aktifkan via portal: pilih SQL Server – Arc → **Extended Security Updates → Enable**.

## 5.4 Setting Penting Lain

| Setting | Default | Lokasi konfigurasi |
|---------|---------|--------------------|
| Extended Security Updates | Disabled | Extension |
| Least privilege mode | Disabled | Extension |
| Automated patching | Disabled | Extension |
| Best Practices Assessment | Disabled | Extension |
| Microsoft Entra Authentication | Disabled | Extension per instance |
| Microsoft Purview integration | Disabled | Extension + Instance |
| Automated backups | Disabled | Instance / Database |
| Performance metrics (preview) | Enabled | Instance |
| Migration assessment | Enabled | Instance |
| Database migration | Enabled | Extension |
| AG discovery | Enabled | Feature flag |
| Inventory discovery | Enabled | Tidak bisa diubah |

## 5.5 Least Privilege Mode (Recommended)

Daripada extension berjalan sebagai `Local System` (Windows) atau `root` (Linux), gunakan **`NT Service\SQLServerExtension`** dengan permission minimal. Detail: [Operate with least privilege](https://learn.microsoft.com/sql/sql-server/azure-arc/configure-least-privilege).

```mermaid
flowchart LR
    A[Default: Local System / root] -- Enable Least Privilege --> B[NT Service\\SQLServerExtension]
    B --> C[Permission minimal di OS & SQL]
    C --> D[Permission auto-revoke saat fitur disable]
```

> Catatan dari Microsoft: *“Currently, least privileged configuration is not applied by default.”* Server dengan extension versi `1.1.2859.223` ke atas (rilis November 2024) **eventually** akan menerapkan least privilege.

Aktifkan via portal SQL Server – Arc → **Configuration → Least privilege**. Untuk setting via PowerShell/CLI dan langkah verifikasi, ikuti [docs Configure Least Privilege](https://learn.microsoft.com/sql/sql-server/azure-arc/configure-least-privilege).

## 5.6 Verifikasi Konfigurasi

```bash
az resource show \
  --ids "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.AzureArcData/sqlServerInstances/<srv>_MSSQLSERVER" \
  --query "properties"
```

Pastikan property `licenseType`, `version`, `edition`, dll. sudah benar.

---

⬅️ [Modul 04](04-onboarding.md) · ➡️ [Modul 06 — Fitur Manajemen](06-fitur-manajemen.md)
