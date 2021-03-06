# Unauthorized Errorlog [Red Hat RHCSA ®/ RHCE® 7 Cert Guide](http://www.sandervanvugt.com/book-red-hat-rhcsa-rhce-7-cert-guide/)

## Chapter 2: Using Essential Tools
- p51 man pages index caches can also be **U** pdated with `man -u` instead of `mandb`.  
  To play around with indexes, delete them all with `find /var/cache/man/ -name index.db -delete` see `man man`
- p

## Chapter 4: Working with Text Files
|  | Glob | regex|
|--|--|--|
| `*` | any string including empty | 0 or more of preceding character |
| `?` | match any _single_ character | 1 or more of preceding character |
| `^` |  | start of string |
| `!` | negate, eg **[!a-d]** any _except_ a-d |  **\[!\]** |
| `.` | match files starting with *.extention*, eg. `tar c .` or `cp .* ~ `| any character except newline **a.c**: abc, aac, a2c |
| `\` | remove special meaning of **?**, *****, **[** & **]** | |
| `{ }` | terms are seperated with commas eg `cp {*.pdf,*.doc}` |  |
| `[ ]` | specifies a range eg **[a-z]** or **[a,o,u]**|  |
| `' '`| escape bash expansion (globbing) eg `touch 'foo*'` |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  | x |
for more details see `man glob` and `man 7 regex` or [Wildcards mini-guide](http://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm)



## Chap 6
- p137 step 6. `for i in lucy, lori, bob;do useradd $i; done` --> `for i in lucy lori bob;do useradd $i; done`
- p139 wrong **groupmems -g sales -l** "This shows users who are a member of this group as a secondary group assignment~~, but also users who are a member of this group as the primary group assignment~~."
<br />**lid -g <groupname>** shows both primary and secondary group memberships eg try:

  ```bash
  for i in lucy lori bob; do useradd $i; done
  usermod -g lucy bob     #make group lucy the primary group of bob
  usermod -aG lucy lori   #add lori to the group lucy
  groupmems -g lucy -l
  lid -g lucy
  lid lori
  ```
- p142 `nslcd service is configured and started when using autconfig-tui` —> `sssd service is configured and started when using authconfig-tui` (with an h). Maybe a change from rhel6? Try the exercise and then run `find / -name nslc` or `systemctl list-unit-files | grep -e sssd -e nslcd` and you'll find nothing related to nslcd.
<br /> see `man authconfig | grep -A2 nslcd` and notice **Used to**
- p145 Step 5 first create the directory `mkdir /etc/openldap/cacerts` then get the certificate with `scp labipa.example.com:/root/cacert.p12 /etc/openldap/cacerts` otherwise scp will create a file cacerts not a file /etc/openldap/cacerts/ca.crt.

- p142 **authconfig-tui** command is _deprecated_, see `man authconfig-tui | grep depr`. [CertDepot](https://www.certdepot.net/ldap-client-configuration-authconfig/) has two alternative exercises:
  #### the nslcd option
  see https://arthurdejong.org/nss-pam-ldapd/setup
  ```bash
  yum install -y openldap-clients nss-pam-ldapd
  sed -i 's/FORCELEGACY=no/FORCELEGACY=yes/g' /etc/sysconfig/authconfig
  sed -i 's/USESSSD=yes/USESSSD=no/g' /etc/sysconfig/authconfig
  grep -E "USESSSDAUTH|USESSSD|FORCELEGACY" /etc/sysconfig/authconfig
  mkdir -p /etc/openldap/cacerts
  scp labipa.example.com:/etc/ipa/ca.crt /etc/openldap/cacerts/
  authconfig --update \
             --enableldap --enableldapauth --enableldaptls \
             --enableforcelegacy --disablesssd --disablesssdauth\
             --ldapserver="labipa.example.com" \
             --ldapbasedn="dc=example,dc=com"
  systemctl list-unit-files | grep -e sssd -e nslcd
  nslcd -V
  grep -v -e "#" -e "^$" /etc/nslcd.conf
  authconfig --test | grep SSSD # SSSD ... disabled
  grep -E "pam_sss|pam_ldap|pam_krb5|$"  /etc/pam.d/system-auth  
  getent passwd ldapuser1
  su - ldapuser1
  whoami
  exit
  authconfig --enablemkhomedir --update
  su - ldapuser1
  pwd
  exit
  ```

  #### the System Security Services Daemon option
  With CentOS 7.0 (images V2.0) you might try sssd on Server2 instead (/usr/lib/systemd/systemd-logind crashed on Server1 with CentOS 7.0)
  ```bash
  yum install -y sssd # on Server2
  mkdir -p /etc/openldap/cacerts
  scp labipa.example.com:/etc/ipa/ca.crt /etc/openldap/cacerts/
  grep -E "USESSSDAUTH|USESSSD|FORCELEGACY" /etc/sysconfig/authconfig
  authconfig --update \
             --enableldap --enableldapauth --enableldaptls \
             --disableforcelegacy  --enablesssdauth --enablesssd \
             --ldapserver="labipa.example.com" \
             --ldapbasedn="dc=example,dc=com" \
             --enablemkhomedir
   systemctl list-unit-files | grep -e sssd -e nslcd
   grep -v -e "#" -e "^$" -e "^\[" /etc/sssd/sssd.conf
   authconfig --test | grep SSSD # SSSD ... *enabled*
   grep -E "pam_sss|pam_ldap|pam_krb5|$"  /etc/pam.d/system-auth  
   getent passwd ldapuser1
   su - ldapuser1
   id
   pwd
   exit
  ```
  To get a better understanding of /etc/pam.d/system-auth, read `man pam.d` (or equivalent `man 5 pam.conf`). See p567
  #### /etc/pam.d/system-auth compare nslcd and sssd
  ```txt
  NSLCD / yum install -y openldap-clients nss-pam-ldapd                     SSSD / yum install -y sssd
  #%PAM-1.0                                                                 #%PAM-1.0
  # This file is auto-generated.                                            # This file is auto-generated.
  # User changes will be destroyed the next time authconfig is run.         # User changes will be destroyed the next time authconfig is run.
  auth        required      pam_env.so                                      auth        required      pam_env.so
  auth        sufficient    pam_fprintd.so                                <
  auth        sufficient    pam_unix.so nullok try_first_pass               auth        sufficient    pam_unix.so nullok try_first_pass
  auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success     auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
  auth        sufficient    pam_ldap.so use_first_pass                    | auth        sufficient    pam_sss.so use_first_pass
  auth        required      pam_deny.so                                     auth        required      pam_deny.so

  account     required      pam_unix.so broken_shadow                       account     required      pam_unix.so broken_shadow
  account     sufficient    pam_localuser.so                                account     sufficient    pam_localuser.so
  account     sufficient    pam_succeed_if.so uid < 1000 quiet              account     sufficient    pam_succeed_if.so uid < 1000 quiet
  account     [default=bad success=ok user_unknown=ignore] pam_ldap.so    | account     [default=bad success=ok user_unknown=ignore] pam_sss.so
  account     required      pam_permit.so                                   account     required      pam_permit.so

  password    requisite     pam_pwquality.so try_first_pass local_users_o   password    requisite     pam_pwquality.so try_first_pass local_users_o
  password    sufficient    pam_unix.so sha512 shadow nullok try_first_pa   password    sufficient    pam_unix.so sha512 shadow nullok try_first_pa
  password    sufficient    pam_ldap.so use_authtok                       | password    sufficient    pam_sss.so use_authtok
  password    required      pam_deny.so                                     password    required      pam_deny.so

  session     optional      pam_keyinit.so revoke                           session     optional      pam_keyinit.so revoke
  session     required      pam_limits.so                                   session     required      pam_limits.so
  -session     optional      pam_systemd.so                                 -session     optional      pam_systemd.so
  session     optional      pam_oddjob_mkhomedir.so umask=0077            | session     optional      pam_mkhomedir.so umask=0077
  session     [success=1 default=ignore] pam_succeed_if.so service in cro   session     [success=1 default=ignore] pam_succeed_if.so service in cro
  session     required      pam_unix.so                                     session     required      pam_unix.so
  session     optional      pam_ldap.so                                   | session     optional      pam_sss.so  
  ```
  grep -v -e "#" -e "^$"  /etc/nsswitch.conf
  ```text
  nslcd:                                         sssd:
  passwd:     files sss ldap                  |  passwd:     files sss
  shadow:     files sss ldap                  |  shadow:     files sss
  group:      files sss ldap                  |  group:      files sss
  hosts:      files dns                          hosts:      files dns
  bootparams: nisplus [NOTFOUND=return] files    bootparams: nisplus [NOTFOUND=return] files
  ethers:     files                              ethers:     files
  netmasks:   files                              netmasks:   files
  networks:   files                              networks:   files
  protocols:  files                              protocols:  files
  rpc:        files                              rpc:        files
  services:   files sss                          services:   files sss
  netgroup:   files sss ldap                  |  netgroup:   files sss
  publickey:  nisplus                            publickey:  nisplus
  automount:  files ldap                      |  automount:  files sss
  aliases:    files nisplus                      aliases:    files nisplus
  ```

## Chap 7 : lapipa
1
2
3
scp root@server1.example.com:/etc/yum.repos.d/labipa.repo /etc/yum.repos.d/

### Chap 8 Networking
In https://www.safaribooksonline.com/library/view/learning-path-red/9780134664040/RCSA_01_09_08.html favorites mentions netstat —> ip -s link & ss -tulpen, see https://dougvitale.wordpress.com/2011/12/21/deprecated-linux-networking-commands-and-their-replacements/
Appendix A “10. tar xvf /tmp/etchome.tgz /etc/passwd” should be “tar xvf /tmp/etchome.tgz etc/passwd"
Appendix B: Table 2.4 remove
“at (i) or after (a) the current cursor position.”
 “The ! forces …are doing”
“, which allows you … the selection”
“Use d to cut, or y to copy the selection.”
Appendix B: Why is "Table 3.6 Overview of tar Options” missing?
Appendix B & Appendix C: Table 2.4 add
in visual mode use to cut
in visual mode use to copy
Appendix C: Table 2.4 add
^ or 0

### Chap 18 Managing and Understanding the Boot Procedure
- p413 _Understanding Wants_: **default** .wants are in **/usr/lib**/systemd/system/*.wants, system-specific (created with `systemctl enable`) wants are in **/etc**/systemd/system/*.wants **
- p147 from [Understanding systemd units and unit files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files#unit-section-directives)
  - `[Unit]`
    - **Wants=**: This directive is similar to Requires=, but less strict. Systemd _will attempt_ to start any units listed here when this unit is activated. If these units are not found or fail to start, the current unit will **continue** to function. This is the recommended way to configure most dependency relationships. Again, this implies a parallel activation unless modified by other directives.
    - **Requires=**: This directive lists any units upon which this unit _essentially_ depends. If the current unit is activated, the units listed here _must_ successfully activate as well, else this unit will **fail**. These units are started _in parallel_ with the current unit by default.
  - `[Install]`
    - **WantedBy=:**
    <br />The WantedBy= directive is the **most common way** to specify how a unit should be enabled. This directive allows you to specify a dependency relationship in a similar way to the Wants= directive does in the [Unit] section. The difference is that this directive is included in the ancillary unit allowing the primary unit listed **to remain relatively clean**. When a unit with this directive is enabled, a _directory_ will be created within /etc/systemd/system named after the specified unit with `.wants` appended to the end. Within this, a symbolic link to the current unit will be created, creating the dependency. For instance, if the current unit has WantedBy=multi-user.target, a directory called multi-user.target.wants will be created within /etc/systemd/system (if not already available) and a symbolic link to the current unit will be placed within. Disabling this unit removes the link and removes the dependency relationship.
    - **RequiredBy=**: This directive is very similar to the WantedBy= directive, but instead specifies a required dependency that will cause the activation to **fail** if not met. When enabled, a unit with this directive will create a directory ending with **.requires**.
- from _man 7 systemd.unit_:
  - Along with a unit file foo.service, the directory foo.service**.wants/** may exist. All unit files symlinked from such a directory are **implicitly** added as dependencies of _type Wants=_ to the unit. This is useful to hook units into the start-up of other units, **without having to modify their unit files.** For details about the semantics of Wants=, see below. The preferred way to create symlinks in the .wants/ directory of a unit file is with the enable command of the systemctl(1) tool which reads information from the [Install] section of unit files (see below). A similar functionality exists for Requires= type dependencies as well, the directory suffix is .requires/ in this case.
  - **Wants=**
  <br />A weaker version of _Requires=_. Units listed in this option will be started if the configuring unit is. However, if the listed units fail to start or cannot be added to the transaction this has no impact on the validity of the transaction as a whole. This is the
  recommended way to hook start-up of one unit to the start-up of another unit.
  <br />Note that dependencies of this type _may also be configured_ outside of the unit configuration file **by adding symlinks** to a .wants/ directory accompanying the unit file.
- p417 `cat /usr/lib/systemd/system/iptables.service` file doesn't exist on Server2 (non-GUI)
- p418 on Server**2** `systemctl start iptables` generates message `Failed to issue method call: Unit iptables.service failed to load: No such file or directory.` as iptables is not installed on non-GUI. Therefor `systemctl enable iptables`also failes on Server2. <br /> Notice that `systemctl mask iptables` **does** generate the link to /dev/null although the services is not installed on Server2.
- p149 from _man systeemctl_:
  <br />`systemctl isolate`
  <br />Start the unit specified on the command line and its dependencies and **stop all others**.
  <br />This is similar to changing the runlevel in a traditional init system. The isolate command will **immediately stop** processes that are not enabled in the new unit, possibly including the graphical environment or terminal you are currently using.
  <br /> Note that this is allowed **only** on units where **AllowIsolate=** is enabled.

  ## Chapter 16: Basic Kernel Management
- p383 demonstrate how to create options in an /etc/modprobe.d/\*.conf file with options that will persist after reboot.  
Notice that these commands disconnect the network, run `modprobe -r` from the console, put the commands in a bash-script (and hope tcp will not notice the disconnection) or reboot.  
In a second tty (eg ctrl-leftalt-F2 or `chvt 2`) run `udevadm monitor` and see the udev events.
    ```bash
  modinfo e1000 | grep -e parm: -e description: # what params do we have?
  ethtool eth0 | grep -A2 "Advertised link modes" # what is the advertised Speed?
  echo "options e1000 Speed=100" > /etc/modprobe.d/e1000.conf
  modprobe -r e1000
  modprobe e1000
  ethtool eth0 | grep -A2 "Advertised link modes"
  rm -f /etc/modprobe.d/e1000.conf
  modprobe -r e1000
  modprobe e1000
  ethtool eth0 | grep -A2 "Advertised link modes"
    ```
    See `man 5 modprobe.d` for details on the .conf file.

### Chapter 25: Configuring External Authentication and Authorization
- p563 see service principles with passwords in keytab using `klist -kt /etc/krb5.keytab`
- p563 Centos 7.2 `man authconfig` states "The **authconfig-tui** is _deprecated_. No new configuration settings will be supported by its text user interface. Use **system-config-authentication GUI** application or the command line options instead." dated 22 July 2011

```bash
┌────────────────┤ Authentication Configuration ├─────────────────┐
│                                                                 │
│  User Information        Authentication                         │
│  [*] Cache Information   [*] Use MD5 Passwords                  │
│  [*] Use LDAP            [*] Use Shadow Passwords               │
│  [ ] Use NIS             [*] Use LDAP Authentication            │
│  [ ] Use IPAv2           [ ] Use Kerberos                       │
│  [ ] Use Winbind         [ ] Use Fingerprint reader             │
│                          [ ] Use Winbind Authentication         │
│                          [*] Local authorization is sufficient  │
│                                                                 │
│            ┌────────┐                      ┌──────┐             │
│            │ Cancel │                      │ Next │             │
│            └────────┘                      └──────┘             │
│                                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
- p569 for the exercises to work, create a few accounts first
https://www.certdepot.net/sys-understand-authconfig/
```bash
dig labipa.example.com
yum groups info Directory\ Client
yum groups install Directory\ Client -y
# make sure that sssd is running
systemctl status sssd
mkdir -p /etc/openldap/cacerts
scp labipa.example.com:/etc/ipa/ca.crt /etc/openldap/cacerts
sed -i 's/USESSSDAUTH=no/USESSSDAUTH=yes/g' /etc/sysconfig/authconfig
grep -E "USESSSDAUTH|USESSSD|FORCELEGACY" /etc/sysconfig/authconfig

# sssd service running, "su - ldapuser1" works fine
authconfig  --update \
            --enablecachecreds \
            --disableforcelegacy \
            --enableshadow \
            --enableldap \
            --enableldapauth \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --enableldaptls \
            --enablemkhomedir


  # without this package 'su - ldapuser1' fails
# if you forgot, install nnss-pam-ldap and run authconfig again
# no sssd installed, only ldap and krb5. kinit ldapuser1 & su - ldapuser1 succeed:
authconfig  --update \
            --disableforcelegacy \
            --enableshadow \
            --enableldap \
            --enableldapauth \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --enableldaptls \
            --enablekrb5 \
            --krb5kdc="labipa.example.com" \
            --krb5adminserver="labipa.example.com" \
            --krb5realm="EXAMPLE.COM" \
            --enablekrb5kdcdns \
            --enablekrb5realmdns \
            --enablemkhomedir
--update \        # update configuration files
--disableforcelegacy # use SSSD implicitly
--enableshadow \  # enable shadowed passwords by default
--enableldap \    # enable LDAP for user information by default
--enableldapauth \ # enable LDAP for authentication by default
--ldapserver="labipa.example.com" \
--ldapbasedn="dc=example,dc=com" \
--enableldaptls \
--enablekrb5 \    # enable kerberos authentication by default
--krb5kdc="labipa.example.com" \ # default kerberos KDC
--krb5adminserver="labipa.example.com" \
--krb5realm="EXAMPLE.COM" \ # default kerberos realm
--enablekrb5kdcdns \ # enable use of DNS to find kerberos KDCs
--enablekrb5realmdns \ # enable use of DNS to find kerberos realms
--enablemkhomedir #\


# with both ldap disabled "kinit ldapuser1" works fine, "su - ldapuser1" fails with "su: user ldapuser1 does not exist":
authconfig  --update \
            --disableforcelegacy \
            --enableshadow \
            --disableldap \
            --disableldapauth \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --enableldaptls \
            --enablekrb5 \
            --krb5kdc="labipa.example.com" \
            --krb5adminserver="labipa.example.com" \
            --krb5realm="EXAMPLE.COM" \
            --enablekrb5kdcdns \
            --enablekrb5realmdns \
            --enablemkhomedir

# ldapauth disabled, no sssd installed, only ldap and krb5. kinit ldapuser1 & su - ldapuser1 succeed:
authconfig  --update \
            --disableforcelegacy \
            --enableshadow \
            --enableldap \
            --disableldapauth \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --enableldaptls \
            --enablekrb5 \
            --krb5kdc="labipa.example.com" \
            --krb5adminserver="labipa.example.com" \
            --krb5realm="EXAMPLE.COM" \
            --enablekrb5kdcdns \
            --enablekrb5realmdns \
            --enablemkhomedir

# both nslcd and sssd are running(?) or enabled(?)
# kinit ldapuser1; su - ldapuser1 works fine
authconfig  --update \
            --disableforcelegacy \
            --enablesssd \
            --enablesssdauth \
            --enableshadow \
            --enableldap \
            --disableldapauth \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --enableldaptls \
            --enablekrb5 \
            --krb5kdc="labipa.example.com" \
            --krb5adminserver="labipa.example.com" \
            --krb5realm="EXAMPLE.COM" \
            --enablekrb5kdcdns \
            --enablekrb5realmdns \
            --enablemkhomedir

# both nslcd and sssd are running
# kinit ldapuser1; su - ldapuser1 works fine
# sssd service failed
yum groups install Directory\ Client -y
yum install -y nss-pam-ldapd
yum install -y sssd nss-pam-ldapd pam_krb5 krb5-workstation
authconfig  --update \
            --enableldap --enableldapauth --enableldaptls \
            --disableforcelegacy  --enablesssdauth --enablesssd \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --enablemkhomedir
# nslcd.service                               disabled
# sssd.service                                enabled
authconfig  --update \
            --enableldap --enableldapauth --enableldaptls \
            --disableforcelegacy  --enablesssdauth --enablesssd \
            --enablekrb5 --enablekrb5kdcdns --enablekrb5realmdns \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --krb5kdc="labipa.example.com" \
            --krb5adminserver="labipa.example.com" \
            --krb5realm="EXAMPLE.COM" \
            --enablemkhomedir
# nslcd.service                               enabled
# sssd.service                                enabled

authconfig  --update \
            --enableldap --enableldapauth --enableldaptls \
            --disableforcelegacy  --enablesssdauth --enablesssd \
            --enablekrb5 --disablecachecreds --disablecache \
            --ldapserver="labipa.example.com" \
            --ldapbasedn="dc=example,dc=com" \
            --krb5kdc="labipa.example.com" \
            --krb5adminserver="labipa.example.com" \
            --krb5realm="EXAMPLE.COM" \
            --enablemkhomedir


systemctl --type=service | grep -e sssd -e nslcd -e UNIT
grep -v -e "#" -e "^$"  /etc/nsswitch.conf
nslcd -V
grep -E "pam_sss|pam_ldap|pam_krb5|$"  /etc/pam.d/system-auth
grep -v -e "#" -e "^$" /etc/nslcd.conf
grep -v -e "#" -e "^$" -e "\[" /etc/sssd/sssd.conf
authconfig --test | grep SSSD # SSSD ... *enabled*
systemctl restart sssd
systemctl status -l sssd
kinit admin
getent passwd ldapuser1
su - ldapuser1
authconfig-tui
yum search pam_ldap
yum search nss-pam-ldapd
yum -y install nss-pam-ldapd
authconfig-tui
kinit admin
kinit ldapuser1
```

