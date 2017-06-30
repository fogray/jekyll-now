---
title: OpenLDAP
layout: post
theme: jekyll-theme-cayman
---
# OpenLDAP+CentOS7.0安装部署

## 安装LDAP Server
```
$ yum install -y openldap openldap-clients openldap-servers migrationtools 
```
## 配置
```
$ yum install -y vim
$ vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{2\}hdb.ldif
dn: olcDatabase={2}hdb
objectClass: olcDatabaseConfig
objectClass: olcHdbConfig
olcDatabase: {2}hdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=ambari,dc=apache,dc=org  #修改自己定义的域名dc
olcRootDN: cn=admin,dc=ambari,dc=apache,dc=org  #修改管理员用户
olcRootPW: 123456a?   #添加这一行，设置管理员密码
olcDbIndex: objectClass eq,pres
olcDbIndex: ou,cn,mail,surname,givenname eq,pres,sub
structuralObjectClass: olcHdbConfig
entryUUID: ab074662-eff6-1036-88da-9b2c3607bbae
creatorsName: cn=config
createTimestamp: 20170628023852Z
entryCSN: 20170628023852.774934Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20170628023852Z
```
主要修改三处：olcSuffix、 olcRootDN、olcRootPW<br>
修改Monitor配置：
```
$ vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{1\}monitor.ldif
dn: olcDatabase={1}monitor
objectClass: olcDatabaseConfig
olcDatabase: {1}monitor
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
 al,cn=auth" read by dn.base="cn=admin,dc=ambari,dc=apache,dc=org" read by * none #修改管理员base dn
structuralObjectClass: olcDatabaseConfig
entryUUID: ab0739e2-eff6-1036-88d9-9b2c3607bbae
creatorsName: cn=config
createTimestamp: 20170628023852Z
entryCSN: 20170628023852.774616Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20170628023852Z
```
## 配置LDAP数据库
```
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
$ chown -R ldap.ldap /var/lib/ldap
```
## 测试配置是否正确
```
$ slaptest -u 
59536060 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
59536060 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded
```
## 添加默认配置到LDAP
将所有的配置LDAP server, 添加到LDAP schemas中
```
$ cd /etc/openldap/schema/
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f cosine.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f nis.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f collective.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f corba.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f core.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f duaconf.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f dyngroup.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f inetorgperson.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f java.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f misc.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f openldap.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f pmi.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f ppolicy.ldif
```
## 创建LDAP DIT
使用 Migration Tools to 创建LDAP DIT，cd /usr/share/migrationtools/进行更改vim migrate_common.ph
```
$ cd /usr/share/migrationtools/
$ vim migrate_common.ph
...
$NAMINGCONTEXT{'group'} = "ou=Groups"          #61行, 改为"ou=Groups"
...
$DEFAULT_MAIL_DOMAIN = "ambari.apache.org"     #71行，改为实际域名
...
$DEFAULT_BASE = "dc=ambari,dc=apache,dc=org";  #74行, 改为实际基本dn
...
$EXTENDED_SCHEMA = 1;                          #90行, 改为1
...
```
 生成LDIF文件，并添加进ldap数据库
 ```
 $ cd /usr/share/migrationtools/ 
 $ ./migrate_base.pl> /root/base.ldif
 $ ldapadd -x -W -D "cn=admin,dc=ambari,dc=apache,dc=org" -f /root/base.ldif
 Enter LDAP Password: 
adding new entry "dc=ambari,dc=apache,dc=org"
adding new entry "ou=Hosts,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Rpc,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Services,dc=ambari,dc=apache,dc=org"
adding new entry "nisMapName=netgroup.byuser,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Mounts,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Networks,dc=ambari,dc=apache,dc=org"
adding new entry "ou=People,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Group,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Netgroup,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Protocols,dc=ambari,dc=apache,dc=org"
adding new entry "ou=Aliases,dc=ambari,dc=apache,dc=org"
adding new entry "nisMapName=netgroup.byhost,dc=ambari,dc=apache,dc=org"
 ```
