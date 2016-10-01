#HDP 2.5/Ambari 2.4 Kerberos with FreeIPA
This tutorial describes how to enable Kerberos using a FreeIPA server for LDAP and KDC functions on HDP 2.5. The following assumptions are made:<br>
- No existing IPA server is available
- There are sufficient resources to create an m3.medium VM to house the FreeIPA server
- DNS is already taken care of in the environment
- FreeIPA will run on RHEL/CentOS 7

##Step 1: Setup FreeIPA Server
###Create a VM for FreeIPA
On the Field Openstack environment, spin up an m3.medium VM with the CentOS 7.2 image. 

###Install FreeIPA Server
To setup the FreeIPA server, install the server software:<br><br>
<code>
ipaserver:~ # yum -y install ipa-server<p>
ipaserver:~ # ipa-server-install<p>
...<i>copious amounts of output</i>
</code>

###Configure krb5.conf ccache
HDP does not support the in-memory keyring storage of the Kerberos credential cache. Edit the <i>/etc/krb5.conf</i> file and change:<br><br>
<code>default_ccache_name = KEYRING:persistent:%{uid}</code><br><br>
to<br>
<code>default_ccache_name = FILE:/tmp/krb5cc_%{uid}</code>


##Step 3: Prepare the HDP VMs

##Step 4: Install the cluster

##Step 5: &lt;Step 5 Title&gt;



Enabling security in HDP 2.5/Ambari 2.4 with FreeIPA and CentOS 7

1. Have to enable experimental mode parameter enableipa
    - http://ambari.server:8080/#/experimental -> enableipa = true
2. Have to modify the krb5.conf to not use KEYRING (FILE=/tmp/krb5cc_${uid}
3. Have to create a group ambari-managed-principals
4. Have to modify the Spark user’s principal to include toLower() (e.g. ${cluster_name|toLower()})
