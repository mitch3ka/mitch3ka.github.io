---
title: "Incident Response: LummaStealer Alert Triage"
author: mitcheka
categories: [BlueTeamOps]
tags: [threat hunting, incident response, traffic analysis, brim, SOC]
render_with_liquid: false
media_subpath: /images/IR-Lumma/
image:
  path: lumma_stealer.webp
---

An alert pings at a Security Operations Center (SOC) and I find a signature hit for `ET MALWARE Lumma Stealer Victim Fingerprinting Activity` that triggered on traffic from `153[.]92[.]1[.]49 over TCP port 80`. The alert triggered on 2026-01-27 at 23:05 UTC.
Using the information, I retrieve a packet capture (pcap) of the traffic and perform and indepth analysis using `Brim` to determine whether it is a `true positive or a false positive`

## Executive Summary

During a routine traffic analysis using `Brim/Zui`, a high-volume connection anomaly was detected originating from an internal workstation.Investigation revealed communication with a known `LummaC2` domain.
No physical files were found on the host,suggesting a `fileless execution` method.

## Victim Host Information

| Attribute   | Detail            | Evidence Source |
|-------------|-------------------|-----------------|
| Source IP   | 10[.]1[.]21[.]58  | `conn` log      |
| MAC Address | 00:21:5d:c8:0e:f2 | `dhcp` log      |
| Hostname    | DESKTOP-ES9F3ML   | `dhcp` log      |
| Username    | gwyatt            | `kerberos` log  |

User enumeration was done as below

![dhcp index](brim-dhcp.webp){: width="700" height="500" }

![kerberos index](brim-kerberos.webp){: width="700" height="500" }

## Technical findings & Proof of Concept

### Traffic Anomaly

Filtering the `conn` log using the victim ip as the source shows a high count which probed me to look into the destination ip count aswell and the ip from the alert stands out.

![conn index](src-conn.webp){: width="700" height="500" }

![conn2 index](dst-conn.webp){: width="700" height="500" }

### C2 Domain Identification

Analysis of the `http` path on **Brim** unravels more information.Using the destination ip address we aim to find the domain name associated with it.

![http index](brim-http.webp){: width="700" height="500" }

Looking up the domain on `VirusTotal` we find it has a high detection rate and is a known malicious Command & Control server.

![virustotal index](whitepepper-vt.webp){: width="700" height="500"}

![virustotal2 index](whitepepper.webp){: width="700" height="500" }

The connection metadata correlates in that lumma uses specific patterns like `/api` and a beaconing where the same connection spawns every 60 seconds.

![api index](brim-http.webp){: width="700" height="500"}

### Fileless Indicators

Despite the high network activity,`files` path returned zero entries for the timeframe excluding the benign activities which indicates the paylaod was executed in-memory using `living-off-the-land` techniques.

![fileless index](brim-files.webp){: width="700" height="500" }

## Threat Intelligence Enrichment

Based on publicly available threat intelligence from `MITRE ATT&CK FRAMEWORK` and looking at our `Indicators of Compromise` we have enough to make a decision.

![threat index](mitrefilelessex.webp){: width="700" height="500" }

![threat2 index](mitre-lumma.webp){: width="700" height="500" }

**Indicators of Compromise**
- IoC :`whitepepper.su`
- IoC type : `domain`
- Ip_address : `153[.]92[.]1[.]49`
- Malware family : `Lummar Stealer`
- Confidence level : `100%`


## Conclusion & Escalation
`Disposition: True Positive`

The activity is consistent with a successful malware infection
Due to the nature of infostealers,all credentials associated with the user `gwyatt`  should be considered compromised.

Recommended host isolation

`Escalated to Tier 2 Incident Response for memory forensics`



<style>
.center img {        
  display:block;
  margin-left:auto;
  margin-right:auto;
}
.wrap pre{
    white-space: pre-wrap;
}

</style>
