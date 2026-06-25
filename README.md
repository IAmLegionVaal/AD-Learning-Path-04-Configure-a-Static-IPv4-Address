# AD Learning Path 04 — Configure a Static IPv4 Address

> **Level:** Beginner  
> **Series:** Activity 4 of 64  
> **Scope:** One standalone Active Directory lab activity.

## Objective
Configure `DC01` with static address `10.10.10.10/24`, gateway `10.10.10.1`, and a documented DNS transition from temporary external resolution to internal AD DNS.

## Prerequisites
- Installed `DC01` VM
- Connected `AD-LAB-NAT` adapter
- Activity 01 address plan
- Console access in case the selected interface is incorrect

## Procedure
1. Identify the intended lab adapter by name, MAC address and switch attachment.
2. Set the exact adapter alias in the script. Never select the first connected interface on a multi-NIC server.
3. Export the current interface configuration before making changes.
4. Remove only the existing IPv4 addresses and default routes assigned to that selected interface.
5. Disable DHCP and assign `10.10.10.10/24` with gateway `10.10.10.1`.
6. Before promotion, configure one approved temporary resolver only when updates require Internet name resolution.
7. After Activity 07, replace the temporary resolver with `10.10.10.10`; domain controllers and clients must use internal AD DNS.

## Implementation

Review and set `$InterfaceAlias` before running this from the server console:

```powershell
#requires -RunAsAdministrator

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string]$InterfaceAlias
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$IPAddress = '10.10.10.10'
$PrefixLength = 24
$DefaultGateway = '10.10.10.1'
$TemporaryDnsServer = '1.1.1.1'
$BackupPath = Join-Path $env:ProgramData 'ADLab-Network-PreChange.xml'

$Adapter = Get-NetAdapter -Name $InterfaceAlias -ErrorAction Stop
if ($Adapter.Status -ne 'Up') {
    throw "Adapter '$InterfaceAlias' is not connected. Current status: $($Adapter.Status)"
}

Get-NetIPConfiguration -InterfaceIndex $Adapter.ifIndex |
    Export-Clixml -LiteralPath $BackupPath -Force

if ($PSCmdlet.ShouldProcess($InterfaceAlias, 'Configure static IPv4 and DNS settings')) {
    Get-NetIPAddress -InterfaceIndex $Adapter.ifIndex -AddressFamily IPv4 `
        -ErrorAction SilentlyContinue |
        Where-Object {
            $_.PrefixOrigin -ne 'WellKnown' -and
            $_.IPAddress -ne $IPAddress
        } |
        Remove-NetIPAddress -Confirm:$false -ErrorAction Stop

    Get-NetRoute -InterfaceIndex $Adapter.ifIndex -AddressFamily IPv4 `
        -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue |
        Remove-NetRoute -Confirm:$false -ErrorAction Stop

    Set-NetIPInterface -InterfaceIndex $Adapter.ifIndex `
        -AddressFamily IPv4 -Dhcp Disabled -ErrorAction Stop

    $ExistingTargetAddress = Get-NetIPAddress -InterfaceIndex $Adapter.ifIndex `
        -AddressFamily IPv4 -IPAddress $IPAddress -ErrorAction SilentlyContinue

    if (-not $ExistingTargetAddress) {
        New-NetIPAddress -InterfaceIndex $Adapter.ifIndex `
            -IPAddress $IPAddress `
            -PrefixLength $PrefixLength `
            -DefaultGateway $DefaultGateway `
            -ErrorAction Stop | Out-Null
    }

    Set-DnsClientServerAddress -InterfaceIndex $Adapter.ifIndex `
        -ServerAddresses $TemporaryDnsServer -ErrorAction Stop
}

Write-Host "Pre-change configuration: $BackupPath"
```

After the server is promoted in Activity 07:

```powershell
$InterfaceAlias = 'LAN'
Set-DnsClientServerAddress -InterfaceAlias $InterfaceAlias `
    -ServerAddresses '10.10.10.10'
Register-DnsClient
```

Rename the adapter only after the static configuration has been validated:

```powershell
Rename-NetAdapter -Name '<current-lab-adapter-name>' -NewName 'LAN'
```

Replace the quoted example with the exact verified adapter name.

## Validation

```powershell
$InterfaceAlias = 'LAN'

Get-NetIPConfiguration -InterfaceAlias $InterfaceAlias
Get-NetIPAddress -InterfaceAlias $InterfaceAlias -AddressFamily IPv4
Get-DnsClientServerAddress -InterfaceAlias $InterfaceAlias -AddressFamily IPv4
Test-NetConnection -ComputerName '10.10.10.1'
Resolve-DnsName -Name 'learn.microsoft.com'
```

## Evidence
Capture the selected adapter identity, pre-change export, IP configuration, DNS server list, gateway test, actual command output, deviations, errors, remediation, and final pass/fail result. Store sanitized evidence under `evidence/`.

## Troubleshooting
- Duplicate IP warning: stop and locate the conflict; never ignore it on a domain controller.
- Gateway unreachable: verify the exact adapter, virtual-switch attachment and host NAT configuration.
- DNS failure with working IP connectivity: test the configured resolver separately.
- Lost connectivity: use console access and execute the rollback procedure against the same adapter.

## Security notes
Keep Windows Firewall enabled. Remove public DNS after promotion because public resolvers do not host AD service records.

## Rollback

This rollback removes manual IPv4 addresses and default routes only from the explicitly selected adapter, restores DHCP, and resets DNS to DHCP-provided values:

```powershell
#requires -RunAsAdministrator

$InterfaceAlias = 'LAN'
$Adapter = Get-NetAdapter -Name $InterfaceAlias -ErrorAction Stop

Get-NetIPAddress -InterfaceIndex $Adapter.ifIndex -AddressFamily IPv4 `
    -ErrorAction SilentlyContinue |
    Where-Object { $_.PrefixOrigin -eq 'Manual' } |
    Remove-NetIPAddress -Confirm:$false -ErrorAction Stop

Get-NetRoute -InterfaceIndex $Adapter.ifIndex -AddressFamily IPv4 `
    -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue |
    Remove-NetRoute -Confirm:$false -ErrorAction Stop

Set-NetIPInterface -InterfaceIndex $Adapter.ifIndex `
    -AddressFamily IPv4 -Dhcp Enabled -ErrorAction Stop
Set-DnsClientServerAddress -InterfaceIndex $Adapter.ifIndex `
    -ResetServerAddresses -ErrorAction Stop
```

Use rollback only when dismantling the lab or correcting an incorrectly selected interface.

## References
- Microsoft Learn: `New-NetIPAddress`
- Microsoft Learn: `Set-DnsClientServerAddress`
- Microsoft Learn: `Set-NetIPInterface`

## Next activity
`AD-Learning-Path-05-Rename-and-Patch-the-Server`
