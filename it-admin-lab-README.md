# IT Administrator Lab

Active Directory administration tasks completed in my VirtualBox home lab. Each section is written as a ticket: Issue, Steps, Result.

## Environment

- **DC01:** Windows Server 2022, domain controller for `lab.local` (AD DS, DNS, DHCP)
- **PC1:** Windows 11 Enterprise, domain-joined client on the internal network `labnet`
- **Tools:** Active Directory Users and Computers, Group Policy Management, DHCP console, PowerShell

---

## Ticket 1: Bulk user creation from a CSV file

**Issue:** New hires need accounts in the right departments, and creating them one by one is slow.

**Steps:**
1. Created `users.csv` with first name, last name, username, department, and title for 6 users.
2. Wrote a PowerShell script that reads the CSV and, for each row, creates the department OU and `<Department>-Staff` group if missing, creates the user, and adds the user to the group.
3. Set the script to skip anything that already exists, so it can be rerun safely.
4. Set "User must change password at next logon" on every new account.
5. Verified with `Get-ADUser` and `Get-ADGroupMember`.

**Result:** 6 users created across the HR, Finance, IT, and Sales OUs, each in the correct group.

**Problem found:** The script could not add one Sales user to `Sales-Staff`. The group's `SamAccountName` was `Sale-Staff` (a typo from when it was first created), while its display name was `Sales-Staff`. Fixed with `Set-ADGroup -Identity "Sale-Staff" -SamAccountName "Sales-Staff"`, then added the user.

![Bulk user creation script pasted into PowerShell on DC01](01-script-pasted.png)

![Script error adding a user to Sales-Staff, Finance users created](02-sales-group-error.png)

![Group display name Sales-Staff versus SamAccountName Sale-Staff](03-group-samaccountname-typo.png)

![Group renamed, Sales-Staff members listed, all 6 users in their departments](04-users-created-verified.png)

---

## Ticket 2: Password reset

**Issue:** A user (mlopez) forgot her password and cannot sign in.

**Steps:**
1. Confirmed the user's identity.
2. Reset the password in PowerShell with `Set-ADAccountPassword -Reset`.
3. Forced a change at next logon with `Set-ADUser -ChangePasswordAtLogon $true`.
4. Verified with `Get-ADUser -Properties PasswordLastSet,PasswordExpired`.

**Result:** `PasswordExpired` showed `True`, meaning the user must set a new password at her next sign-in. The same reset can also be done in ADUC (right-click the user, Reset Password).

![Password reset commands and verification showing PasswordExpired True](05-password-reset.png)

---

## Ticket 3: Account lockout and unlock

**Issue:** A user (jchen) cannot sign in after entering the wrong password several times.

**Steps:**
1. Set the domain lockout policy: 3 failed attempts, 30-minute lockout duration and observation window (`Set-ADDefaultDomainPasswordPolicy`).
2. Reproduced the lockout on PC1 with three wrong passwords.
3. Confirmed the lockout with `Search-ADAccount -LockedOut`.
4. Unlocked the account with `Unlock-ADAccount -Identity jchen`.
5. Repeated the unlock in ADUC (user Properties, Account tab, Unlock account).

**Result:** `LockedOut` returned to `False` and the user could sign in again.

![Unlock-ADAccount run and LockedOut showing False for James Chen (top of output)](06-unlock-and-offboard.png)

---

## Ticket 4: Employee offboarding

**Issue:** An employee (jkim) has left the company and the account must be secured.

**Steps:**
1. Created a `Disabled Users` OU.
2. Recorded the account's group memberships before changing anything (`Domain Users`, `HR-Staff`).
3. Disabled the account with `Disable-ADAccount`.
4. Reset the password so the old one cannot be used.
5. Removed the user from `HR-Staff`.
6. Added an offboarding note in the account description.
7. Moved the account to the `Disabled Users` OU.
8. Verified the final state.

**Result:** `Enabled` is `False`, `MemberOf` is empty, the description shows the offboarding note, and the account sits in `OU=Disabled Users`. The account is kept in AD for records.

![Offboarding commands and final verification: disabled, empty MemberOf, Disabled Users OU](06-unlock-and-offboard.png)

