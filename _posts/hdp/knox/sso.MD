# Knox+Ambari+OpenLDAP集成单点登录

## Knox配置
knoxsso.xml:
```
      <topology>
          <gateway>
              <provider>
                  <role>webappsec</role>
                  <name>WebAppSec</name>
                  <enabled>true</enabled>
                  <param><name>xframe.options.enabled</name><value>true</value></param>
              </provider>

              <provider>
                  <role>authentication</role>
                  <name>ShiroProvider</name>
                  <enabled>true</enabled>
                  <param>
                      <name>sessionTimeout</name>
                      <value>30</value>
                  </param>
                  <param>
                      <name>redirectToUrl</name>
                      <value>/gateway/knoxsso/knoxauth/login.html</value>
                  </param>
                  <param>
                      <name>restrictedCookies</name>
                      <value>rememberme,WWW-Authenticate</value>
                  </param>
                  <param>
                      <name>main.ldapRealm</name>
                      <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm</value>
                  </param>
                  <param>
                      <name>main.ldapContextFactory</name>
                      <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory</value>
                  </param>
                  <param>
                      <name>main.ldapRealm.contextFactory</name>
                      <value>$ldapContextFactory</value>
                  </param>
                  <param>
                      <name>main.ldapRealm.userDnTemplate</name>
                      <value>uid={0},ou=people,dc=ambari,dc=apache,dc=org</value>
                  </param>
                  <param>
                      <name>main.ldapRealm.contextFactory.url</name>
                      <value>ldap://u1401.ambari.apache.org</value>
                  </param>    
                  <param>
                      <name>main.ldapRealm.authenticationCachingEnabled</name>
                      <value>false</value>
                  </param>
                  <param>
                      <name>main.ldapRealm.contextFactory.authenticationMechanism</name>
                      <value>simple</value>
                  </param>
                  <param>
                      <name>urls./**</name>
                      <value>authcBasic</value>
                  </param>
              </provider>

              <provider>
                  <role>identity-assertion</role>
                  <name>Default</name>
                  <enabled>true</enabled>
              </provider>
          </gateway>

          <application>
            <name>knoxauth</name>
          </application>

          <service>
              <role>KNOXSSO</role>
              <param>
                  <name>knoxsso.cookie.secure.only</name>
                  <value>false</value>
              </param>
              <param>
                  <name>knoxsso.token.ttl</name>
                  <value>30000</value>
              </param>
              <param>
                 <name>knoxsso.redirect.whitelist.regex</name>
                 <value>^https?:\/\/(u14\d\d\.ambari\.apache\.org|localhost|127\.0\.0\.1|0:0:0:0:0:0:0:1|::1):[0-9].*$</value>
              </param>
          </service>

      </topology>
```
说明：<br>
  main.ldapRealm.userDnTemplate: 配置ldap用户模板
  main.ldapRealm.contextFactory.url： ldap service url
  main.ldapRealm.contextFactory.authenticationMechanism：认证模式，默认simple
  knoxsso.redirect.whitelist.regex：knoxsso重定向url白名单（正则表达式），必须包含ambari server和knox server

