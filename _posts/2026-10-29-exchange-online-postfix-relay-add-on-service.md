---
layout: post
title: "Building a Postfix relay in Azure and wiring it into Exchange Online as an email add-on service"
date: 2026-10-29 09:00:00 +0900
categories: [exchange-online, mail-flow, azure, postfix]
---

I needed a realistic way to test message relay (and message recall) through a third-party "add-on" hop, the kind of thing signature or archiving services do. Instead of signing up for a vendor, I built my own: a small Ubuntu VM in Azure running Postfix, with a public TLS certificate, connected to Exchange Online following Microsoft's documented pattern for [integrating Exchange Online with an email add-on service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/integrate-office-365-with-an-email-add-on-service).

This post walks through the build, the Exchange Online side, and the two real-world problems I hit along the way.

> All names here are fictional: the tenant domain is **domain.com**, the relay is **relay.domain.com**, and the relay's public IP is shown as **203.0.113.25** (a documentation-only range).

## The mail flow

1. A user in domain.com sends a message (in my lab, only messages from my admin account with **RelayTest** in the subject).
2. A mail flow rule redirects it to an outbound connector.
3. Exchange Online connects to **relay.domain.com:25**, requires TLS and checks the certificate name.
4. Postfix accepts it (only Exchange Online ranges may relay) and stamps a header: `X-RelayVM-Processed: true`.
5. Postfix sends it back to the tenant's own endpoint over mandatory TLS, presenting its certificate.
6. Exchange Online recognises the relay through a certificate-based inbound connector, sees the processed header so the rule doesn't fire again, and delivers.

## Part 1: The relay VM in Azure

From Azure Cloud Shell (Bash):

```bash
RG=rg-relay-test; LOC=japaneast; VM=relay01; MYIP=$(curl -s ifconfig.me)
az group create -n $RG -l $LOC
az network nsg create -g $RG -n nsg-$LOC -l $LOC
az network nsg rule create -g $RG --nsg-name nsg-$LOC -n SSH  --priority 100 \
  --destination-port-ranges 22 --source-address-prefixes $MYIP/32 --protocol Tcp
az network nsg rule create -g $RG --nsg-name nsg-$LOC -n SMTP --priority 110 \
  --destination-port-ranges 25 587 --protocol Tcp
az network nsg rule create -g $RG --nsg-name nsg-$LOC -n HTTP --priority 120 \
  --destination-port-ranges 80 --protocol Tcp     # Let's Encrypt renewal
az vm create -g $RG -n $VM -l $LOC --image Canonical:ubuntu-24_04-lts:server:latest \
  --size Standard_B2s_v2 --admin-username azureuser --generate-ssh-keys \
  --public-ip-address ${VM}-pip --public-ip-sku Standard \
  --public-ip-address-allocation static --nsg nsg-$LOC
```

Lessons from this step:

- **Capacity and policy vary by subscription.** One subscription refused public IP creation outright, and small B-series sizes had no capacity in some regions. Loop over a few sizes/regions rather than fighting one.
- **Outbound port 25** happened to be open in my subscription type. Many subscription types block it; if yours does, you need a smart host on 587.

Then DNS: an A record `relay.domain.com -> 203.0.113.25`, and reverse DNS on the public IP:

```bash
az network public-ip update -g $RG -n ${VM}-pip --reverse-fqdn relay.domain.com.
```

## Part 2: Postfix and a public certificate

```bash
sudo hostnamectl set-hostname relay.domain.com
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y postfix certbot mailutils
sudo certbot certonly --standalone -d relay.domain.com --agree-tos \
  -m admin@domain.com --non-interactive --deploy-hook "systemctl reload postfix"
```

Core configuration:

```bash
sudo postconf -e \
 "myhostname = relay.domain.com" \
 "mydestination = localhost" \
 "mynetworks = 127.0.0.0/8 40.92.0.0/15 40.107.0.0/16 52.100.0.0/14 104.47.0.0/17" \
 "smtpd_relay_restrictions = permit_mynetworks, reject_unauth_destination" \
 "smtpd_tls_cert_file = /etc/letsencrypt/live/relay.domain.com/fullchain.pem" \
 "smtpd_tls_key_file = /etc/letsencrypt/live/relay.domain.com/privkey.pem" \
 "smtpd_tls_security_level = may" \
 "smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1" \
 "relayhost = [domain-com.mail.protection.outlook.com]:25" \
 "smtp_tls_security_level = encrypt" \
 "smtp_tls_cert_file = /etc/letsencrypt/live/relay.domain.com/fullchain.pem" \
 "smtp_tls_key_file = /etc/letsencrypt/live/relay.domain.com/privkey.pem" \
 "header_checks = regexp:/etc/postfix/header_checks"

echo '/^From:/ PREPEND X-RelayVM-Processed: true' | sudo tee /etc/postfix/header_checks
sudo systemctl reload postfix
```

Key points:

- `mynetworks` limits relaying to Exchange Online's sending ranges, so this is **not an open relay**. Check these against Microsoft's published endpoint list before relying on them.
- `smtp_tls_cert_file` makes Postfix present its certificate when it connects back to Exchange Online. That is what the certificate-based inbound connector matches on.
- The `header_checks` line is the loop breaker.