---

## Ticket 5: Finance shared drive mapped with Group Policy

**Issue:** Finance staff need a shared folder that only they can access, mapped automatically at sign-in.

**Steps:**
1. Created `C:\Shares\FinanceData` and shared it as `\\DC01\FinanceData`.
2. Set NTFS permissions: `Finance-Staff` Modify, `Administrators` and `SYSTEM` Full Control, inheritance removed.
3. Created the GPO `Map Finance Drive` and linked it to the Finance OU.
4. Added a Drive Maps preference (User Configuration > Preferences > Windows Settings > Drive Maps): `F:` to `\\DC01\FinanceData`.
5. Tested on PC1 with a Finance user, then with an IT user.

**Result:**
- Finance user: `Finance (F:)` appears at sign-in and files can be saved.
- IT user: no `F:` drive.
- IT user opening `\\DC01\FinanceData` directly: "You do not have permission to access."

**Problem found:** My first `icacls` grant failed because two commands ran together on one line, leaving the folder with no permissions after inheritance was removed. Reran the grant on its own and confirmed with `icacls`.

![FinanceData share created and first icacls attempt failing](07-share-created-permissions-error.png)

![Folder permissions fixed: Finance-Staff Modify, Administrators and SYSTEM Full Control](08-permissions-fixed.png)

![Map Finance Drive GPO linked to the Finance OU](09-gpo-scope.png)

![Finance (F:) mapped on PC1 for a Finance user](10-finance-drive-mapped.png)

![Test file saved to Finance (F:)](11-finance-test-file.png)

![IT user on PC1 with no F: drive](12-it-user-no-drive.png)

![IT user denied access to \\DC01\FinanceData](13-access-denied.png)

---

## Ticket 6: Troubleshooting "domain isn't available" at sign-in

**Issue:** A domain user could not sign in on PC1. Error: "We can't sign you in with this credential because your domain isn't available."

**Steps:**
1. Signed in with a previously used account (cached credentials) to investigate.
2. `ipconfig /all` showed `DHCP Enabled: No` and a static address of `192.168.50.10`, left over from an earlier VPN lab. `ipconfig /renew` failed with "no adapter is in the state permissible for this operation."
3. Switched the adapter back to DHCP with `netsh interface ip set address name="Ethernet" source=dhcp`. Pinged DC01 successfully.
4. `nslookup` still failed. On DC01, found the adapter's DNS server list included a mistyped address (`192.168.10.101`). Corrected it to `192.168.10.10`.
5. Checked the DHCP scope's DNS option (006), which was correct (`192.168.10.10`).
6. Found DC01 was authorized in AD for DHCP under an old address (`10.0.2.15`) from `Get-DhcpServerInDC`. Authorized `192.168.10.10` with `Add-DhcpServerInDC`, removed the old entry with `Remove-DhcpServerInDC`, and restarted the DHCP service.
7. On PC1, set DNS back to automatic, renewed the lease, and flushed the DNS cache.

**Result:** PC1 received `192.168.10.100` from DC01 with DNS `192.168.10.10`. The domain resolved and domain users could sign in again.

**Takeaway:** "Domain isn't available" is usually a network or DNS problem, not an account problem. Check the IP address, DHCP status, and DNS server before touching the account.

![Domain isn't available sign-in error on PC1](14-domain-unavailable-error.png)

![Broken ipconfig /all: DHCP disabled, static 192.168.50.10](15-static-ip-dhcp-off.png)

![Ping to DC01 works but nslookup fails](16-ping-ok-dns-fail.png)

![DC01 ipconfig /all showing the mistyped DNS server](17-dc01-dns-typo.png)

![DHCP scope option 006 DNS Servers set to 192.168.10.10](18-dhcp-scope-dns-option.png)

![DHCP server authorized under the old address 10.0.2.15](19-dhcp-authorization-stale.png)

![DHCP authorization after removing the stale entry](20-dhcp-authorization-fixed.png)

![Fixed ipconfig /all on PC1: DHCP enabled, 192.168.10.100, DNS 192.168.10.10](21-pc1-fixed-ipconfig.png)
