junos-radius-ldap-auth
======================

Quick RADIUS server setup using LDAP backend for authentication on Juniper devices

Using a vanilla install of Ubuntu Server install required packages:
````
apt-get install slapd ldap-utils freeradius-ldap
dpkg-reconfigure slapd
````
Initial LDAP database should look something like this:
````
root@auth1:~# slapcat
dn: dc=dyn,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: dyn.com
dc: dyn
structuralObjectClass: organization
entryUUID: b7bb922c-bacb-1033-8caa-d19568de51f9
creatorsName: cn=admin,dc=dyn,dc=com
createTimestamp: 20140818023301Z
entryCSN: 20140818023301.089120Z#000000#000#000000
modifiersName: cn=admin,dc=dyn,dc=com
modifyTimestamp: 20140818023301Z

dn: cn=admin,dc=dyn,dc=com
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: e1NTSEF9Zk96Q1NWVGR4eUY3dk9qMHdkRFRhcmlVMGN0ZHdFWHQ=
structuralObjectClass: organizationalRole
entryUUID: b7be85cc-bacb-1033-8cab-d19568de51f9
creatorsName: cn=admin,dc=dyn,dc=com
createTimestamp: 20140818023301Z
entryCSN: 20140818023301.108625Z#000000#000#000000
modifiersName: cn=admin,dc=dyn,dc=com
modifyTimestamp: 20140818023301Z

````
Create initial LDIF file and load into LDAP:
````
root@auth1:~# ldapadd -H ldap://localhost -x -D "cn=admin,dc=dyn,dc=com" -f /home/jmillay/test.ldif -w secret
adding new entry "ou=groups, dc=dyn, dc=com"

adding new entry "ou=people, dc=dyn, dc=com"

adding new entry "cn=Jeremiah Millay, ou=people, dc=dyn, dc=com"

adding new entry "cn=admins, ou=groups, dc=dyn, dc=com"
````

Edit /etc/freeradius/sites-enabled/default
````
authorize { 
	auth_log 
	ldap
}

authenticate {
        Auth-Type LDAP {
                ldap
        }

}
````
Edit /etc/freeradius/modules/ldap
````
33,36c33,36
< 	server = "localhost"
< 	identity = "cn=admin,dc=dyn,dc=com"
< 	password = secret
< 	basedn = "dc=dyn,dc=com"
---
> 	server = "ldap.your.domain"
> 	#identity = "cn=admin,o=My Org,c=UA"
> 	#password = mypass
> 	basedn = "o=My Org,c=UA"
159c159
< 	set_auth_type = yes
---
> 	# set_auth_type = yes
````

Edit /etc/freeradius/ldap.attrmap and add as a checkItem entry:
````
checkItem       User-Password                   userPassword
 
````
Edit /etc/freeradius/clients.conf and place this at the bottom replacing the IP,secret, and name of your Juniper device:
````
client 192.168.1.114 {
       secret          = "i<3JUNOS"
       shortname       = LAB-SRX-1
}
````
Restart FreeRADIUS
````
service freeradius restart
````
Point your Juniper device to the RADIUS server:
````
root@LAB-SRX-1# show | compare
[edit system]
+  authentication-order [ radius password ];
+  radius-server {
+      192.168.1.146 {
+          secret "$9$NEdwgqmfzn/IEw2oGHkCApBhy"; ## SECRET-DATA
+          timeout 10;
+          retry 1;
+      }
+  }
+  login {
+      user remote {
+          full-name "RADIUS User";
+          uid 9999;
+          class super-user;
+      }
+  }
````
Attempt to log into your Juniper device and verify successul authentication in /var/log/freeradius/radius.log
````
Tue Aug 19 21:45:23 2014 : Auth: Login OK: [jmillay] (from client LAB-SRX-1 port 0 cli 192.168.1.130)
````
