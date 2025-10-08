# EspoCRM — Extension Upload Remote Code Execution via `scripts/AfterInstall.php` (<= 9.2.1)

**Disclosure status:** Reported to vendor (GitHub Security Advisory GHSA-x3m9-h2gg-68mx). Vendor acknowledged but closed advisory as "by design". No vendor patch available as of this advisory.

**Affected product**
- EspoCRM — versions **<= 9.2.1**

**Summary**
EspoCRM's extension installation mechanism executes `scripts/AfterInstall.php` from uploaded extension packages during installation with the web server user privileges. An authenticated administrator can upload a crafted extension ZIP containing a malicious `scripts/AfterInstall.php`, which executes arbitrary PHP as the web server user (RCE), resulting in full compromise of the application instance (database credentials, admin accounts, filesystem access, persistent backdoors).

**Required privileges**
- Requires an authenticated administrator account with permission to upload/install extensions via the web UI.

**Impact**
- Remote Code Execution (as web server user, typically `www-data`).
- Disclosure of database credentials and encryption keys.
- Compromise of admin accounts and application data.
- Ability to achieve persistence and full host-level compromise in many deployments.

**Proof-of-Concept (high level)**
1. Create an extension package (ZIP) based on an existing extension skeleton (e.g., Real Estate extension).
2. Add or replace the file `scripts/AfterInstall.php` inside the ZIP with PHP code that performs an action on the server (e.g., writes a file, runs a reverse shell, reads config.php to extract DB credentials).  
   - **NOTE:** Full reverse shell payload omitted here for safety.
3. Login as an EspoCRM admin → *Admin → Extensions → Upload Package* and upload the crafted ZIP.
4. The extension installation process executes `scripts/AfterInstall.php` during install; the payload runs as the web server user — proving RCE.

**Confirmed results (observed)**
- Extraction of database credentials.
- Reading of application encryption keys.
- Creation of files under the application directory and continuous access (persistence).
- Full filesystem access within the application container/host (depending on deployment privileges).

**Mitigation / Workarounds**
- Temporary workaround: set `adminUpgradeDisabled => true` in the application config to disable extension uploads via the UI. This blocks the immediate attack vector but does not change the underlying behavior of executing `AfterInstall.php`. (Vendor referenced this configuration option.)
- Recommended permanent fixes for vendor:
  - Do **not** execute arbitrary PHP from uploaded packages during installation.
  - Require cryptographic signing of extension packages, and validate signatures before execution.
  - Run install scripts in a secure sandboxed environment (or use a privileged CLI-only installer with strict verification).
  - Validate contents of uploaded packages and deny any PHP execution during the install process.

**Disclosure timeline**
- 2025-09-30 — Reported via GitHub Security Advisory GHSA-x3m9-h2gg-68mx.
- 2025-10-01 — Vendor commented and closed advisory, suggested `adminUpgradeDisabled` as mitigation.

**References**
- GitHub Security Advisory: GHSA-x3m9-h2gg-68mx (opened on GitHub).  

**Contact**
- Reporter: uartu0@gmail.com

---

This advisory documents a server-side RCE that is reached through the product's extension installation mechanism. Although an admin credential is required to trigger the exploit, the impact is full remote code execution in the web server context and thus meets criteria for a vulnerability record / CVE assignment. If you need the exact PoC code for verification, please contact the reporter for an embargoed/private PoC file or request temporary access to the GitHub advisory containing the PoC.
