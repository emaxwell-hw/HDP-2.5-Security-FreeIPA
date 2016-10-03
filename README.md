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
<pre>
yum -y install ntp
systemctl enable ntpd
systemctl start ntpd

yum -y install ipa-server
ipa-server-install --domain=field.hortonworks.com \
    --realm=FIELD.HORTONWORKS.COM \
    --hostname=ipaserver.field.hortonworks.com
</pre>

###Configure krb5.conf credential cache
HDP does not support the in-memory keyring storage of the Kerberos credential cache. Edit the <i>/etc/krb5.conf</i> file and change:
<pre>default_ccache_name = KEYRING:persistent:%{uid}</pre>
to
<pre>default_ccache_name = FILE:/tmp/krb5cc_%{uid}</pre>

###Create a hadoopadmin user
In order to create users in FreeIPA, an administrative use is required. The default admin@REALM user can be used (password created during IPA server install). Alternatively, create a hadoopadmin user:
<pre>
ipa add-user hadoopadmin --first=Hadoop --last=Admin
ipa group-add-member admins --users=hadoopadmin
ipa passwd hadoopadmin
</pre>
Ambari also requires a group to be created called ambari-managed-principals. This group is not currently created by the Ambari Kerberos wizard. Create the group:
<pre>
ipa group-add ambari-managed-principals
</pre>
Because of the way FreeIPA automatically expires the new password, it is necessary to kinit as hadoopadmin and change the initial password. The password can be set to the same password unless the password policy prohibits password reuse:
<pre>
kinit hadoopadmin@REALM
</pre>

##Step 3: Prepare the HDP Nodes
All nodes in the HDP cluster must have the ipa-client software installed and be joined to the FreeIPA server:
<pre>
yum -y install ipa-client
ipa-client-install --domain=field.hortonworks.com \
    --server=ipaserver.field.hortonworks.com \
    --realm=FIELD.HORTONWORKS.COM \
    --principal=hadoopadmin@FIELD.HORTONWORKS.COM \
    --password=<i>hadoopadmin_password</i>
</pre>
On the Amberi server node, install the ipa-admintools package:
<pre>
yum -y install ipa-admintools
</pre>

##Step 4: Enable Experimental FreeIPA Support
Support for FreeIPA is not enabled by default in Ambari. You must enable the experimental functionality in Ambari before you can select FreeIPA as an option in the Kerberos wizard. In a browser, navigate to:
<pre>
http://ambariserver.field.hortonworks.com:8080/#/experimental
</pre>
Check the box next to enableipa:
<image href="images/ambari-exp.png">

##Step 5: &lt;Step 5 Title&gt;



Enabling security in HDP 2.5/Ambari 2.4 with FreeIPA and CentOS 7

1. Have to enable experimental mode parameter enableipa
    - http://ambari.server:8080/#/experimental -> enableipa = true
2. Have to modify the krb5.conf to not use KEYRING (FILE=/tmp/krb5cc_${uid}
3. Have to create a group ambari-managed-principals
4. Have to modify the Spark user’s principal to include toLower() (e.g. ${cluster_name|toLower()})
