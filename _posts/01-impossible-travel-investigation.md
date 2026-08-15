---
title: "SOC Investigation 01: Impossible Travel"
category: SOC Investigation Portfolio
tags: [Microsoft Sentinel, KQL, Entra ID, Conditional Access, Identity]
---

# SOC Investigation 01: Impossible Travel

## Alert

Sign-in activity for a single tenant account (`adesco@securityinspired0.onmicrosoft.com`) was reviewed in Microsoft Sentinel against the `SigninLogs` table. The activity showed successful authentications from two geographically distant locations within a time window too short for legitimate travel: a sign-in from Great Britain followed by a sign-in from the Netherlands 5 minutes and 20 seconds later, and on a separate occasion, a sign-in from Great Britain followed by one from Canada within under 3 minutes. Both locations reverted back to Great Britain shortly after, producing the same impossible gap in reverse.

This pattern was surfaced by querying successful sign-ins over a rolling window and reviewing timestamp, location, and IP address for the account, rather than through a pre-built Sentinel analytics rule.

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType == 0
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, Location, DeviceDetail
| order by TimeGenerated desc
| take 50
```

![Sign-in log results showing alternating GB and NL sign-ins minutes apart](/assets/images/impossible-travel/01-alert-signinlogs-raw.png)
*Raw `SigninLogs` output showing the account alternating between Great Britain and Netherlands IP addresses within minutes.*

## Validation

To confirm the pattern was genuine and not a logging artefact, sign-in events were reordered chronologically and the time delta between consecutive location changes for the account was calculated directly in KQL.

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where UserPrincipalName == "adesco@securityinspired0.onmicrosoft.com"
| where ResultType == 0
| project TimeGenerated, Location, IPAddress, AppDisplayName, AuthenticationRequirement, ConditionalAccessStatus
| order by TimeGenerated asc
| serialize
| extend PrevLocation = prev(Location), PrevTime = prev(TimeGenerated)
| extend MinutesSinceLastSignin = datetime_diff('minute', TimeGenerated, PrevTime)
| where Location != PrevLocation
| project TimeGenerated, PrevLocation, Location, MinutesSinceLastSignin, IPAddress, AuthenticationRequirement, ConditionalAccessStatus
```

![Time-delta query results showing 5 and 8 minute gaps between GB and NL sign-ins, with ConditionalAccessStatus notApplied](/assets/images/impossible-travel/02-validation-time-delta-query.png)
*Time-delta calculation confirming the location change occurred within 5 and 8 minutes, with `ConditionalAccessStatus` showing `notApplied` on every row.*

This confirmed two location changes of 5 and 8 minutes respectively, both well below any realistic travel time between the countries involved. Critically, the `ConditionalAccessStatus` field returned `notApplied` on every row, indicating no Conditional Access policy was evaluating these sign-ins at all.

## Scope

Before drilling further into the account itself, the investigation checked whether this pattern was isolated or tenant-wide.

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType == 0
| project TimeGenerated, UserPrincipalName, Location, IPAddress
| order by UserPrincipalName, TimeGenerated asc
| serialize
| extend PrevLocation = prev(Location), PrevUser = prev(UserPrincipalName), PrevTime = prev(TimeGenerated)
| where UserPrincipalName == PrevUser
| extend MinutesSinceLastSignin = datetime_diff('minute', TimeGenerated, PrevTime)
| where Location != PrevLocation and MinutesSinceLastSignin < 60
| project UserPrincipalName, TimeGenerated, PrevLocation, Location, MinutesSinceLastSignin, IPAddress
```

A supporting query confirmed only one account had any authentication activity in the tenant during the review window:

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| distinct UserPrincipalName
```

![Query result showing only one distinct UserPrincipalName in the tenant](/assets/images/impossible-travel/03b-scope-distinct-users.png)
*Only one account shows any authentication activity in the tenant during the review window.*

Scope was therefore confirmed as contained to a single account, with no lateral spread to other identities.

## Investigation

The root cause investigation focused on why sign-ins from two different countries, minutes apart, were both succeeding without challenge. A follow-up query pulled `AuthenticationRequirement` and `AuthenticationDetails` alongside `ConditionalAccessStatus` to see the full picture:

