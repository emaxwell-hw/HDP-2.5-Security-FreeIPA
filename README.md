#HDP 2.5/Ambari 2.4 Kerberos with FreeIPA
This tutorial describes how to enable Kerberos using a FreeIPA server for LDAP and KDC functions on HDP 2.5. The following assumptions are made:<br>
- An existing HDP 2.5 cluster
- No existing IPA server
- There are sufficient resources to create an m3.medium VM to house the FreeIPA server
- DNS is already taken care of in the environment
- FreeIPA will run on RHEL/CentOS 7

##Step 1: Setup FreeIPA Server
###Create a VM for FreeIPA
On the Field Openstack environment, spin up an m3.medium VM with the CentOS 7.2 image. 

###Install FreeIPA Server
To setup the FreeIPA server, install the server software:
```
yum -y install ntp
systemctl enable ntpd
systemctl start ntpd

yum -y install ipa-server
ipa-server-install --domain=field.hortonworks.com \
    --realm=FIELD.HORTONWORKS.COM \
    --hostname=ipaserver.field.hortonworks.com
```

###Configure krb5.conf credential cache
HDP does not support the in-memory keyring storage of the Kerberos credential cache. Edit the <i>/etc/krb5.conf</i> file and change:
```
default_ccache_name = KEYRING:persistent:%{uid}
```
to
```
default_ccache_name = FILE:/tmp/krb5cc_%{uid}
```

###Create a hadoopadmin user
In order to create users in FreeIPA, an administrative use is required. The default admin@REALM user can be used (password created during IPA server install). Alternatively, create a hadoopadmin user:
```
ipa user-add hadoopadmin --first=Hadoop --last=Admin
ipa group-add-member admins --users=hadoopadmin
ipa passwd hadoopadmin
```
Ambari also requires a group to be created called ambari-managed-principals. This group is not currently created by the Ambari Kerberos wizard. Create the group:
```
ipa group-add ambari-managed-principals
```
Because of the way FreeIPA automatically expires the new password, it is necessary to kinit as hadoopadmin and change the initial password. The password can be set to the same password unless the password policy prohibits password reuse:
```
kinit hadoopadmin@REALM
```

##Step 3: Prepare the HDP Nodes
All nodes in the HDP cluster must have the ipa-client software installed and be joined to the FreeIPA server:
```
yum -y install ipa-client
ipa-client-install --domain=field.hortonworks.com \
    --server=ipaserver.field.hortonworks.com \
    --realm=FIELD.HORTONWORKS.COM \
    --principal=hadoopadmin@FIELD.HORTONWORKS.COM \
    --password=<i>hadoopadmin_password</i>
```
On the Amberi server node, install the ipa-admintools package:
```
yum -y install ipa-admintools
```

##Step 4: Enable Experimental FreeIPA Support
Support for FreeIPA is not enabled by default in Ambari. You must enable the experimental functionality in Ambari before you can select FreeIPA as an option in the Kerberos wizard. In a browser, navigate to:
```
http://ambariserver.field.hortonworks.com:8080/#/experimental
```
Check the box next to enableipa:

![Image](images/ambari-exp.png?raw=true)

##Step 5: Run the Kerberos Wizard
Run the Kerberos wizard from Ambari (Admin -> Kerberos -> Enable Kerberos). Select "Existing IPA" and verify that the prerequisites have been met.

![Image](images/ambari-kerb-wizard.png?raw=true)

Enter the appropriate information into the KDC page:

![Image](images/ambari-kdc-props.png?raw=true)

Click through to the Configure Identities page of the wizard. There is a bug in the name of the Spark principal that needs to be corrected. FreeIPA requires principal names to be in lower case, but ambari allows the cluster name to be in mixed case. If the cluster contains capital letters, the creation of the Spark principal will fail. To account for this, the principal names should all contain a reference to the toLower() function in the cluster name variable to ensure that capital letters are corrected before creating the principal.

Change the spark.history.kerberos.principal parameter to include the toLower() function: 

Change from:
```
${spark-env/spark_user}-${cluster_name}@${realm}
```
To:
```
${spark-env/spark_user}-${cluster_name|toLower()}@${realm}
```

![Image](images/ambari-princ-config.png?raw=true)

The rest of the Wizard should complete successfully.
