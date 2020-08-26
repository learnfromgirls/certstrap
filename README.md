# certstrap
[![godoc](http://img.shields.io/badge/godoc-certstrap-blue.svg?style=flat)](https://godoc.org/github.com/square/certstrap)
[![build](https://img.shields.io/travis/square/certstrap.svg?style=flat)](https://travis-ci.org/square/certstrap) [![license](http://img.shields.io/badge/license-apache_2.0-red.svg?style=flat)](https://raw.githubusercontent.com/square/certstrap/master/LICENSE)

A simple certificate manager written in Go, to bootstrap your own certificate authority and public key infrastructure.  Adapted from square/certstrap , etcd-ca.

certstrap is a very convenient app if you don't feel like dealing with openssl, its myriad of options or config files. In particular a --codesigning boolean option has been added to the certificate "sign" command so that the javaws jar files can be accepted as valid codesigned jars later. 

## Common Uses

certstrap allows you to build your own certificate system:

1. Initialize certificate authorities
2. Create identities and certificate signature requests for hosts
3. Sign and generate certificates (including the codesigner extra key usage flag)

## Certificate architecture

certstrap can init multiple certificate authorities to sign certificates with.  Users can make arbitrarily long certificate chains by using signed hosts to sign later certificate requests, as well.

## Examples

## Getting Started

### Building

certstrap must be built with Go 1.13+. You can build certstrap from source:

```
$ git clone https://github.com/learnfromgirls/certstrap
$ cd certstrap
$ go build
```

This will generate a binary called `certstrap` under project root folder.

### Initialize a new certificate authority:

```
$ ./certstrap init --common-name "CertAuthTurboAlgorithms"
Created out/CertAuthTurboAlgorithms.key
Created out/CertAuthTurboAlgorithms.crt
Created out/CertAuthTurboAlgorithms.crl
```

Note that the `-common-name` flag is required, and will be used to name output files.

Moreover, this will also generate a new keypair for the Certificate Authority,
though you can use a pre-existing private PEM key with the `-key` flag.

If the CN contains spaces, certstrap will change them to underscores in the filename for easier use.  The spaces will be preserved inside the fields of the generated files:

```
$ ./certstrap init --common-name "Cert Auth"
Created out/Cert_Auth.key
Created out/Cert_Auth.crt
Created out/Cert_Auth.crl
```

### Request a certificate, including keypair:

```
$ ./certstrap request-cert --domain www.turboalgorithms.com,turboalgorithms.com
Created out/www.turboalgorithms.com.key (encrypted by passphrase)
Created out/www.turboalgorithms.com.csr
```

certstrap requires either `-common-name` or `-domain` flag to be set in order to generate a certificate signing request.  The CN for the certificate will be found from these fields.

If your server has mutiple ip addresses or domains, use comma separated ip/domain/uri list. eg: `./certstrap request-cert -ip $ip1,$ip2 -domain $domain1,$domain2 -uri $uri1,$uri2`

If you do not wish to generate a new keypair, you can use a pre-existing private
PEM key with the `-key` flag

### Sign certificate request of host and generate the certificate:

```
$ ./certstrap sign www.turboalgorithms.com --CA CertAuthTurboAlgorithms --codesigning
Including extkeyusage codesigning
Created out/www.turboalgorithms.com.crt from out/www.turboalgorithms.com.csr signed by out/CertAuthTurboAlgorithms.key
```

### Retrieving Files

Outputted key, request, and certificate files can be found in the depot directory.
By default, this is in `out/`

### printing the certificate:

```
$ cd out
$ openssl x509 -in www.turboalgorithms.com.crt -text
...
 X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication, Code Signing
...
```
### Use the java keytool to import the CA certificate into the java client keystore:
```
$ keytool -import -keystore ~/.keystore -file CertAuthTurboAlgorithms.crt -alias CertAuthTurboAlgorithms

...
Trust this certificate? [no]:  yes
Certificate was added to keystore

```

#### PKCS Format (needed as keytool can only import keys using pkcs12) :
Convert your certificate and key to PKCS12 format, simply run:
```
$ openssl pkcs12 -export -out outputCert.p12 -inkey www.turboalgorithms.com.key -in www.turboalgorithms.com.crt -certfile CertAuthTurboAlgorithms.crt -name www.turboalgorithms.com


```
Note that passwords will generally be keystore passwords to protect the whole file. Do not confuse with key passwords which are only used when actually using the key for signing something.

`inputKey.key` and `inputCert.crt` make up the leaf private key and certificate pair of your choosing (generated by a `sign` command), with `CA.crt` being the certificate authority certificate that was used to sign it.  The output PKCS12 file is `outputCert.p12`


### Use the java keytool to import the signed certificate for the associated client alias in the keystore:
```
$ keytool -importkeystore -srckeystore outputCert.p12 -destkeystore ~/.keystore -srcstoretype pkcs12 -alias www.turboalgorithms.com
 
```

### Use the java keytool to check you have a full certificate chain with correct alias.
```
$ keytool -list -keystore ~/.keystore -alias www.turboalgorithms.com -v
...
Entry type: PrivateKeyEntry
Certificate chain length: 2
Certificate[1]:
Owner: CN=www.turboalgorithms.com
Issuer: CN=CertAuthTurboAlgorithms
...
Certificate[2]:
Owner: CN=CertAuthTurboAlgorithms
Issuer: CN=CertAuthTurboAlgorithms
...



```

### Using keytool to sign your jar files with the new codesigning certificate chain and key:

Here are some ant tasks which I use.
```
    <target name="keystorepassword">

    <input message="keystore password? " addproperty="keystorepass" /> 
    <input message="key password? " addproperty="keypass" /> 
</target>

     <target name="-post-jar" depends="keystorepassword,docroot">
       <signjar jar="${dist.jar}"  alias="www.turboalgorithms.com" keypass="${keypass}" storepass="${keystorepass}" storetype="pkcs12" />
 <mkdir dir="${docroot.dir}"/>
    <mkdir dir="${docroot.dir}/lib"/>
	<copy tofile="${docroot.dir}/lib/login2.jar" file="${dist.jar}" overwrite="true" />
     </target>
      <target name="myx509Signed" depends="keystorepassword">
    <mkdir dir="${docroot.dir}"/>
    <mkdir dir="${docroot.dir}/lib"/>
	<copy tofile="${docroot.dir}/lib/myx509Signed.jar" file="${srcbin.dir}/myx509.jar" overwrite="true" />
	<copy tofile="${docroot.dir}/lib/log4jSigned.jar" file="${srcbin.dir}/log4j.jar" overwrite="true" />
	<copy tofile="${docroot.dir}/lib/vecmathSigned.jar" file="${srcbin.dir}/vecmath.jar" overwrite="true" />
	
    <signjar jar="${docroot.dir}/lib/myx509Signed.jar" alias="www.turboalgorithms.com" storetype="pkcs12" keypass="${keypass}" storepass="${keystorepass}" />
    <signjar jar="${docroot.dir}/lib/log4jSigned.jar" alias="www.turboalgorithms.com" keypass="${keypass}" storetype="pkcs12"  storepass="${keystorepass}" />
    <signjar jar="${docroot.dir}/lib/vecmathSigned.jar" alias="www.turboalgorithms.com" keypass="${keypass}" storetype="pkcs12"  storepass="${keystorepass}" />
    </target>
   <target name="docroot" depends="myx509Signed" description="copies files to the docroot">
     <mkdir dir="${docroot.dir}"/>

     <copy todir="${docroot.dir}">
       <fileset dir="${srcdoc.dir}" >
	<include name="**/*.jnlp" />
	<include name="**/*.gif" />
	<include name="**/*.jpg" />
	<include name="**/*.cer" />
	<include name="**/*.html" />
	<include name="**/*.java" />
	<include name="**/*.class" />
	</fileset>
     </copy>
  </target>
  
```
### Upload you signed jar files to your webserver together with a suitable .jnlp file

Here is the jnlp file which I use:-
```
<?xml version="1.0" encoding="utf-8"?> 
     <jnlp 
       spec="1.0+" 
       codebase="https://www.turboalgorithms.com" 
       href="https://www.turboalgorithms.com/vertical.jnlp"> 
       <information> 
         <title>Vertical</title> 
         <vendor>Cycom Limited</vendor> 
         <homepage href="docs/help.html"/> 
         <description>Vertical</description> 
         <description kind="short">Vertical</description> 
         <icon href="images/login.jpg"/> 
         <offline-allowed/> 
       </information> 
       <security> 
           <all-permissions/> 
       </security> 
       <resources> 
         <j2se version="1.4+"/> 
         <jar href="lib/login2.jar"/> 
         <jar href="lib/myx509Signed.jar"/> 
         <jar href="lib/log4jSigned.jar"/> 
         <jar href="lib/vecmathSigned.jar"/> 
       </resources> 
       <application-desc main-class="com.cyterm.login.Main"/> 
     </jnlp> 

```
### Run your signed javaws application from the command line as:-


```
$ javaws https://www.turboalgorithms.com/vertical.jnlp

```

If there is no javaws command even available, you will need to install java from java.com first.

The jawaws command will 
most likely fail initially as the self signed certificates are not trusted and there is no infrastructure for timestamps and certificate revocations.

You can bypass this by adding the website to a list of trusted sites (on each client machine).

```
$ javaws -viewer

```

This brings up 2 dialogs. You can close the "Java Cache Viewer"  dialog leaving only the "Java Control Panel" dialog. Select the "Security" tab , ensure that the security level radio option is "High" (and not "Very High") and use the "Edit Site List" button which brings up the "Exception Site List" dialog. On this new dialog use the "Add" button and click into the new line in the site list to add your site to the  exclusion list
e.g. typing in  
 
```
https://wwww.turboalgorithms.com/
```

Then finish the entry with a tab key and click "OK" to close the "Exception Site List" dialog , then "OK" to close the "Java Control Panel" dialog.

Now you can retry the original command and it will now be allowed after you determinately tell it that you trust this program and wish to proceed with execution. (javaws will be trying to avoid execution)  you from  Note that the signer will be still shown as "UNKNOWN" and that is again due to fact that javaws does not have full trust in the self signed certificate authority and its associated timestamp and revocation list infrastructure and so has chosen not to display the arbitrary name information on the certificates.

```
$ javaws https://www.turboalgorithms.com/vertical.jnlp

```

Note that javaws will remember and cache your responses so that the subsequent invocations will be briefer.


## Project Details

### Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for details on submitting patches.

### License

certstrap is under the Apache 2.0 license. See the [LICENSE](LICENSE) file for details.
