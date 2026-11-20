# Microsoft 365 / Entra ID / Intune Lab

I set up a small fake company in a Microsoft 365 Business Premium trial tenant to practice
the cloud side of IT support: managing users and groups, securing sign-ins, and managing a
Windows laptop with Intune. The "employee laptop" is a Windows 11 Enterprise VM in VirtualBox.

Everything here was done by hand in the admin portals, and I tested each part from the
user's side, not just the admin side.

## The company

| Department | Employees |
|---|---|
| IT | Andrii Khmelevskyi (admin), Daniel Reyes (help desk) |
| Sales | Emma Clark, Liam Turner, Sofia Martinez |
| HR | Olivia Bennett, Noah Patel |
| Finance | Ava Nguyen, Ethan Brooks |
| Marketing | Mason Wright (later offboarded) |

## 1. Users, groups, and roles

- Created the first user by hand (onboarding), then added 8 more in bulk from a CSV file
- Built **dynamic security groups** per department (`SG-Sales`, `SG-HR`, etc.). People are added
  and removed automatically based on their Department field
- Gave Daniel the **Helpdesk Administrator** role and tested it by signing in as him and
  resetting a regular user's password (least privilege: he can't change tenant settings)

![Users](screenshots/01-users.png)
![Groups](screenshots/02-groups.png)
![SG-Sales members](screenshots/03-sg-sales-members.png)
![Help desk resets a password](screenshots/04-helpdesk-resets-emma.png)

## 2. Sign-in security

- Created an **emergency access (break-glass) account** first, excluded from all policies,
  so a bad policy can't lock me out of the tenant
- Turned off Security Defaults and replaced them with my own **Conditional Access** policies:
  - `CA01` – require MFA for all users
  - `CA02` – block sign-ins from outside the United States (using a named location)
- Enabled **self-service password reset**, and tested it as a user with Microsoft Authenticator

![Conditional Access policies](screenshots/05-ca-policies.png)
![Named location](screenshots/06-named-location.png)
![Self-service password reset](screenshots/07-sspr.png)

## 3. Intune device management

- Turned on automatic MDM enrollment (it was set to None)
- Joined the Windows 11 VM to Entra ID as an employee, so it enrolled in Intune automatically
- Created a **compliance policy** (password, firewall, antivirus, Defender, real-time protection)
- Deployed **Company Portal** as a required app, only to the Sales group
- Pushed an **Edge configuration policy** (company new tab page) and confirmed it in `edge://policy`
- Sent a **remote restart** from Intune and watched the VM get the notice and reboot

![Compliant corporate device and remote restart](screenshots/14-device-compliant-restart.png)
![Reboot notice on the laptop](screenshots/15-reboot-notice.png)
![Company Portal installed](screenshots/16-company-portal.png)
![Edge policy applied](screenshots/17-edge-policy.png)
![Compliance policy](screenshots/18-compliance-policy.png)

## 4. Offboarding

Mason Wright "left the company." In this order:

1. Blocked sign-in
2. Signed him out of all sessions
3. Reset his password
4. Cleared his department, so he dropped out of `SG-Marketing` automatically
5. Removed his license, which freed it up for someone else

Access goes first, licenses last. If the license is removed first, the person still has access
for a few minutes.

![Sign-in blocked](screenshots/19-mason-blocked.png)
![License removed](screenshots/20-mason-license.png)
![Account disabled](screenshots/21-mason-disabled.png)

## Problems I ran into

**The app was stuck on "Waiting."** Company Portal never installed on the first laptop.
I found that the Intune Management Extension service, which installs apps, was missing even
though its log folder existed. The log showed it had been running normally and then just stopped.
The VM's clock had been wrong by two days when it enrolled, which I think broke things.
I fixed the clock, then re-enrolled the device.

![App waiting](screenshots/09-app-waiting.png)
![IME log](screenshots/10-ime-log.png)

**I deleted the wrong device.** After rolling the VM back to a snapshot, the "new" laptop had
the same computer name as the old one, because Windows sets the name during install. I deleted
the old record in Intune and Entra, but it was actually the new one, so the user couldn't sign in
("No mapping between account names and security IDs"). Lesson: check the device ID, not the name,
before deleting.

**The laptop showed up as "Personal" and Noncompliant.** During Windows setup a local account
was created, and the work account was only added on top. `dsregcmd /status` showed
`AzureAdJoined : NO`, so the device was only Entra *registered* (like a personal BYOD laptop),
not Entra *joined*. I disconnected the work account, used "Join this device to Microsoft Entra ID,"
and signed in with the work account. After that it showed as Corporate and Compliant.

![Personal and noncompliant](screenshots/11-personal-noncompliant.png)
![dsregcmd before](screenshots/12-dsregcmd-no.png)
![dsregcmd after](screenshots/13-dsregcmd-yes.png)

**Small things:**
- New users couldn't sign in to Windows until they changed their temporary password on the web first
- I almost removed the license from the wrong user during offboarding and had to put it back
- BitLocker and Secure Boot checks are left out of the compliance policy, because they often
  fail in a VM even when the settings are correct

## What I'd do next

- Rebuild my on-premises Active Directory lab and sync it to Entra ID with Entra Connect (hybrid identity)
- Add a compliance-based Conditional Access policy, so only compliant devices can open company apps
