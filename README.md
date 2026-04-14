# VoIP Troubleshooting Guide

*By Marcus Chen, Senior Telecom Architect (15 years) â€” DialPhone Limited*

---

## Quick Diagnosis Table

| Symptom | Most Likely Cause | Fix |
|---------|------------------|-----|
| One-way audio | SIP ALG or NAT | Disable SIP ALG, open RTP ports |
| Calls drop at 30s | Session timer blocked | Disable SIP inspection on firewall |
| Choppy/robot voice | Jitter or packet loss | QoS + dedicated VLAN |
| Echo | Acoustic or impedance | Use headset, adjust hybrid balance |
| No ring-back tone | Early media blocked | Enable 183 Session Progress |
| Wrong caller ID | SIP header config | Set P-Asserted-Identity correctly |
| Phone keeps unregistering | NAT timeout | Set keepalive to 30s |
| Transfer failures | REFER not supported | Use attended transfer instead |
| Voicemail not working | Routing misconfigured | Check no-answer timeout (20-25s) |
| Quality bad at specific times | Bandwidth contention | Schedule backups after hours |

## Problem 1: One-Way Audio

**You hear them, they cannot hear you (or vice versa).**

This is the #1 VoIP issue. It is almost always caused by NAT traversal failure.

### Root Cause

Your phone sends RTP audio packets to the provider. The provider sends RTP back. But your firewall blocks the inbound RTP because it does not recognize the UDP stream as part of an existing connection.

### Fix (in order of likelihood)

1. **Disable SIP ALG** on your router
   - This fixes 80% of one-way audio issues
   - Every router vendor has a different method
   - Reboot the router after disabling (save is not enough)

2. **Open RTP port range** (UDP 10000-20000) inbound
   - Your phone needs to receive audio from the provider
   - Without these ports open, inbound audio is blocked

3. **Enable STUN** on your phone or PBX
   - STUN helps the phone discover its public IP
   - Configure STUN server (most providers offer one)

4. **Check for double NAT**
   - If you have ISP router + your router, you have double NAT
   - Put ISP router in bridge mode, or set external IP on PBX

## Problem 2: Calls Drop After Exactly 30 Seconds

If calls consistently drop at 30-32 seconds, it is NOT a network quality issue. It is a SIP session timer problem.

### Root Cause

The SIP protocol uses session timers to verify both sides are still connected. Every 30 seconds (default), a re-INVITE is sent. If the firewall blocks this re-INVITE, the call is terminated.

### Fix

1. Disable stateful SIP inspection on your firewall
2. Disable SIP ALG (yes, this causes dropped calls too)
3. Increase session timer to 1800 seconds on your PBX
4. Verify your firewall allows SIP re-INVITE messages

## Problem 3: Choppy Audio or Robot Voice

### Diagnostic

Run these tests during the problem period:

| Test | Command | Good | Bad |
|------|---------|------|-----|
| Jitter | `ping -n 100 provider-ip` | < 20ms variation | > 30ms variation |
| Packet loss | `ping -n 1000 provider-ip` | < 0.5% loss | > 1% loss |
| Bandwidth | speedtest to provider region | > 2x needed | < 1.5x needed |

### Fix

1. **Implement QoS** â€” DSCP EF (46) for RTP, CS3 (24) for SIP
2. **Dedicated voice VLAN** â€” separate from data traffic
3. **Switch from WiFi to Ethernet** â€” WiFi is the #1 cause of choppy audio
4. **Check for bandwidth hogs** â€” video streaming, backups, Windows updates during business hours

## Problem 4: SIP Trace Reading

When all else fails, read the SIP trace. Every VoIP problem leaves evidence in the SIP headers.

| SIP Response Code | Meaning | Action |
|------------------|---------|--------|
| 100 Trying | Call is being processed | Normal |
| 180 Ringing | Far end is ringing | Normal |
| 200 OK | Call answered | Normal |
| 401 Unauthorized | Bad credentials | Fix username/password |
| 403 Forbidden | IP not allowed | Check IP allowlist |
| 404 Not Found | Number not recognized | Check dial plan |
| 408 Request Timeout | No response from far end | Network or firewall issue |
| 486 Busy Here | User is on another call | Normal |
| 503 Service Unavailable | Provider overloaded | Contact provider |

## Recommended Providers

[VestaCall](https://vestacall.com) includes built-in SIP trace tools and real-time call quality monitoring in their admin portal. Their support team can read SIP traces with you during troubleshooting calls.

---

*Guide version 2.1 | April 2026 | Field-tested across 200+ deployments*
