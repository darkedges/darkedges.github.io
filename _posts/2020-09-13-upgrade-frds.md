---
id: 40
title: "Upgrading ForgeRock Directory Services 6.5.x to 7.0.0"
date: 2020-09-13T16:00:00+00:00
author: admin
layout: post
guid: http://www.darkedges.com/blog/?p=40
permalink: /2020/09/13/upgrade-frds/
categories:
  - forgerock
  - frds
  
---

## Pre-Requisites

The following files are required to be downloaded to `/tmp` of your Linux server.

- OpenJDK8U-jre_x64_linux_openj9_8u265b01_openj9-0.21.0.tar.gz
- OpenJDK11U-jre_x64_linux_openj9_11.0.8_10_openj9-0.21.0.tar.gz
- DS-eval-6.5.1.zip
- DS-7.0.0.zip

The OpenJDK Files can be dowwnloaded from [https://adoptopenjdk.net/releases.html?variant=openjdk11&jvmVariant=openj9](https://adoptopenjdk.net/releases.html?variant=openjdk11&jvmVariant=openj9) and the ForgeRock files from [https://backstage.forgerock.com](https://backstage.forgerock.com)

Then we will unarchive the two JRE versions 

```bash
cd /tmp
tar zxvf OpenJDK8U-jre_x64_linux_openj9_8u265b01_openj9-0.21.0.tar.gz
tar zxvf OpenJDK11U-jre_x64_linux_openj9_11.0.8_10_openj9-0.21.0.tar.gz
```

## Upgrade  ForgeRock Directory Service

### Deploy FRDS 6.5.1

First we need to install an existing service. For this example I am using FRDS 6.5.1.
This uses JDK 8

```bash
export JAVA_HOME=/tmp/jdk8u265-b01-jre
cd /tmp
unzip DS-eval-6.5.1.zip
/tmp/opendj/setup directory-server \
    --instancePath /tmp/opendj-instance \
    --rootUserDn "cn=Directory Manager" \
    --rootUserPassword Passw0rd \
    --productionMode \
    --hostname localhost \
    --adminConnectorPort 4444 \
    --ldapPort 1389 \
    --ldapsPort 1636 \
    --httpsPort 8443 \
    --profile am-config:6.5.0 \
    --profile am-cts:6.5.0 \
    --profile am-identity-store:6.5.0 \
    --set am-config/amConfigAdminPassword:Passw0rd \
    --set am-cts/amCtsAdminPassword:Passw0rd \
    --set am-cts/tokenExpirationPolicy:ds \
    --set am-identity-store/amIdentityStoreAdminPassword:Passw0rd \
    --acceptLicense
```

should return

```bash
Validating parameters..... Done
Configuring certificates..... Done
Configuring server..... Done
Configuring profile AM 6.5 configuration data store....... Done
Configuring profile AM 6.5 CTS data store......... Done
Configuring profile AM 6.5 identity data store........ Done
Starting directory server................... Done

To see basic server status and configuration, you can launch
/tmp/opendj/bin/status
```

Lets confirm it is working

```bash
/tmp/opendj/bin/ldapsearch -h localhost -p 1636 -D "cn=Directory Manager" -w Passw0rd -b "" -s base --useSsl --trustAll "(objectclass=*)" namingContexts \*+
```

should return

```bash
dn:
namingContexts: ou=tokens
namingContexts: ou=am-config
namingContexts: ou=identities
```

Lets stop this instance so that we can upgrade

```bash
/tmp/opendj/bin/stop-ds
```

should return

```bash
Stopping Server...
[13/Sep/2020:09:09:47 +1000] category=BACKEND severity=NOTICE msgID=370 msg=The backend cfgStore is now taken offline
[13/Sep/2020:09:09:47 +1000] category=BACKEND severity=NOTICE msgID=370 msg=The backend amIdentityStore is now taken offline
[13/Sep/2020:09:09:47 +1000] category=BACKEND severity=NOTICE msgID=370 msg=The backend amCts is now taken offline
[13/Sep/2020:09:09:47 +1000] category=CORE severity=NOTICE msgID=203 msg=The Directory Server is now stopped
```

### Deploy FRDS 7.0.0

*Note:* there is an issue where a Backstage Plugin has been defined so we need to remove it and its configuration.

```bash
export JAVA_HOME=/tmp/jdk-11.0.8+10-jre
cd /tmp
unzip -o DS-7.0.0.zip
rm -rf /tmp/opendj/lib/extensions/opendj-backstage-connect-plugin-6.5.1.jar
```

Edit `/tmp/opendj-instance/config/config.ldif` and remove the `dn: cn=Backstage Connect,cn=Plugins,cn=config` stanza.

```bash
dn: cn=Backstage Connect,cn=Plugins,cn=config
objectClass: top
objectClass: ds-cfg-plugin
cn: Backstage Connect
ds-cfg-java-class: org.forgerock.opendj.backstage.BackstageConnectPlugin
ds-cfg-enabled: true
ds-cfg-plugin-type: startup
ds-cfg-plugin-type: shutdown
```

Because this is an upghrade we need to tell FRDS that there is a new JDK and this can be done by editing `/tmp/opendj-instance/config/java.properties` using the following command

```bash
sed -i 's/default.java-home=\/tmp\/jdk8u265-b01-jre/default.java-home=\/tmp\/jdk-11.0.8+10-jre/g' /tmp/opendj-instance/config/java.properties
```

we can confirm this has worked through

```bash
cat /tmp/opendj-instance/config/java.properties | grep ^default.java-home
```

should return

```bash
default.java-home=/tmp/jdk-11.0.8+10-jre
```

We can now start the upgrade process

```bash
/tmp/opendj/upgrade --no-prompt --force
```

should return

```bash
>>>> OpenDJ Upgrade Utility

 * OpenDJ will be upgraded from version
 6.5.1.67f520472558c373d2615bccfe7871bad5b3efad to
 7.0.0.8f86179938248fdff37e3273e98405df7ae6f7a6
 * See '/tmp/opendj-instance/logs/upgrade.log' for a detailed log of this
 operation

>>>> Preparing to upgrade

  OpenDJ 7.0.0 changed the indexing algorithm for TelephoneNumber equality and
  substring matching rules. All TelephoneNumber syntax based attribute indexes
  must be rebuilt which may take a long time. Do you want to rebuild the
  indexes automatically at the end of the upgrade? (yes/no) [no]: yes

  The upgrade is ready to proceed. Do you wish to continue? (yes/no) [yes]:


>>>> Performing upgrade

  Removing configuration for replication monitoring...................   100%
  Setting new global server ID........................................   100%
  Removing global server ID from replication domains and replication
  server..............................................................   100%
  Set the proxy backend configuration property 'hash-function' to MD5.   100%
  Update replication SSL configuration................................   100%
  Adding Mail Servers.................................................   100%
  Renaming ds-cfg-connection-pool-min-size attributes to
  ds-cfg-bind-connection-pool-min-size................................   100%
  Renaming ds-cfg-connection-pool-max-size attributes to
  ds-cfg-bind-connection-pool-max-size................................   100%
  Renaming ds-cfg-connection-pool-idle-timeout attributes to
  ds-cfg-bind-connection-pool-idle-timeout............................   100%
  Set the database cache mode 'ds-cfg-db-cache-mode' to 'cache-ln'....   100%
  Add 'inheritFromDNParent' attribute type to the
  'inheritedCollectiveAttributeSubentry' object class.................   100%
Unable to retrieve the hostname from the admin backend using the truststore as
source of keys; 'advertised-listen-address' attribute in global configuration
will use the local hostname as value. Cause: Unable to look up server key id
'F2537E513D06D4C2375D3053745CF9EA' in the admin backend
  Adding 'listen-address' and 'advertised-listen-address' attributes
  to the global configuration.........................................   100%
  Removing 'listen-address' attributes that are redundant with
  default value provided by the global configuration..................   100%
  Migrating truststore backend configuration..........................   100%
  Add ACI to make Root DSE fullVendorVersion operational attribute
  user visible........................................................   100%
  Removing references to 'backup' backend.............................   100%
  Removing 'backup' backend...........................................   100%
  Migrating security settings.........................................   100%
  Removing profiler plugins...........................................   100%
  Removing max-work-queue-capacity....................................   100%
  Renaming the 'use-mutual-tls' configuration property to
  'use-sasl-external'.................................................   100%
  Renaming the 'replication-server' configuration property to
  'bootstrap-replication-server'......................................   100%
  Allowing dsbackup purge tasks.......................................   100%
  Replacing schema file '02-config.ldif'..............................   100%
  Removing 'ds-task-backup-all' attribute(s) from backup tasks........   100%
  Removing 'ds-backup-id', 'ds-task-backup-incremental',
  'ds-task-backup-incremental-base-id', 'ds-task-backup-compress',
  'ds-task-backup-hash', 'ds-task-backup-sign-hash' attribute(s) from
  backup tasks........................................................   100%
  Rename attribute 'ds-backup-directory-path' to 'ds-backup-location'
  in entries of objectClass 'ds-task-backup'..........................   100%
  Rename attribute 'ds-backup-directory-path' to 'ds-backup-location'
  in entries of objectClass 'ds-task-restore'.........................   100%
  Removing 'ds-task-backup-encrypt' attribute(s) from backup tasks....   100%
  Archiving concatenated schema.......................................   100%

>>>> OpenDJ was successfully upgraded to version
7.0.0.8f86179938248fdff37e3273e98405df7ae6f7a6


>>>> Performing post upgrade tasks

  Rebuilding index(es) '[.telephoneNumberMatch,
  .telephoneNumberSubstringsMatch]' for base dn(s) '[ou=am-config]'...   100%
  Rebuilding index(es) '[.telephoneNumberMatch,
  .telephoneNumberSubstringsMatch]' for base dn(s) '[ou=identities]'..   100%
  Rebuilding index(es) '[.telephoneNumberMatch,
  .telephoneNumberSubstringsMatch]' for base dn(s) '[ou=tokens]'......   100%

>>>> Post upgrade tasks complete

 * See '/tmp/opendj-instance/logs/upgrade.log' for a detailed log of this
 operation
````

Lets start the instance and confirm it is working

```bash
/tmp/opendj/bin/start-ds
/tmp/opendj/bin/ldapsearch -h localhost -p 1636 -D "cn=Directory Manager" -w Passw0rd -b "" -s base --useSsl --trustAll "(objectclass=*)" namingContexts \*+
```

should return 

```bash
[13/Sep/2020:09:42:49 +1000] category=CORE severity=NOTICE msgID=134 msg=ForgeRock Directory Services 7.0.0 (build 20200806092554, revision number 8f86179938248fdff37e3273e98405df7ae6f7a6) starting up
[13/Sep/2020:09:42:49 +1000] category=JVM severity=NOTICE msgID=21 msg=Installation Directory:  /tmp/opendj
[13/Sep/2020:09:42:49 +1000] category=JVM severity=NOTICE msgID=23 msg=Instance Directory:      /tmp/opendj-instance
[13/Sep/2020:09:42:49 +1000] category=JVM severity=NOTICE msgID=17 msg=JVM Information: 11.0.8+10 by AdoptOpenJDK, 64-bit architecture, 8399355904 bytes heap size
[13/Sep/2020:09:42:49 +1000] category=JVM severity=NOTICE msgID=18 msg=JVM Host: localhost default/Kameko_Mawst, running Linux 4.19.123-coreos amd64, 33597566976 bytes physical memory size, number of processors available 8
[13/Sep/2020:09:42:49 +1000] category=JVM severity=NOTICE msgID=19 msg=JVM Arguments: "-Xoptionsfile=/tmp/jdk-11.0.8+10-jre/lib/options.default", "-Xlockword:mode=default,noLockword=java/lang/String,noLockword=java/util/MapEntry,noLockword=java/util/HashMap$Entry,noLockword=org/apache/harmony/luni/util/ModifiedMap$Entry,noLockword=java/util/Hashtable$Entry,noLockword=java/lang/invoke/MethodType,noLockword=java/lang/invoke/MethodHandle,noLockword=java/lang/invoke/CollectHandle,noLockword=java/lang/invoke/ConstructorHandle,noLockword=java/lang/invoke/ConvertHandle,noLockword=java/lang/invoke/ArgumentConversionHandle,noLockword=java/lang/invoke/AsTypeHandle,noLockword=java/lang/invoke/ExplicitCastHandle,noLockword=java/lang/invoke/FilterReturnHandle,noLockword=java/lang/invoke/DirectHandle,noLockword=java/lang/invoke/ReceiverBoundHandle,noLockword=java/lang/invoke/DynamicInvokerHandle,noLockword=java/lang/invoke/FieldHandle,noLockword=java/lang/invoke/FieldGetterHandle,noLockword=java/lang/invoke/FieldSetterHandle,noLockword=java/lang/invoke/StaticFieldGetterHandle,noLockword=java/lang/invoke/StaticFieldSetterHandle,noLockword=java/lang/invoke/IndirectHandle,noLockword=java/lang/invoke/InterfaceHandle,noLockword=java/lang/invoke/VirtualHandle,noLockword=java/lang/invoke/PrimitiveHandle,noLockword=java/lang/invoke/InvokeExactHandle,noLockword=java/lang/invoke/InvokeGenericHandle,noLockword=java/lang/invoke/VarargsCollectorHandle,noLockword=java/lang/invoke/ThunkTuple", "-Xjcl:jclse29", "-Dcom.ibm.oti.vm.bootstrap.library.path=/tmp/jdk-11.0.8+10-jre/lib/compressedrefs:/tmp/jdk-11.0.8+10-jre/lib", "-Dsun.boot.library.path=/tmp/jdk-11.0.8+10-jre/lib/compressedrefs:/tmp/jdk-11.0.8+10-jre/lib", "-Djava.library.path=/tmp/jdk-11.0.8+10-jre/lib/compressedrefs:/tmp/jdk-11.0.8+10-jre/lib::/usr/lib64:/usr/lib", "-Djava.home=/tmp/jdk-11.0.8+10-jre", "-Duser.dir=/tmp", "-Djava.class.path=/tmp/opendj-instance/classes:/tmp/opendj/lib/bootstrap.jar", "-Dorg.opends.server.scriptName=start-ds", "-Dsun.java.command=org.opends.server.core.DirectoryServer --configFile /tmp/opendj-instance/config/config.ldif", "-Dsun.java.launcher=SUN_STANDARD", "-Dsun.java.launcher.pid=3163"
[13/Sep/2020:09:42:50 +1000] category=BACKEND severity=NOTICE msgID=513 msg=The database backend amCts containing 5 entries has started
[13/Sep/2020:09:42:51 +1000] category=BACKEND severity=NOTICE msgID=513 msg=The database backend cfgStore containing 3 entries has started
[13/Sep/2020:09:42:51 +1000] category=BACKEND severity=NOTICE msgID=513 msg=The database backend amIdentityStore containing 5 entries has started
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Subject Attribute to User Attribute,cn=Certificate Mappers,cn=config references attribute type cn which does not have an equality index defined in backend amCts
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Subject Attribute to User Attribute,cn=Certificate Mappers,cn=config references attribute type mail which does not have an equality index defined in backend amCts
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Subject DN to User Attribute,cn=Certificate Mappers,cn=config references attribute type ds-certificate-subject-dn which does not have an equality index defined in backend amCts
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Subject DN to User Attribute,cn=Certificate Mappers,cn=config references attribute type ds-certificate-subject-dn which does not have an equality index defined in backend cfgStore
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Subject DN to User Attribute,cn=Certificate Mappers,cn=config references attribute type ds-certificate-subject-dn which does not have an equality index defined in backend amIdentityStore
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Fingerprint Mapper,cn=Certificate Mappers,cn=config references attribute type ds-certificate-fingerprint which does not have an equality index defined in backend amCts
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Fingerprint Mapper,cn=Certificate Mappers,cn=config references attribute type ds-certificate-fingerprint which does not have an equality index defined in backend cfgStore
[13/Sep/2020:09:42:51 +1000] category=EXTENSIONS severity=WARNING msgID=568 msg=The subject attribute to user attribute certificate mapper defined in configuration entry cn=Fingerprint Mapper,cn=Certificate Mappers,cn=config references attribute type ds-certificate-fingerprint which does not have an equality index defined in backend amIdentityStore
[13/Sep/2020:09:42:51 +1000] category=PROTOCOL severity=NOTICE msgID=276 msg=Started listening for new connections on Administration Connector 0.0.0.0:4444
[13/Sep/2020:09:42:51 +1000] category=PROTOCOL severity=NOTICE msgID=276 msg=Started listening for new connections on LDAP 0.0.0.0:1389
[13/Sep/2020:09:42:51 +1000] category=PROTOCOL severity=NOTICE msgID=276 msg=Started listening for new connections on HTTPS 0.0.0.0:8443
[13/Sep/2020:09:42:51 +1000] category=PROTOCOL severity=NOTICE msgID=276 msg=Started listening for new connections on LDAPS 0.0.0.0:1636
[13/Sep/2020:09:42:51 +1000] category=CORE severity=NOTICE msgID=135 msg=The Directory Server has started successfully
[13/Sep/2020:09:42:51 +1000] category=CORE severity=NOTICE msgID=139 msg=The Directory Server has sent an alert notification generated by class org.opends.server.core.DirectoryServer (alert type org.opends.server.DirectoryServerStarted, alert ID org.opends.messages.core-135): The Directory Server has started successfully

dn:
namingContexts: ou=tokens
namingContexts: ou=am-config
namingContexts: ou=identities
```

*Note:* the followung warnings are unknown to me in regards to the issue and when I complete my investigation I will update this post with the necessary details to remove them.