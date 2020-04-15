
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
<subsystem xmlns="urn:wildfly:elytron:8.0" final-providers="combined-providers" disallowed-providers="OracleUcrypto">
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
</subsystem>
 
. . .

<subsystem xmlns="urn:jboss:domain:undertow:10.0" default-server="default-server" default-virtual-host="default-host" default-servlet-container="default" default-security-domain="other" statistics-enabled="${wildfly.under
tow.statistics-enabled:${wildfly.statistics-enabled:false}}">

. . .

 <https-listener name="https" socket-binding="https" ssl-context="MySSLContext" enable-http2="true"/>

. . .
</subsystem>

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

# Script Generate Self Signed Certs:

```
#!/bin/sh

function create_keystore
{
  KEY_FILE=$1
  ALIAS=$2
  DN=$3
  PASS=$4
  keytool -genkey -alias $ALIAS -keyalg RSA -keystore $KEY_FILE -validity 365 -storetype pkcs12 -storepass $PASS -keypass $PASS -dname $DN
}

function export_cert
{
  KEY_FILE=$1
  ALIAS=$2
  EXPORT_FILE=$3
  PASS=$4
  keytool -export -alias $ALIAS -keystore $KEY_FILE -storepass $PASS -file $EXPORT_FILE
}

function import_cert
{
  KEY_FILE=$1
  ALIAS=$2
  IMPORT_FILE=$3
  PASS=$4
  keytool -import -noprompt -alias $ALIAS -keystore $KEY_FILE -storepass $PASS -file $IMPORT_FILE
}

PASSWORD="123456"
APACHE_CN="/C=US/ST=AR/L=Somewhere/CN=apache"
JBOSS_CN="CN=localhost"
JBOSS_KEYSTORE="jboss.keystore"
JBOSS_CERT="jboss.cert"
JBOSS_KEY_ALIAS="server"
JBOSS_TRUSTSTORE="jboss.truststore"

create_keystore $JBOSS_KEYSTORE $JBOSS_KEY_ALIAS $JBOSS_CN $PASSWORD

export_cert $JBOSS_KEYSTORE $JBOSS_KEY_ALIAS $JBOSS_CERT $PASSWORD

echo "Add the following to server.xml"
echo "  keystoreFile=\"\${jboss.server.home.dir}/conf/$JBOSS_KEYSTORE\"
  keystorePass=\"$PASSWORD\"
  truststoreFile=\"\${jboss.server.home.dir}/conf/$JBOSS_TRUSTSTORE\"
  truststorePass=\"$PASSWORD\"
  clientAuth=\"true\""

echo "Building public/private key to be used with Apache"
#openssl req -x509 -subj $APACHE_CN -nodes -days 365 -newkey rsa:1024 -keyout apache_key.pem -out apache_cert.pem
openssl genrsa -out apache_key.pem 1024
openssl req -new -key apache_key.pem -x509 -subj $APACHE_CN -out apache_cert.pem -days 365
cat apache_key.pem apache_cert.pem > apache_proxy.pem

import_cert $JBOSS_TRUSTSTORE "apache" "apache_cert.pem" $PASSWORD

openssl x509 -in $JBOSS_CERT -inform DER -out jboss_cert.pem -outform PEM
```

