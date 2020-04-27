# How do I setup 2 way SSL or mutual authentication between Apache and Wildfly using mod_proxy over https?


> **Ref/Courtesy**: Jboss documentation - https://access.redhat.com/solutions/82363. This documentation is heavily based of that but modified it for Wildfly 19 & Elytron SSL Context.

## Environment

- Redhat or Centos 7 or 8
- Wildfly 19
- Apache Httpd 2.4 and Mod_Proxy over Https
- OpenJdk 1.8.0_242 (TLS v1.2). 
  > If you need TLS v1.3 use JDK 11 or higher.


## Solution Steps

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
ProxyRequests Off
ProxyPreserveHost On
ProxyTimeout 600

SSLProxyEngine On
SSLProxyVerify On
SSLProxyProtocol all -SSLv2 -SSLv3 -TLSv1

# SSLProxyCACertificateFile - can be either the cert of the JBoss server (when using self-signed certs) 
# or the CA that signed the JBoss cert. 
# If you using actual CA signed cert you don't need to specify SSLProxyCACertificateFile.
SSLProxyCACertificateFile certs/jboss_cert.pem

# SSLProxyMachineCertificateFile - contains the public/private key pair (PEM formatted, concatenated). 
# This is what tells wildfly whether the request is coming a trusted apache. 
# Once again, don't have to specify this if you have an CA signed Cert. Only for Self Generated Certs.
SSLProxyMachineCertificateFile certs/apache_proxy.pem

ProxyPass / https://wildfly-localhost:8443/  keepalive=On
ProxyPassReverse / https://wildfly-localhost:8443/

```

# Script to Generate Self Signed Certs:

``` bash
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

PASSWORD="secret"
# Specify your domain name CN=example.com.
APACHE_CN="/C=US/ST=AR/L=Somewhere/CN=apache"

# Specify your domain name CN=example.com.
JBOSS_CN="CN=localhost"     

JBOSS_KEYSTORE="jboss.keystore"
JBOSS_CERT="jboss.cert"
JBOSS_KEY_ALIAS="server"
JBOSS_TRUSTSTORE="jboss.truststore"

echo "Creating public and private keys for Wildfly (Server-side)"

create_keystore $JBOSS_KEYSTORE $JBOSS_KEY_ALIAS $JBOSS_CN $PASSWORD

export_cert $JBOSS_KEYSTORE $JBOSS_KEY_ALIAS $JBOSS_CERT $PASSWORD

echo "Building public/private key to be used with Apache (Client-side)"

#openssl req -x509 -subj $APACHE_CN -nodes -days 365 -newkey rsa:1024 -keyout apache_key.pem -out apache_cert.pem

# Apache Private Key
openssl genrsa -out apache_key.pem 1024
# Apache Cert (Public)
openssl req -new -key apache_key.pem -x509 -subj $APACHE_CN -out apache_cert.pem -days 365
# Apache Combined
cat apache_key.pem apache_cert.pem > apache_proxy.pem

import_cert $JBOSS_TRUSTSTORE "apache" "apache_cert.pem" $PASSWORD

openssl x509 -in $JBOSS_CERT -inform DER -out jboss_cert.pem -outform PEM
```
**Result:**
```
-rw-r--r-- 1 user root 1253 Apr 15 00:44 apache_cert.pem
-rw------- 1 user root 1679 Apr 15 00:44 apache_key.pem
-rw-r--r-- 1 user root 2932 Apr 15 00:44 apache_proxy.pem
-rw-r--r-- 1 user root  717 Apr 15 00:44 jboss.cert
-rw-r--r-- 1 user root 1025 Apr 15 00:44 jboss_cert.pem
-rw-r--r-- 1 user root 2421 Apr 15 00:44 jboss.keystore
-rw-r--r-- 1 user root  948 Apr 15 00:44 jboss.truststore
```

Please feel free to create tickets if you have any issues.
