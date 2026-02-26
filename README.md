# LogAnalyticsTLS

Azure Resource Graph (ARG) queries to identify Virtual Machines and Azure Arc machines that may be using **TLS 1.0 or 1.1** for Log Analytics agent ingestion.

These queries support the TLS 1.2 enforcement initiative documented by Microsoft:  
[Best practices for security in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/best-practices-security?tabs=vms-os#recommended-action)

## Queries

| File | Scope | Output |
|------|-------|--------|
| `azure-aggregate.qry` | Azure VMs (`microsoft.compute/virtualmachines`) | Grouped by OS/version/agent with VM count |
| `azure-detailed.qry` | Azure VMs | Per-VM listing with subscription, resource group, VM name |
| `arc-aggregate.qry` | Azure Arc machines (`microsoft.hybridcompute/machines`) | Grouped by OS/version/agent with machine count |
| `arc-detailed.qry` | Azure Arc machines | Per-machine listing with subscription, resource group, machine name |

## LowTLS Classification

Each query produces a `LowTLS` column:

| Value | Meaning |
|-------|---------|
| `true` | OS is known to **not support TLS 1.2** natively — requires remediation or decommission |
| `false` | OS supports TLS 1.2 — no action needed |
| `unknown` | OS could not be determined from available metadata |

### OS versions flagged as `LowTLS = true`

Per Microsoft's documented criteria:

- **Windows**: XP, Vista, Server 2003, Server 2008 *(non-R2 only)*
- **CentOS**: 5.x and earlier
- **RHEL**: 5.x and earlier
- **Ubuntu**: 12.x and earlier

> **Note:** Windows Server 2008 **R2** supports TLS 1.2 natively (disabled by default) and is classified as `false`.

## How It Works

### Azure VM queries

OS detection uses multiple data sources in priority order:

1. **InstanceView** (`properties.extended.instanceView.osName/osVersion`) — available for running VMs
2. **Marketplace image** (`imageReference.publisher/offer/sku`) — parsed for OS family and version
3. **Shared Image Gallery** (`imageReference.id`) — image name extracted and pattern-matched
4. **OS disk name** — last resort fallback (often yields `unknown`)

### Arc machine queries

OS detection is straightforward since Arc machines self-report:

1. **osSku** (`properties.osSku`) — descriptive string like *"Windows Server 2019 Standard"* or *"Red Hat Enterprise Linux 8.10 (Ootpa)"*
2. **osName** (`properties.osName`) — fallback if osSku is empty
3. **Kernel version** (`properties.osVersion`) — `.el` tag extraction for RHEL/CentOS identification

### Agent detection

Both Azure VM and Arc queries detect the monitoring agent type:

| Agent | Publisher | Extensions |
|-------|-----------|------------|
| **AMA** (Azure Monitor Agent) | `Microsoft.Azure.Monitor` | `AzureMonitorWindowsAgent`, `AzureMonitorLinuxAgent` |
| **MMA** (Microsoft Monitoring Agent) | `Microsoft.EnterpriseCloud.Monitoring` | `MicrosoftMonitoringAgent`, `OmsAgentForLinux` |

VMs/machines with both agents installed are reported as `AMA+MMA`.

## Usage

1. Open the [Azure Resource Graph Explorer](https://portal.azure.com/#blade/HubsExtension/ArgQueryBlade)
2. Paste the contents of any `.qry` file
3. Select the target scope (management group or subscriptions)
4. Run the query

Results can be exported to CSV for further analysis.

## Known Limitations

- **Stopped Azure VMs** without a marketplace or SIG image reference fall back to OS disk name, which typically produces `unknown` results. Start the VM or onboard it to Azure Arc for proper detection.
- **OpenShift cluster nodes** appear as `unknown` because their OS disk names are cluster hash IDs. These run Red Hat CoreOS which supports TLS 1.2.
- **MMA was deprecated on August 31, 2024.** VMs still running MMA-only should be migrated to AMA regardless of TLS status.
