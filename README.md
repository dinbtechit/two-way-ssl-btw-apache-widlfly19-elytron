
> **Ref/Courtesy**: Jboss documentation - https://access.redhat.com/solutions/82363. This documentation heavily based on of that but modified it for Wildfly 19 & Elytron SSL Context.


- Redhat or Centos 7 or 8
- Wildfly 19
- Apache Httpd 2.4 and Mod_Proxy over Https
- OpenJdk 1.8.0_242



Here is my Configuration:
-------------------------

**Wildfly 19** 

**CLI Commands**
``` bash
/subsystem=elytron/key-store=MyKeyStore:add(path=/etc/certs/selfSigned/jboss.keystore, credential-reference={clear-text=secret},type=JKS)

/subsystem=elytron/key-manager=MyKeyManager:add(key-store=MyKeyStore,credential-reference={clear-text=secret}})

/subsystem=elytron/key-store=MyKeyTrustStore:add(path=/etc/certs/selfSigned/jboss.truststore, credential-reference={clear-text=secret},type=JKS)

/subsystem=elytron/trust-manager=MyTrustManager:add(key-store=MyKeyTrustStore)

/subsystem=elytron/server-ssl-context=MySSLContext:add(key-manager=MyKeyManager,protocols=["TLSv1.2"],trust-manager=MyTrustManager,need-client-auth=true)

/subsystem=undertow/server=default-server/https-listener=https:add(socket-binding=https, ssl-context=MySSLContext, enable-http2=true)

:reload

```

**Results:**
 
**Standalone-full.xml**
``` xml
. . .
<tls>
    <key-stores>
        <key-store name="MyKeyStore">
            <credential-reference clear-text="secret"/>
            <implementation type="JKS"/>
            <file path="/etc/certs/selfSigned/jboss.keystore"/>
        </key-store>
        <key-store name="MyKeyTrustStore">
            <credential-reference clear-text="secret"/>
            <implementation type="JKS"/>
            <file path="/etc/certs/selfSigned/jboss.truststore"/>
        </key-store>
    </key-stores>
    <key-managers>
        <key-manager name="MyKeyManager" key-store="MyKeyStore">
            <credential-reference clear-text="secret"/>
        </key-manager>
    </key-managers>
    <trust-managers>
        <trust-manager name="MyTrustManager" key-store="MyKeyTrustStore"/>
    </trust-managers>
    <server-ssl-contexts>
        <server-ssl-context name="MySSLContext" protocols="TLSv1.2" need-client-auth="true" key-manager="MyKeyManager" trust-manager="MyTrustManager"/>
    </server-ssl-contexts>
</tls>
. . .

<subsystem xmlns="urn:jboss:domain:undertow:10.0" default-server="default-server" default-virtual-host="default-host" default-servlet-container="default" default-security-domain="other" statistics-enabled="${wildfly.under
tow.statistics-enabled:${wildfly.statistics-enabled:false}}">

. . .

 <https-listener name="https" socket-binding="https" ssl-context="MySSLContext" enable-http2="true"/>

. . .

```

**Apache Configuration:**

``` apacheconf
SSLProxyEngine On
SSLProxyVerify On
SSLProxyCACertificateFile certs/jboss_cert.pem
SSLProxyMachineCertificateFile certs/apache_proxy.pem
ProxyRequests Off
ProxyPass / https://wildfly-localhost:8443/
ProxyPassReverse / https://wildfly-localhost:8443/
```
