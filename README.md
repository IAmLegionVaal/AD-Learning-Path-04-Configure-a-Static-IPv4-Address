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

## Procedure
1. Identify the connected adapter instead of assuming it is named `Ethernet`.
2. Disable DHCP, remove any old non-link-local IPv4 address, and assign `10.10.10.10/24` with gateway `10.10.10.1`.
3. Before promotion, configure one approved temporary resolver only if updates require Internet name resolution.
4. After Activity 07, replace the temporary resolver with `10.10.10.10`; domain controllers and clients must use internal AD DNS.

## Implementation
```powershell
$Adapter = Get-NetAdapter | Where-Object Status -eq 'Up' | Sort-Object ifIndex | Select-Object -First 1
if (-not $Adapter) { throw 'No connected adapter found.' }

Set-NetIPInterface -InterfaceIndex $Adapter.ifIndex -AddressFamily IPv4 -Dhcp Disabled
Get-NetIPAddress -InterfaceIndex $Adapter.ifIndex -AddressFamily IPv4 -ErrorAction SilentlyContinue |
    Where-Object PrefixOrigin -ne 'WellKnown' | Remove-NetIPAddress -Confirm:$false

New-NetIPAddress -InterfaceIndex $Adapter.ifIndex -IPAddress '10.10.10.10' `
    -PrefixLength 24 -DefaultGateway '10.10.10.1'
Set-DnsClientServerAddress -InterfaceIndex $Adapter.ifIndex -ServerAddresses '1.1.1.1'
Rename-NetAdapter -Name $Adapter.Name -NewName 'LAN'
```

## Validation
```powershell
Get-NetIPConfiguration -InterfaceAlias LAN
Get-NetIPAddress -InterfaceAlias LAN -AddressFamily IPv4
Get-DnsClientServerAddress -InterfaceAlias LAN -AddressFamily IPv4
Test-NetConnection 10.10.10.1
Resolve-DnsName learn.microsoft.com
```

## Evidence
Capture the IP configuration, DNS server list, gateway test, network settings screenshot, actual command output, deviations, errors, remediation, and final pass/fail result. Store sanitized evidence under `evidence/`.

## Troubleshooting
- Duplicate IP warning: stop and locate the conflict; never ignore it on a domain controller.
- Gateway unreachable: verify switch attachment and host NAT adapter.
- DNS failure with working IP connectivity: test the configured resolver separately.

## Security notes
Keep Windows Firewall enabled. Remove public DNS after promotion because public resolvers do not host AD service records.

## Rollback
Return to DHCP with `Set-NetIPInterface -InterfaceAlias LAN -AddressFamily IPv4 -Dhcp Enabled` and reset DNS only when dismantling the lab.

## References
- Microsoft Learn: `New-NetIPAddress`
- Microsoft Learn: `Set-DnsClientServerAddress`

## Next activity
`AD-Learning-Path-05-Rename-and-Patch-the-Server`