## Ambari配置
ssh登录ambari server
### ambari ldap设置
```
$ ambari-server setup-ldap  ##执行ambari ldap设置，依次按照步骤输入：
Using python  /usr/bin/python
Setting up LDAP properties...
Primary URL* {host:port} : u1401.ambari.apache.org:389 # ldap server host+端口
Secondary URL {host:port} : u1401.ambari.apache.org:389
Use SSL* [true/false] : false  #是否启用ssl
User object class* : person  #ldap 用户对象类，固定person
User name attribute* : uid  #ldap 用户名称对应属性，固定uid
Group object class* : group  #ldap 用户组类，固定group
Group name attribute* :member   #ldap 用户组名称对应属性，固定member
Group member attribute* : distinguishedName  #ldap 用户组成员属性，固定distinguishedName
Distinguished name attribute* : dn  #ldap 名称属性，固定dn 
Base DN* : dc=ambari,dc=apache,dc=org   #ldap 基础dn
Referral method [follow/ignore] (ignore): 
Bind anonymously* [true/false] (false): 
Manager DN* : cn=admin,dc=ambari,dc=apache,dc=org   #ldap 管理员dn
Enter Manager Password* : vagrant    #ldap 管理员密码
Re-enter password:  vagrant
====================
Review Settings
====================
authentication.ldap.managerDn: cn=admin,dc=ambari,dc=apache,dc=org
authentication.ldap.managerPassword: *****
Save settings [y/n] (y)? y
Saving...done
Ambari Server 'setup-ldap' completed successfully.
```
ambari ldap设置成功后，需要同步ldap用户：
```
$ ambari-server sync-ldap --all  #同步ldap用户，并输入ambari admin用户以及密码：
Using python  /usr/bin/python
Syncing with LDAP...
Enter Ambari Admin login: admin
Enter Ambari Admin password: admin
Syncing all...

Completed LDAP Sync.
Summary:
  memberships:
    removed = 0
    created = 0
  users:
    updated = 0
    removed = 0
    created = 1
  groups:
    updated = 0
    removed = 0
    created = 0

Ambari Server 'sync-ldap' completed successfully.

```
同步成功后，ambari ldap设置完成
### 生成knox认证证书
```
$  keytool -exportcert -keystore data/security/keystores/gateway.jks -alias gateway-identity -rfc -file gateway.pem
```
### ambari sso设置
参考[https://cwiki.apache.org//confluence/display/KNOX/Ambari+via+KnoxSSO+and+Default+IDP](https://cwiki.apache.org//confluence/display/KNOX/Ambari+via+KnoxSSO+and+Default+IDP)
```
$ ambari-server setup-sso
Using python  /usr/bin/python
Setting up SSO authentication properties...
Do you want to disable SSO authentication [y/n] (n)?y
Ambari Server 'setup-sso' completed successfully.
root@u1401:~# ambari-server setup-sso
Using python  /usr/bin/python
Setting up SSO authentication properties...
Do you want to configure SSO authentication [y/n] (y)?y
Provider URL [URL] (https://u1403.ambari.apache.org:8443/gateway/knoxsso/api/v1/websso):
Public Certificate pem (stored) (empty line to finish input):
MIICVTCCAb6gAwIBAgIIMwanHwzNXrMwDQYJKoZIhvcNAQEFBQAwbTELMAkGA1UE
BhMCVVMxDTALBgNVBAgTBFRlc3QxDTALBgNVBAcTBFRlc3QxDzANBgNVBAoTBkhh
ZG9vcDENMAsGA1UECxMEVGVzdDEgMB4GA1UEAxMXdTE0MDMuYW1iYXJpLmFwYWNo
ZS5vcmcwHhcNMTcwNjA1MDIzODA3WhcNMTgwNjA1MDIzODA3WjBtMQswCQYDVQQG
EwJVUzENMAsGA1UECBMEVGVzdDENMAsGA1UEBxMEVGVzdDEPMA0GA1UEChMGSGFk
b29wMQ0wCwYDVQQLEwRUZXN0MSAwHgYDVQQDExd1MTQwMy5hbWJhcmkuYXBhY2hl
Lm9yZzCBnzANBgkqhkiG9w0BAQEFAAOBjQAwgYkCgYEAi0ZYloU8XirUrHIItkoq
PzYAFvvlvdyrPLyca3VTvYoLwST7JkWo1RvFGwZg1E4+rddA1yG/l66JUfL0wYGo
oNE4rjZo8K8V5rzQMDhoquuOdaVVO3V4r4KAabtt2pICLshsbgulu83IYsEP+BBW
ApVPI2zcWZBmlMPJb4feSNECAwEAATANBgkqhkiG9w0BAQUFAAOBgQCFzPPfBcf/
dqEOuFmY+KaQMgIUd+DoBOl5xzdCy1r/bAGcADRJLBiy8k8EG17ljBOTkO9NC8Ib
kMFRHmTiX6+eD2do5Nfb7NoNS9arG28qQdCUhYIYEE5AeRkfD9mo2BU/VRf12/wj
qg2IeMRelfw+tcoqA6COH4T/CsyvmMnF8A==

Do you want to configure advanced properties [y/n] (n) ?n
Ambari Server 'setup-sso' completed successfully.
```
ambari sso设置成功，重启ambari server
```
$ ambari-server restart
```