[IdM client install manual](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Linux_Domain_Identity_Authentication_and_Policy_Guide/Installing_the_IPA_Client_on_Linux.html) RHEL 7
```bash
systemctl list-unit-files | grep -e chronyd -e ntpd
systemctl mask chronyd.service # see p863
yum -y install ipa-client
# make sure the admin password is not expired on labipa
ipa-client-install  --force-join \
                    --domain="example.com" \
                    --mkhomedir --force-ntpd \
                    --request-cert \
                    -p admin -w password -U #unattended
systemctl --type=service  | grep -E "chronyd|ntpd|sssd|nslcd" # no nslcd!
grep -E "pam_sss|pam_ldap|pam_krb5|$"  /etc/pam.d/system-auth # no pam_ldap nor pam_krb5 !!
klist -kt /etc/krb5.keytab
systemctl status -l systemd-logind # everthing ok? if not try:
grep systemd-logind /var/log/audit/audit.log | audit2allow -M mypol;semodule -i mypol.pp

```
### Chapter 23: Configuring Remote Mounts and FTP
- p533 to change defaults, on labipa run: `ipa config-mod --homedirectory /home/ldap --defaultshell /bin/bash`  
to change existing users, on labipa run `for i in `ipa user-find | grep "User login: " | sed 's/User login: //g'`; do ipa user-mod $i --homedir=/home/ldap/$i --shell=/bin/bash; done`