![Query results showing AuthenticationRequirement alternating between single and multi-factor, with ConditionalAccessStatus notApplied throughout](/assets/images/impossible-travel/03-investigation-authreq-conditionalaccess-columns.png)
*`AuthenticationRequirement` alternates between single and multi-factor across the same session, while `ConditionalAccessStatus` remains `notApplied` throughout.*

Expanding the `AuthenticationDetails` field on the affected sign-ins (via the sign-in log's Authentication Details tab in the Entra portal) showed the following for every sign-in in the impossible-travel window, including the ones from foreign locations:

```
"authenticationMethod": "Previously satisfied"
"authenticationStepResultDetail": "MFA requirement satisfied by claim in the token"
```

![Expanded AuthenticationDetails JSON showing MFA requirement satisfied by claim in the token across GB and NL sign-ins](/assets/images/impossible-travel/04-investigation-authdetails-json-previously-satisfied.png)
*Expanded `AuthenticationDetails` JSON confirming MFA was satisfied once and then reused as a token claim across both the GB and NL sign-ins.*

This showed that MFA had been satisfied once, early in the session, and that satisfaction had been embedded as a claim inside the issued token. Every subsequent sign-in, regardless of originating location, was inheriting that claim rather than triggering a fresh MFA challenge. In other words, the account's MFA control was being satisfied by session token reuse rather than being re-validated per sign-in, meaning a stolen or replayed token would carry the same trust from anywhere in the world.

Cross-referencing this against the tenant's Conditional Access configuration confirmed why nothing intervened: the Conditional Access overview showed 0 enabled policies, 1 policy in Report-only mode, and 0 policies with a Named Location condition. There was no control in place capable of detecting an atypical location or forcing re-authentication when the session context changed.

![Conditional Access overview showing 0 enabled, 1 Report-only, 0 Named Location conditions](/assets/images/impossible-travel/05-investigation-ca-overview-zero-enforced.png)
*Conditional Access overview at the time of the incident: 0 enabled policies, 1 in Report-only, 0 policies with a Named Location condition.*

## Containment

The immediate containment action was to revoke the account's active sessions and refresh tokens via the Entra admin center (Users > [account] > Revoke sessions), invalidating any token issued before that point, including the one carrying the reused MFA claim.

![Confirmation dialog for revoking all sessions for the user in Entra ID](/assets/images/impossible-travel/06b-containment-revoke-sessions-confirmation.png)
*Revoking all active sessions for the account via the Entra admin center.*

This was confirmed via the native Entra ID audit log, which recorded the following operation:

```
Activity: Update StsRefreshTokenValidFrom
Status: Success
Target: [account]
```

![Entra ID audit log showing Update StsRefreshTokenValidFrom operation succeeding](/assets/images/impossible-travel/06-containment-audit-log-token-revoke.png)
*Native Entra ID audit log confirming the `Update StsRefreshTokenValidFrom` operation succeeded.*

This operation resets the refresh token validity timestamp for the account, meaning any session token issued before it becomes invalid and the account must fully re-authenticate.

## Recovery

To prevent recurrence, a new Conditional Access policy (`CA01-ImpossibleTravel-SessionControls-Response`) was designed and deployed with the following configuration:

- **Users:** targeted account
- **Cloud apps:** all resources
- **Locations:** Include – Any location; Exclude – a Named Location representing the account's trusted IP
- **Grant control:** Require multifactor authentication
- **Session controls:** Sign-in frequency (forces periodic re-authentication rather than indefinite token reuse), persistent browser session set to never persistent

The policy was deployed in Report-only mode first, as standard practice, to validate its behaviour against real sign-in traffic before enforcement.

Initial validation testing found the policy was not evaluating correctly: sign-ins from both the trusted GB location and a foreign test location returned `Network: Not matched`, meaning the policy was not applying to any sign-in regardless of origin.

![Conditional Access policy details showing Network: Not matched for a GB sign-in](/assets/images/impossible-travel/07-recovery-bug-discovery-notmatched.png)
*Even a sign-in from the account's own trusted GB location returned `Network: Not matched`, the first sign that the location condition was broken.*

![Conditional Access policy details showing Network: Not matched for a foreign CA sign-in](/assets/images/impossible-travel/09-recovery-diagnostic-network-not-matched.png)
*A foreign test sign-in from Canada also returned `Not matched`, confirming the policy wasn't evaluating location at all, regardless of origin.*

Troubleshooting identified two underlying configuration defects. First, the Named Location intended to represent the trusted GB location had not been flagged as a trusted location in Entra:

![Named Locations list showing the Trusted column blank and no linked Conditional Access policy](/assets/images/impossible-travel/10-recovery-namedlocation-untrusted-unlinked.png)
*The Named Location's `Trusted` column is blank, and it shows as "Not configured in any policy yet."*

Second, the policy's own location condition was misconfigured, with **Include** set to "All trusted locations" rather than "Any location." This meant the policy would only ever evaluate sign-ins already coming from a trusted source, the inverse of what an impossible-travel control needs to do:

![Policy details showing Locations set to All trusted locations under Include](/assets/images/impossible-travel/11-recovery-root-cause-inverted-include-logic.png)
*Policy summary confirming the Include condition was set to "All trusted locations" instead of "Any location," inverting the intended logic.*

Both issues were corrected: the Named Location was marked as trusted, and the policy's location condition was changed to Include: Any location, Exclude: the trusted Named Location.

![Corrected policy configuration with Network set to any location and one excluded](/assets/images/impossible-travel/12-recovery-fix-applied-exclude-trusted-location.png)
*Corrected configuration: Include set to any location, with the trusted Named Location excluded.*

Re-testing with a fresh foreign sign-in confirmed the fix was starting to take effect, with the policy moving from "Not applied" to an actual evaluated result:

![Report-only tab showing the CA policy result as Report-only: Success](/assets/images/impossible-travel/13-recovery-reportonly-success-transition.png)
*The policy now evaluates the sign-in rather than skipping it entirely, returning `Report-only: Success`.*

A final validation sign-in from the Netherlands confirmed the fix was fully correct:

- **Network:** Matched (correctly identified as outside the trusted location)
- **Grant Controls:** Satisfied
- **Session Controls:** Enforced

![Policy details showing Network: Matched, Grant Controls: Satisfied, Session Controls: Enforced for an NL sign-in](/assets/images/impossible-travel/14-recovery-final-validation-matched-enforced.png)
*Final validation: the corrected policy correctly matches the foreign sign-in and would enforce both grant and session controls if switched to On.*

With the corrected configuration validated in Report-only mode, the policy was switched to On:

![Policy Enable toggle set to On](/assets/images/impossible-travel/15-recovery-policy-enabled-on.png)
*The policy's Enable setting switched from Report-only to On.*

The state change was corroborated in the native Entra ID audit log, which recorded the exact field-level transition:

```
Old Value: "state":"enabledForReportingButNotEnforced"
New Value: "state":"enabled"
```

![Audit log Modified Properties showing the policy state changing from enabledForReportingButNotEnforced to enabled](/assets/images/impossible-travel/15c-recovery-audit-modified-properties-state-change.png)
*Audit log evidence of the exact state transition, from Report-only to enforced, along with the full corrected policy configuration.*

A follow-up sign-in from the trusted GB location after the policy went live confirmed the change did not lock the account out:

![Sign-in log entry showing a successful GB sign-in timestamped after the policy was enabled](/assets/images/impossible-travel/16-recovery-postenforcement-signin-success.png)
*A successful sign-in from the trusted location, timestamped after enforcement, confirming no lockout occurred.*

## Conclusion

What began as a simple impossible-travel pattern in the sign-in logs traced back to a token-based MFA bypass: MFA satisfaction was being carried as a portable claim inside the session token rather than being re-validated against session context, meaning the control did not actually stop a token from being used across geographically impossible locations. This is the same mechanism exploited by real-world token theft and adversary-in-the-middle phishing attacks, where an attacker who steals a session token inherits its MFA-satisfied state without needing to solve MFA themselves.

The tenant had no enforced Conditional Access policies at the time of the incident, and the one existing policy was in Report-only mode, meaning no control was in a position to catch this even if it had been correctly configured. Remediation involved not only deploying a new policy with session controls (sign-in frequency) specifically chosen to force re-validation of tokens on a schedule, but also diagnosing and correcting two independent misconfigurations that were silently preventing that policy from ever applying, regardless of the sign-in's actual location.

This investigation demonstrates the difference between confirming a symptom (impossible travel in the logs) and identifying the underlying control gap that allowed it (unenforced, misconfigured Conditional Access), along with the ability to validate a fix with evidence rather than assuming a deployed policy is working as intended.
