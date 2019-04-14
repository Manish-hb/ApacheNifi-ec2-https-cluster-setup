# ApacheNifi-ec2-https-cluster-setup
Setting up a Secure Apache NIFI-1.6.0 cluster on multiple ec2 instances,
by creating self signed certificates using keytool and Ldap configuration for user Authentication and Authorisation.

Apache NiFi:
Is designed to automate the flow of data between software systems.
It is based on the "NiagaraFiles" software previously developed by the NSA, 
which is also the source of a part of its present name – NiFi. 
It was open-sourced as a part of NSA's technology transfer program in 2014.
The software design is based on the flow-based programming model and offers features,
which prominently include the ability to operate within clusters, security using TLS encryption,
extensibility (users can write their own software to extend its abilities) 
and improved usability features like a portal which can be used to view and modify behaviour visually.




Nifi Cluster Setup with https certificates:

#login to amazon ec2 instance through terminal.

#Shift as root

$ sudo su

#update linux box

$ yum update

# install java 8 and remove java 7 :-

$ yum install java-1.8.0-openjdk-devel 

$ yum remove java-1.7.0-openjdk 

# go to /opt, download Nifi and unzip the zipped file:-

$ cd /opt

$ wget https://archive.apache.org/dist/nifi/1.6.0/nifi-1.6.0-bin.tar.gz

$ tar zxf nifi-1.6.0-bin.tar.gz

#following are the commands to create required certificates using keytool:-
$ cd /etc/security

$ mkdir sslstore

$ cd sslstore

$ keytool -genkey -alias <hostname>  -keyalg RSA -keysize 1024 -dname "CN=<hostname>, OU=NIFI" -keypass keytool123 -keystore keystore.jks -storepass keytool123

$ keytool -export  -alias <hostname>  -keystore keystore.jks  -rfc -file cert.file  -storepass keytool123

$ keytool -import -noprompt -alias  <hostname>  -keystore truststore.jks  -file cert.file -storepass keytool123

$ keytool -import -noprompt -alias  <hostname>  -keystore truststore_all.jks  -file cert.file -storepass keytool123

$ mkdir server

$ mkdir client

$ cp keystore.jks server/

$ cp truststore.jks server/

$ cp truststore_all.jks client/

$ cp cert.file server/

$ cd server

$ chmod 440 *

#Execute above block on next host.
#But before you run command for "truststore_all.jks" make sure you have copied it to "sslstore" directory from previous host. 
#After you are done with last host, make sure this file "truststore_all.jks" is copied to all the hosts in client directory.

#move to Nifi conf folder and change properties:-

$ cd /opt/nifi-1.6.0/conf

$ vi nifi.properties

#following are the changes to be made in nifi.properties:-

nifi.state.management.embedded.zookeeper.start=true

nifi.remote.input.host=<hostname>
nifi.remote.input.secure=true
nifi.remote.input.socket.port=9093
nifi.remote.input.http.enabled=false

nifi.web.http.host=
nifi.web.http.port=
nifi.web.https.host=<hostname>
nifi.web.https.port=9443

nifi.security.keystore=/etc/security/sslstore/server/keystore.jks
nifi.security.keystoreType=jks
nifi.security.keystorePasswd=keytool123
nifi.security.keyPasswd=keytool123
nifi.security.truststore=/etc/security/sslstore/client/truststore_all.jks
nifi.security.truststoreType=jks
nifi.security.truststorePasswd=keytool123
nifi.security.needClientAuth=true
nifi.security.user.authorizer=managed-authorizer
nifi.security.user.login.identity.provider=ldap-provider

nifi.cluster.protocol.is.secure=true

nifi.cluster.is.node=true
nifi.cluster.node.address=hostname
nifi.cluster.node.protocol.port=9998
nifi.zookeeper.connect.string=hostname1:2181,hostname2:2181,hostname3:2181,etc
nifi.zookeeper.connect.timeout=15 secs
nifi.zookeeper.session.timeout=15 secs


#following should be only data in authorizers.xml rest should be commented:-

<userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">./conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>

        <property name="Initial User Identity 1">admin</property>
        <property name="Initial User Identity 2">CN=<hostname1>, OU=NIFI</property>
        <property name="Initial User Identity 3">CN=<hostname2>, OU=NIFI</property>
        <property name="Initial User Identity 4">CN=<hostname3>, OU=NIFI</property>
</userGroupProvider>

<accessPolicyProvider>
        <identifier>file-access-policy-provider</identifier>
        <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
        <property name="User Group Provider">file-user-group-provider</property>
        <property name="Authorizations File">./conf/authorizations.xml</property>
        <property name="Initial Admin Identity">admin</property>
        <property name="Legacy Authorized Users File"></property>

	<property name="Node Identity 1">CN=<hostname1>, OU=NIFI</property>
        <property name="Node Identity 2">CN=<hostname2>, OU=NIFI</property>
        <property name="Node Identity 3">CN=<hostname3>, OU=NIFI</property>
</accessPolicyProvider>

<authorizer>
        <identifier>managed-authorizer</identifier>
        <class>org.apache.nifi.authorization.StandardManagedAuthorizer</class>
        <property name="Access Policy Provider">file-access-policy-provider</property>
</authorizer>

#following should be only data in login-identity-providers.xml rest should be commented:-
<provider>
        <identifier>ldap-provider</identifier>
        <class>org.apache.nifi.ldap.LdapProvider</class>
        <property name="Authentication Strategy">SIMPLE</property>

        <property name="Manager DN">cn=admin,dc=example,dc=org</property>
        <property name="Manager Password">admin</property>

        <property name="TLS - Keystore"></property>
        <property name="TLS - Keystore Password"></property>
        <property name="TLS - Keystore Type"></property>
        <property name="TLS - Truststore"></property>
        <property name="TLS - Truststore Password"></property>
        <property name="TLS - Truststore Type"></property>
        <property name="TLS - Client Auth"></property>
        <property name="TLS - Protocol"></property>
        <property name="TLS - Shutdown Gracefully"></property>

        <property name="Referral Strategy">FOLLOW</property>
        <property name="Connect Timeout">10 secs</property>
        <property name="Read Timeout">10 secs</property>
        <property name="Identity Strategy">USE_USERNAME</property>
        <property name="Url">ldap://<hostnameOfLdapServer>:389</property>
        <property name="User Search Base">dc=example,dc=org</property>
        <property name="User Search Filter">cn={0}</property>

        <property name="Authentication Expiration">12 hours</property>
    </provider>

#Add this zookeeper.properties:-
server.1=<hostname1>:2888:3888
server.2=<hostname2>:2888:3888
server.3=<hostname3>:2888:3888

#run following commands for zookeeperin all nodes inside nifi-1.6.0/:-
$ mkdir ./state
$ mkdir ./state/zookeeper

# in first node of cluster inside nifi-1.6.0/
$ echo 1 > ./state/zookeeper/myid

# in second node of cluster inside nifi-1.6.0/
$ echo 2 > ./state/zookeeper/myid

# in third node of cluster inside nifi-1.6.0/
$ echo 3 > ./state/zookeeper/myid

#Remove existing users.xml and authorizations.xml files if present :-
$ rm -rf users.xml
$ rm -rf authorizations.xml

#go to this location in each node and run this command to start Nifi with logs:-
$ bin/nifi.sh start && tail -f ./logs/nifi-app.log