### Chapter 25: Configuring External Authentication and Authorization
- p567 To get a better understanding of /etc/pam.d/system-auth, read `man pam.d` (or equivalent `man 5 pam.conf`).

### Chapter 34: Configuring DNS
- p750 Exercise 34.1
  ```bash
yum clean all                               # server2 V3.0 has issues with yum
yum -y install unbound vim bind-utils       # bind-utils for dig
yum list installed | grep unbound
ss -tulpen | grep :53                       # is anything bound to port 53?
systemctl start unbound; systemctl enable unbound
ss -tulpen | grep :53                       # is unbound bound to ip-adresses?
cp -n /etc/unbound/unbound.conf ~           # make a backup, just in case
# vim /etc/unbound/unbound.conf             # see p750
# listen on all interfaces
sed -i 's/# interface: 0.0.0.0$/interface: 0.0.0.0/g' /etc/unbound/unbound.conf
# accept requests from 192.168.4.0/24
sed -i 's/# access-control: ::ffff:127.0.0.1 allow/# access-control: ::ffff:127.0.0.1 allow\n\taccess-control: 192.168.4.0\/24 allow/g' /etc/unbound/unbound.conf
# forward all requests to 192.168.4.200
echo $'forward-zone:\n\tname: \".\"\n\tforward-addr: 192.168.4.200' | tee -a /etc/unbound/unbound.conf
# don't require DNSSEC validation for the example.com domain
sed -i 's/# domain-insecure: "example.com"/domain-insecure: "example.com"/g' /etc/unbound/unbound.conf
unbound-checkconf
firewall-cmd --permanent --add-service=dns; firewall-cmd --reload
firewall-cmd --list-all
systemctl restart unbound
ss -tulpen | grep :53                       # is unbound bound to ip-adresses?
man dig | grep -A5 adflag
dig @192.168.4.220 example.com DNSKEY       # no ad-bit
dig @192.168.4.220 rhatcert.com DNSKEY  | grep -E " ad|$"   
  ```
