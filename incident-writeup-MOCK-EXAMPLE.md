> **MOCK EXAMPLE - ALL DATA IS FICTIONAL.**
> This shows the format and level of detail of a good write-up. Replace every name, IP, domain, hash, time and finding with what you actually saw in your own lab room before publishing. Delete this banner when done.

# Incident Investigation: Phishing Email with Credential-Harvesting Link

**Analyst:** Hamza Naeem
**Environment:** TryHackMe lab - [your room name] (simulated, no real data)
**Tools used:** Email header analysis, VirusTotal, Splunk, MITRE ATT&CK
**Date:** [DD Month YYYY]

## 1. Summary

A user reported a suspicious email claiming their account would be suspended unless they verified their password. Header and URL analysis showed the sender was spoofed and the link led to a fake login page. Splunk logs showed one user clicked the link but no credentials were submitted to internal systems. Verdict: **True Positive, Medium severity**.

## 2. Alert Details

| Field | Value |
| --- | --- |
| Alert name / source | User-reported phishing email |
| Time (UTC) | 2026-03-14 09:12 |
| Affected user | user.finance01 |
| Affected host | WKS-FIN-04 |
| Initial severity | Medium |

## 3. Investigation Steps

1. **Checked the email headers.** The display name said "IT Support" but the real sender address was `support@it-helpdesk-verify.example.com`. The Return-Path and Reply-To domains did not match the company domain, and SPF showed `fail`.
2. **Extracted the link** from the email body: `http://login-verify.example.net/update`. I did not open it directly.
3. **Checked the domain and IP in VirusTotal.** The domain was flagged by 9 of 90 security vendors as phishing, and it resolved to `203.0.113.45`.
4. **Searched Splunk for who clicked.** I searched proxy logs for the malicious domain:

```
index=proxy dest_domain="login-verify.example.net" | stats count by user, src_ip, _time
```

5. **Result:** one user (`user.finance01`, host WKS-FIN-04) visited the URL once at 09:20. No other users contacted the domain.
6. **Checked for credential submission.** Proxy logs showed a single POST request to the same domain. No later suspicious logins for this user appeared in the authentication logs.

## 4. Timeline

| Time (UTC) | Event | Source |
| --- | --- | --- |
| 09:05 | Phishing email delivered to user.finance01 | Mail gateway log |
| 09:12 | User reported the email as suspicious | Alert ticket |
| 09:20 | User visited login-verify.example.net | Proxy log (Splunk) |
| 09:20 | One POST request sent to the same domain | Proxy log (Splunk) |
| 09:35 | Analyst confirmed domain as malicious | VirusTotal |

## 5. Evidence

[Insert screenshots: 1) email headers showing SPF fail, 2) VirusTotal result page, 3) Splunk search result table. Add a one-line caption under each.]

## 6. Indicators of Compromise (IOCs)

| Type | Value | Verdict |
| --- | --- | --- |
| Sender address | support@it-helpdesk-verify.example.com | Malicious |
| URL | http://login-verify.example.net/update | Malicious |
| IP | 203.0.113.45 | Malicious |
| Attachment hash | None | N/A |

## 7. MITRE ATT&CK Mapping

| Tactic | Technique | ID |
| --- | --- | --- |
| Initial Access | Phishing: Spearphishing Link | T1566.002 |
| Credential Access | Input Capture (credential harvesting page) | T1056 |

## 8. Verdict

**True Positive** | **Final severity: Medium**

The email was a spoofed phishing message, and one user clicked the link and submitted a form to the attacker's site. Severity is Medium and not High because only one user was affected and no suspicious logins followed, but the user's password must be treated as compromised.

## 9. Recommended Actions

- **Contain:** Block the domain and IP at the proxy and firewall. Remove the email from all mailboxes.
- **Eradicate:** Force a password reset for `user.finance01` and revoke active sessions.
- **Prevent:** Add the sender domain to the email blocklist, enable multi-factor authentication for all accounts, and send a short phishing-awareness reminder to the finance team.

## 10. What I Learned

Checking the email headers first saved time, because the SPF fail confirmed spoofing before I looked at anything else. I would build a saved Splunk search for "users who contacted a flagged domain" so scoping an incident takes minutes instead of manual searching.
