# Windows VPN Gateway

This is a how to on setting up a Windows VM so that it acts as a Gateway for VPNs. You can connect to this Gateway using the Wireguard VPN protocol.

In this example the Azure VPN Client is used as an example, but it's easy to adapt to other less trustworthy VPN clients like
Cisco. Note that Azure VPN is a flavour of OpenVPN. So there's a chance that you can run it through NetworkManager, but keep in 
mind, that certain features like AAD auth are not supported by a vanilla OpenVPN.

## Prerequisites

- A Windows VM connected to the internet.
- An Azure VPN endpoint configuration. It can be downloaded from the Azure Portal.

## IP setup

The following IP addresses are assumed. Change the to your setup:

| Placeholder | CIDR/IP/Port        | Description |
|-----|----------------|-------------|
| {NETWORK} | 10.8.55.0/24   | Network of your Wireguard Network that is used for routing |
| {WG_WINVM} | 10.8.55.1      | The wireguard IP address of your windows VM |
| {WG_LOCAL} | 10.8.55.2      | The wireguard IP address of your client VM |
| {WG_PORT} | 50505 | the port that wireguard uses for communication |
| {WINVM} | 192.168.1.55   | The IP address of the Windows VM |
| {REMOTE_NET} | 10.11.0.0/16, 172.0.0.0/16     | the subnets that are routed through the Azure VPN tunnel |

## Install Wireguard

Go to [the Wireguard Download Page](https://www.wireguard.com/install/) and download the newest wireguard version for windows

Install the downloaded package

## Install Azure VPN

Go to the Microsoft Store Application. It should already be available.

Search for `Azure VPN Client` and install it.

## Configure Wireguard

On your host generate two new wireguard keys and their public key counterparts. Also generate a pre shared key:

```bash
WG_KEY_LOCAL=$(wg genkey)
WG_KEY_WINVM=$(wg genkey)
WG_PUB_LOCAL=$(echo "$WG_KEY_LOCAL" | wg pubkey)
WG_PUB_WINVM=$(echo "$WG_KEY_WINVM" | wg pubkey)
WG_PSK=$(wg genpsk)
```


### local wireguard config

For the local wireguard config use the following file:

Note: We don't need to allow the remote IP of the windows host
(`{WG_WINVM}`, 10.8.55.1/24). This would only be needed if we want to access the Windows VM directly through the tunnel, which isn't needed.

```ini
[Interface]
; Address = {WG_LOCAL}/24
Address = 10.8.55.2/32
; replace with $WG_KEY_LOCAL
PrivateKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=

[Peer]
; replace with $WG_PUB_WINVM
PublicKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
; replace with $WG_PSK
PresharedKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
; AllowedIPs = {REMOTE_NET}
AllowedIPs = 10.11.0.0/16, 172.0.0.0/16
; Endpoint = {WINVM}:{WG_PORT}
Endpoint = 192.168.1.55:50505
```

### windows wireguard config

For the Windows VM use the following configuration


Note: On the windows site, we use a `/24` subnet, which quite large for
a point-to-point connection. Nevertheless, I ran into issues when using
smaller subnets on windows. Also on the local site, only the wireguard
address is routed (`{WG_WINVM}`), so it isn't a huge issue.

```ini
[Interface]
; Address = {WG_WINVM}/24
Address = 10.8.55.1/24
; replace with $WG_KEY_WINVM
PrivateKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
; ListenPort = {WG_PORT}
ListenPort = 50505

[Peer]
; replace with $WG_PUB_LOCAL
PublicKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
; replace with $WG_PSK
PresharedKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
; AllowedIPs = {WG_LOCAL}/32
AllowedIPs = 10.8.55.2/32
```

## Configure Azure VPN

`TODO`

## Setup NAT

open a powershell prompt as administrator. First we enable ip routing in windows:
```powershell
Set-Itemproperty -path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -value '1'
```

Second - and this seems to be only needed on Windows 10/11 - we need to enable Hyper-V in order to enable the routing subsystem.
Otherwise commands like `New-NetNat` will fail
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

You'll be prompted to reboot. Select `Yes`.

After the reboot, open a new powershell prompt as administrator. Now we can enable the NAT:

```powershell
New-NetNat -Name 'wg-gw' -InternalIPInterfaceAddressPrefix '10.8.55.0/24'
# New-NetNat -Name 'wg-gw' -InternalIPInterfaceAddressPrefix '{NETWORK}'
```

## Advice

I'm not a windows administrator. Therefore I'm not sure if the changes (especially enabling the routing)
enables any attacks. Keep the Windows VM in a secure zone and setup a proper firewall in front of it.