- p751 Listing 34.2 can not be true:
  - _either_ the domain is signed and returns the `ad` "Authenticated Data" flag,
  - _or_ the domain is not signed, doesn't return the ad-bit and can't return DNSKEYs.  
Try it yourself and verify that the rhatcert.com domain now is signed and the AD-flag is shown.
- p752 TIP part2: using `unbound-control-setup` is far from travial. Unless practised, don't try on the exam.
- p752 you get the trust anchors  
- p752


### Chapter 40: Managing Time Synchronization
- p863 chronyd is not compatible with ipa, and [this won't change any soon](https://fedorahosted.org/freeipa/ticket/4963#comment:5)

### Chapter 41: Final Preparation
- p877 Selfstudy candidates can take two types of exams

|  | individual exam | classroom exam |  
|--|--|--|
| location | **few** locations, [check here](https://www.redhat.com/en/services/training/locations-facilities) | multiple locations |
| date/time | self select, upon seat availability | **limited** dates, fixed time & date  |
| price  | $/€500+VAT | $/€500+VAT |
| designation |  EX200K or EX300K \(Kiosk\)| EX200 or EX300 |
| test provider | [PSI](https://www.examslocal.com/) | RedHat |
| SSO provider | RedHat account | RedHat account |




RedHat now refers to *individual exams* to what Sander calls **Kiosk** exams and they have a trailing **K** eg EX200K. You can find them listed at [PSI](https://www.examslocal.com/). Take the following steps to book an kiosk exam at you local training provider.
  - create an account at RedHat, do not use your Linux Foundation credentials as at the test location authentication with Linux Foundation will *fail*, see [SSO login problem that prevents the exam from being taken](https://www.certdepot.net/bad-experience-redhat-labs-exam/).
  - [login at RedHat](https://www.redhat.com/wapps/ugc/protected/account.html) and in EU verify you have you VAT number added (*if you have one*).
  - select an **individual** exam [EX200](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam#)/[EX300](https://www.redhat.com/en/services/training/ex300-red-hat-certified-engineer-rhce-exam)
  - then select *Enroll*
  - then select the city where you want to take the exam
  - add to cart
  - pay for your exam
  - it might take a few days for your voucher code to become available.
  - you get a voucher code that you can use to book at the selected venue at a date & time you prefer. The voucher is valid for 1 year.
  - ???
