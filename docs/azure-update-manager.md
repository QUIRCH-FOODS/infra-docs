# Azure Update Manager — Maintenance Configurations

**Source subscription:** QuirchFoodsSubscription (`2ea66c5f-74a0-4844-801e-e17a7cd5c9b3`)  
**QuirchBI / QuirchDev:** No maintenance configurations found.  
**Last queried:** 2026-06-23 via QAzureReadOnlyApp

---

## Common Settings (all configurations)

| Setting | Value |
|---------|-------|
| Scope | `InGuestPatch` |
| Patch mode | `User` (customer-managed scheduling) |
| Reboot | `IfRequired` |
| Duration | 3h 55m |
| Time zone | Eastern Standard Time |
| Tag | `Application: ALL` |
| Windows patches | Critical, Security, UpdateRollup, FeaturePack, ServicePack, Definition, Tools, Updates |
| Linux patches | Critical, Security |

---

## Configurations

### Weekly Schedules

| Name | Schedule | Start Time | Resource Group | Location |
|------|----------|------------|----------------|----------|
| EveryThursday | Every Thursday | 00:00 EST | testgroup | eastus |
| EveryFriday | Every Friday | 00:00 EST | aitest | eastus |

### Monthly Schedules

| Name | Schedule | Start Time | Resource Group | Location |
|------|----------|------------|----------------|----------|
| FirstSaturdayMonth | 1st Saturday of month | 18:00 EST | aitest | eastus |
| SecondSaturdayMonth | 2nd Saturday of month | 18:00 EST | aitest | eastus |
| ThirdSaturdayMonth | 3rd Saturday of month | 18:00 EST | testgroup | eastus |
| FourthSaturdayMonth | 4th Saturday of month | 18:00 EST | testgroup | eastus2 |
| LastSaturdayMonth | Last Saturday of month | 18:00 EST | testgroup | eastus2 |

---

## VM Assignments

### QTS Servers (PRODUCTIONGROUP — QuirchFoodsSubscription)

| VM | OS | VM Agent | Patch Orchestration | Assigned Schedule | Status |
|----|----|----------|--------------------|--------------------|--------|
| QTS-01 | Windows Server 2019 Datacenter (10.0.17763.8880) | 2.7.41491.1216 — Ready | Customer Managed Schedules | TBD | Ready to assign |
| QTS-02 | Windows Server 2019 Datacenter (10.0.17763.8880) | 2.7.41491.1216 — Ready | Manual (needs change) | None | Unsupported — action required |
| QTS-03 | Windows Server 2019 Datacenter (10.0.17763.8755) | 2.7.41491.1216 — Ready | Manual (needs change) | None | Unsupported — action required |
| QTS4  | Windows Server 2019 Datacenter | — | Customer Managed Schedules | FourthSaturdayMonth | Assigned |

> **Note:** QTS-01/02/03 were built from a **custom image** (no marketplace publisher/SKU metadata). This causes Azure Update Manager to block the patch orchestration change via both portal and API with error *"The selected VM image is not supported for VM Guest patch operations"*. The workaround is to change Patch Orchestration manually from the VM's **Update settings** blade in the portal — this bypasses the image validation check. QTS-01 completed 2026-06-23. QTS-02 and QTS-03 still pending.

> **Note:** QTS VMs are in the **PRODUCTIONGROUP** resource group, not the **QTS** resource group. The QTS resource group contains only supporting resources: `QFQTS` Log Analytics workspace, `StartAutoShutdown-QTS` Automation Account, and `QTSAS` availability set.

---

## Resource IDs

```
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/aitest/providers/microsoft.maintenance/maintenanceconfigurations/everyfriday
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/testgroup/providers/microsoft.maintenance/maintenanceconfigurations/everythursday
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/aitest/providers/microsoft.maintenance/maintenanceconfigurations/firstsaturdaymonth
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/aitest/providers/microsoft.maintenance/maintenanceconfigurations/secondsaturdaymonth
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/testgroup/providers/microsoft.maintenance/maintenanceconfigurations/thirdsaturdaymonth
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/testgroup/providers/microsoft.maintenance/maintenanceconfigurations/fourthsaturdaymonth
/subscriptions/2ea66c5f-74a0-4844-801e-e17a7cd5c9b3/resourcegroups/testgroup/providers/microsoft.maintenance/maintenanceconfigurations/lastsaturdaymonth
```
