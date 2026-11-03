# Responsible Disclosure of VitaFit Android and Cloud Security Vulnerabilities

Researcher: Jose Lomeli<br>
Contact: jelvmx@proton.me<br>
Developer: Shenzhen Shine Industrial Co., Ltd. (`深圳市森禾实业有限公司`)<br>
Product: VitaFit Android application and supporting cloud services<br>
Android package: `com.vt.vitafit`<br>
Published: November 3, 2026

## What is VitaFit?

VitaFit is a smart-scale platform with mobile applications for Android and iOS. The applications connect to compatible scales over Bluetooth, let users manage personal profiles and measurement history, and track measurements such as weight, BMI, body fat, body water, muscle mass, and bone mass. Account, profile, measurement, and user-uploaded image data are synchronized with VitaFit's cloud services.

[VitaFit offers a range of consumer scales](https://vitafitscale.com/#choose-your-vitafit-scale), which are widely sold online through retail channels including Amazon. This research assessed VitaFit 2.8.0.4 for Android and the platform's supporting production cloud services.

## Summary

The security assessment covered version 2.8.0.4 of the VitaFit Android application and its production cloud services tested in mid-May 2026. As a result of the assessment, thirteen (13) vulnerabilities were found, including two (2) Critical and six (6) High-severity findings. The findings affected Android transport security, session handling, OAuth integrations, API authorization, AWS credential issuance, object storage, account privacy, and social login.

The two Critical findings allowed any authenticated VitaFit user to obtain AWS credentials with cross-user storage access, and allowed an unauthenticated attacker who knew a linked social-login identifier to take over the targeted VitaFit account. A demonstrated chain combined account discovery, forged social login, and AWS access into a 9.9 Critical attack path.

VulnCheck allocated CVE identifiers to twelve (12) findings. One additional valid finding, GDPR incomplete data erasure (Fixed), did not receive a separate CVE because it was exploitable only through the IDOR and public S3 vulnerabilities described under CVE-2026-54377 and CVE-2026-54381.

VitaFit was very responsive throughout the coordinated disclosure and treated the findings with the seriousness they deserved from the first disclosure email onward. I would like to thank its engineering and support teams for providing clear remediation plans, deploying fixes, and supporting repeated verification through closure.

I would also like to thank Wade Sparks III and VulnCheck for their support throughout the disclosure process and for coordinating the CVE allocations.

## Versions

- Affected Android application: VitaFit 2.8.0.4.
- Retested Android application: VitaFit 2.8.1.0, version code `2810`.
- The initial assessment was performed in May 2026, and remediation retesting ran from October to November 3, 2026.

## Findings

### CVE-2026-54372: Trust-All X509TrustManager Disables Certificate Validation

- Component: Android application
- CWE: CWE-295
- **8.8 High**: `CVSS:3.1/AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- Status: Fixed

The Android application used an `OkHttpClient` with a custom `X509TrustManager` that trusted every certificate. Its hostname verifier also accepted every hostname. The same connection code handled login, health-data synchronization, and storage requests.

An interception proxy could read HTTPS application traffic without a trusted CA installed on the device. An attacker positioned on the network path could intercept and modify account sessions and health-related API data. A captured VitaFit session could also delete the account's authentication record and prevent later login.

VitaFit 2.8.1.0 rejects untrusted certificates while continuing to operate normally with a trusted certificate.

### CVE-2026-54373: Cleartext Traffic Permitted and No Network Security Config

- Component: Android application
- CWE: CWE-319
- **3.5 Low**: `CVSS:3.1/AV:A/AC:L/PR:N/UI:R/S:U/C:N/I:L/A:N`
- Status: Fixed

The Android application permitted unencrypted HTTP traffic. During testing, no credentials or health data were observed in cleartext. The demonstrated impact was limited to modifying support content shown to a user on an unsafe network.

VitaFit 2.8.1.0 blocks cleartext traffic.

### CVE-2026-54374: Hardcoded Third-Party API Secrets (Facebook, WeChat, QQ, Fitbit)

- Component: Android application
- CWE: CWE-798
- **7.5 High**: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`
- Status: Fixed

The APK included third-party integration values for Facebook, WeChat, Tencent QQ, and Fitbit. The confidential WeChat, Tencent QQ, and Fitbit credentials could be extracted by decompiling the application. The associated third-party application registrations were active, but the credentials were not used to change data.

VitaFit 2.8.1.0 no longer contains the reported confidential credentials.

### CVE-2026-54375: Session Token with 10-Year Expiry Stored in Cleartext SharedPreferences

- Component: Android application and session service
- CWE: CWE-613
- **8.4 High**: `CVSS:3.1/AV:L/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- Status: Partially Fixed

VitaFit issued `m_session_id` login sessions that remained valid for ten years. The Android application stored them without encryption and allowed them to be included in application backups. An attacker who extracted or intercepted a session could read and modify the account for the token's ten-year validity period. The same session could delete the account's authentication record and prevent the owner from logging in again.

The current application encrypts stored sessions, and logout and password reset now invalidate them. During the retest, a couple of old sessions preserved from the original testing period were still accepted by the server. One of those sessions was exchanged for a new 90-day session, and the server rejected the old session after a five-minute grace period.

The main issue is confirmed fixed: newly issued sessions expire after 90 days, the application encrypts stored sessions, and logout and password reset invalidate active sessions.

This item remains Partially Fixed because session tokens issued before the remediation are still temporarily accepted. When an old token is used, the server replaces it with a new 90-day token and disables the old token after five minutes. VitaFit plans to reject any old tokens that have not been exchanged after January 28, 2027.

### CVE-2026-54376: Fitbit OAuth Token Theft via Custom Scheme Hijack

- Component: Android application
- CWE: CWE-926
- **8.2 High**: `CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N`
- Status: Fixed

The Android application used the custom URI scheme callback `vitafit://vtio.cn` to receive Fitbit login tokens. A proof-of-concept Android application registered the same scheme, appeared as an eligible callback handler, and received a test token intended for VitaFit.

A successful Fitbit authorization returned a 30-day token with `weight` access directly in the URL. [Fitbit's Body API documentation](https://dev.fitbit.com/build/reference/web-api/body/) shows that this access can read, create, and delete weight and body-fat records. In order to exploit this issue, an attacker requires a malicious application installed on the phone and a user who completes the Fitbit authorization flow through that application.

VitaFit 2.8.1.0 removed this callback and no longer uses the affected token flow.

### CVE-2026-54377: BOLA/IDOR: Any Authenticated User Can Read All Users' Health Data and PII

- Component: Cloud API
- CWE: CWE-639
- **6.5 Medium**: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N`
- Status: Fixed

Nine API endpoints accepted a `user_id` or `profile_id` without verifying that the requested record belonged to the authenticated user. Cross-account testing confirmed access to another test account's email address, profile, body-composition report, current measurements, and measurement history. Write attempts were rejected, confirming that the demonstrated weakness affected read operations.

User identifiers were available through the account-enumeration issue described below, and the original account structure used a predictable relationship between `user_id` and `profile_id`. This made the cross-account read practical at scale.

The tested endpoints now deny cross-account access or have been removed.

Account deletion through `POST /front/gdpr-clear-data/` also failed to erase data that remained reachable through this weakness and the public-storage exposure described under CVE-2026-54381. Deleting an account removed its login, but its profile and health data remained readable through the cross-account endpoints, and the application provided no way to delete the related stored files.

VulnCheck did not allocate a separate CVE because the residual access depended on the two allocated vulnerabilities.

After remediation, the deleted test profile was no longer available through either `GET /front/profile/{profile_id}/report/` or `GET /front/profile/{profile_id}/body_index_per_day/`, its old session was rejected, and the cross-account read was closed.

### CVE-2026-54378: AWS Cognito Credentials Expose S3 Read/Write/List to Any Authenticated User

- Component: Cloud API and AWS infrastructure
- CWE: CWE-522
- **9.6 Critical**: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:N`
- Status: Fixed

Any logged-in user could call `POST /front/user/s3/token/` and exchange its response for temporary AWS credentials. Those credentials could list the shared storage bucket and read or overwrite objects belonging to other users. A user could request new credentials whenever needed.

VitaFit has removed the affected credential endpoint in every tested region.

### CVE-2026-54380: Stored XSS via SVG Upload to Public S3 Bucket

- Component: AWS infrastructure and browser delivery
- CWE: CWE-79
- **5.4 Medium**: `CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N`
- Status: Fixed

Using the AWS credentials obtained through CVE-2026-54378, an authenticated user could upload an SVG image containing JavaScript. VitaFit's storage served the file as `image/svg+xml` without forcing a download or restricting scripts. When the uploaded file was opened in a browser, the browser executed the JavaScript.

Testing confirmed that the JavaScript executed only after direct browser navigation. The Android application used a native image renderer, so the script did not execute inside the application's security context. No method to achieve session theft or self-propagation was identified during the assessment.

VitaFit removed the affected AWS credential endpoint, and the current upload process rejects SVG files while still accepting ordinary PNG images.

### CVE-2026-54381: Public S3 Bucket Exposes 129,947 Objects Without Authentication

- Component: AWS infrastructure
- CWE: CWE-732
- **7.5 High**: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`
- Status: Fixed

Anyone could list VitaFit's storage bucket and download its files without authentication. The bucket listed 129,947 objects dating from November 2020 through May 2026, including 40,169 user-uploaded profile images from Android and 86,712 user-uploaded profile images from iOS. Exposure was confirmed by downloading one listed image without authentication.

The bucket now returns `AccessDenied` for anonymous listings and direct downloads, including requests for known test objects.

### CVE-2026-54382: Overpermissioned Cognito IAM Role Allows Overwriting Any User's Files

- Component: AWS infrastructure
- CWE: CWE-266
- **7.7 High**: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:H/A:N`
- Status: Fixed

The AWS role allowed `s3:PutObject` across the entire shared bucket instead of limiting each user to their own folder. In a test with two accounts, the first account created an object and the second replaced it. Reading the object back confirmed the replacement. Direct deletion was blocked, but overwriting still destroyed the original content.

VitaFit removed the endpoint that supplied the AWS credentials, and current uploads use separate storage paths for each account.

### CVE-2026-54383: Mass User Enumeration: Search Endpoint Exposes User PII at Scale

- Component: Cloud API
- CWE: CWE-862
- **6.5 Medium**: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N`
- Status: Fixed

The `GET /front/relation/search/` endpoint allowed any authenticated user to search the VitaFit user directory and retrieve PII and health-related profile information. A single search for `nickname=a` reported 313,135 user records, while an empty search reported 497,770 user records. Across 36 overlapping searches for `a-z` and `0-9`, the server reported 2,842,529 user-record matches. The same user record could appear in more than one search.

Returned fields included user and account identifiers, email addresses, birthdays, gender, height, weight goals, phone numbers where registered, and usernames containing Firebase UIDs for Google-linked accounts. The exposed identifiers also enabled access to complete health measurements through CVE-2026-54377 and supported the demonstrated account-takeover chain.

The resulting account-takeover and AWS-access chain is described under Finding chains.

VitaFit has removed the search endpoint from every tested API region.

### CVE-2026-54384: Account Takeover via Forged OAuth Login: No Server-Side Token Validation

- Component: Cloud API
- CWE: CWE-287
- **9.8 Critical**: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- Status: Fixed

The social-login endpoint `POST /front/login/sns/` trusted a Google or Facebook account identifier without requiring proof from the provider. Sending the same identifier from another device returned a new VitaFit session for the linked account.

The issue was reproduced against a researcher-controlled Google-linked test account using an identifier exposed by the account-enumeration weakness. The forged login returned a full session that could read and modify the profile. It could also delete the account's authentication record and prevent later login. The original proof used Google, although the Android application used the same design for Facebook.

VitaFit now accepts genuine Google and Facebook tokens and rejects invalid ones without creating a session. The fix was verified for Google and Facebook.

## Finding chains

The findings combined into several distinct attack paths:

1. **Cross-account health-data access.** Any registered VitaFit user could access CVE-2026-54383, which exposed user identifiers and the predictable profile relationship. CVE-2026-54377 accepted those identifiers when returning another account's profile and health measurements.
2. **Account takeover and AWS access.** CVE-2026-54383 exposed the social-login identifiers required by CVE-2026-54384. The forged login returned a full target-account session, which could reach the AWS credential route in CVE-2026-54378 and the cross-user overwrite capability in CVE-2026-54382. This fully demonstrated attack chain has a separate CVSS v3.1 score of **9.9 Critical** (`CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`). This chained score does not replace the 6.5 component score for CVE-2026-54383.
3. **Captured session to AWS access.** Sessions obtained through CVE-2026-54372 or CVE-2026-54375 could reach the AWS credential route in CVE-2026-54378 and the cross-user overwrite in CVE-2026-54382.
4. **Stored browser script execution.** The AWS credentials from CVE-2026-54378 allowed SVG upload, and CVE-2026-54380 caused the uploaded JavaScript to run when the file was opened in a browser. The same storage permissions also allowed cross-user replacement through CVE-2026-54382.
5. **Incomplete erasure through surviving access paths.** Account deletion removed the login record, but profile and health data remained reachable through CVE-2026-54377 and stored files remained exposed through CVE-2026-54381. Because the residual exposure depended on those two vulnerabilities, no separate CVE was allocated for incomplete erasure.

The individual transitions were verified with researcher-controlled accounts and test objects. Testing did not perform bulk extraction, mass access to unrelated accounts, session theft through the SVG, or self-propagation.

## Remediation status

Eleven of the twelve CVE-listed findings are Fixed. The long-lived session issue is Partially Fixed only while VitaFit completes the transition for old sessions issued before the remediation. VitaFit plans to reject every remaining old session after January 28, 2027. The additional GDPR incomplete data erasure finding is Fixed.

## Coordinated disclosure timeline

- **May 19, 2026:** VitaFit was notified, and a 90-day coordinated-disclosure period was proposed.
- **June 3, 2026:** VitaFit acknowledged the disclosure and requested the complete technical assessment.
- **June 6, 2026:** The complete assessment was delivered to VitaFit.
- **June 17, 2026:** VulnCheck allocated CVE identifiers to the submitted findings.
- **July 21, 2026:** VitaFit supplied a per-finding remediation plan and requested additional time.
- **October 18, 2026:** The complete remediation-retest report was delivered to VitaFit with three items remaining open.
- **October 30, 2026:** VitaFit reported that the remaining server-side changes had been deployed.
- **November 3, 2026:** Final verification of the remaining deployed changes was completed.

## Credit

Research and coordinated disclosure: Jose Lomeli.

CVE allocation and disclosure support: Wade Sparks III and VulnCheck.