Quick check from outside:

```bash
openssl s_client -starttls smtp -connect relay.domain.com:25 -servername relay.domain.com -brief
```

## Part 3: Exchange Online, the four documented steps

Connect with `Connect-ExchangeOnline`, then:

**1. Outbound connector (On-premises type, rule-scoped)**

```powershell
New-OutboundConnector -Name "RelayVM-Outbound" -ConnectorType OnPremises `
  -IsTransportRuleScoped $true -UseMxRecord $false -SmartHosts relay.domain.com `
  -TlsSettings DomainValidation -TlsDomain relay.domain.com -CloudServicesMailEnabled $true
```

I first created both connectors as **Partner**. For this scenario they need to be **On-premises** ("your organization's email server"), so returning mail is treated as internal and can be relayed onward.

**2. Mail flow rule with a loop-breaking exception**

```powershell
New-TransportRule -Name "RelayVM - Route RelayTest mail" `
  -From admin@domain.com -SubjectContainsWords "RelayTest" `
  -RouteMessageOutboundConnector "RelayVM-Outbound" `
  -ExceptIfHeaderContainsMessageHeader "X-RelayVM-Processed" `
  -ExceptIfHeaderContainsWords "true" -StopRuleProcessing $true
```

The sender and subject conditions keep the blast radius tiny while testing. Drop them to route everything.

**3. Accepted domain**

Add `relay.domain.com` in the Microsoft 365 admin center (Settings > Domains). As a subdomain of an already verified domain it verified automatically. I skipped adding any service DNS records so the existing A record was untouched.

**4. Inbound connector (certificate-based)**

```powershell
New-InboundConnector -Name "RelayVM-Inbound" -ConnectorType OnPremises -SenderDomains * `
  -RequireTls $true -RestrictDomainsToCertificate $true `
  -TlsSenderCertificateName relay.domain.com -CloudServicesMailEnabled $true
```

Note that an inbound connector's type can't be changed in place. Remove it and recreate it.

## Problem 1: "450 4.4.317 Cannot establish session"

Exchange Online suddenly couldn't reach the relay:

```
450 4.4.317 Cannot establish session with remote server ... Winsock error code: 995
```

Postfix was listening, the NIC firewall allowed 25, yet a packet capture on the VM saw **zero** inbound SYNs. The Azure activity log explained it: an organisation-level **Azure Policy** had silently attached a new network security group to the **subnet** with only the default deny-all rule. Subnet and NIC NSGs are both evaluated, so the subnet one won.

Takeaways:

- When a port "should" be open, check **effective** security rules, not just the NSG you created.
- Look at the activity log for changes made by policy identities.
- Get an approved exception rather than fighting the policy.

## Problem 2: "550 5.7.606 Access denied, banned sending IP"

Once traffic flowed, some returning messages were rejected:

```
550 5.7.606 Access denied, banned sending IP [203.0.113.25]
```

Azure public IPs that send on port 25 often land on Microsoft's blocked-sender list. The fix is the self-service delist portal at <https://sender.office.com/>: enter an email address and the IP, solve a CAPTCHA, then click the link in the confirmation email.

What I observed:

- Delisting took effect on **most** Exchange Online front ends within minutes.
- Some servers (one regional group in particular) kept rejecting for over an hour, so the same IP was accepted for one mailbox and rejected for another seconds apart.
- Postfix treats a 5xx as permanent, so those copies bounce and must be resent.

If it persists, resubmit the delisting, try a fresh public IP, or open a support case quoting the rejecting server name and timestamp from the bounce.

## Testing checklist

- `openssl s_client -starttls smtp` against the relay: TLS 1.3, valid chain.
- Send from the relay to a tenant mailbox: the log shows `Trusted TLS connection established` and `status=sent (250 ...)`.
- Send a RelayTest message to an internal and an external recipient, then run a message trace (`Get-MessageTraceV2`). You should see the outbound connector, the inbound connector, and delivery.
- Open the delivered message's headers and look for `X-RelayVM-Processed: true`.
- Recall the message. The recall request also matches the rule, so it takes the same path.

## Day-2 management

| Task | Command |
|---|---|
| Active settings | `postconf -n` |
| Queue / flush / delete | `postqueue -p`, `postqueue -f`, `postsuper -d ID` |
| Live log | `tail -f /var/log/mail.log` |
| Certificate expiry | `openssl x509 -in /etc/letsencrypt/live/relay.domain.com/cert.pem -noout -enddate` |
| Renewal test | `certbot renew --dry-run` |

You don't need SSH for any of this. `az vm run-command invoke --command-id RunShellScript --scripts "postconf -n"` works even when port 22 is closed. To pause routing in Exchange Online, run `Disable-TransportRule "RelayVM - Route RelayTest mail"`.

## Cleaning up

Remove things in this order: the rule, then both connectors, then the accepted domain, then the DNS record, then the resource group (`az group delete -n rg-relay-test`). Removing the rule first means no mail is ever routed to a connector that no longer exists.
