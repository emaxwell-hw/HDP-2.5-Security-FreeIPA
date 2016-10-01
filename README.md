#HDP 2.5/Ambari 2.4 Kerberos with FreeIPA
This tutorial describes how to enable Kerberos using a FreeIPA server for LDAP and KDC functions on HDP 2.5. The following assumptions are made:<br>
- No existing IPA server is available
- There are sufficient resources to create an m3.medium VM to house the FreeIPA server
- DNS is already taken care of in the environment
- FreeIPA will run on RHEL/CentOS 7

##Step 1: Create VMs
On the Field Openstack environment, create the following VMs:
- 1 m3.medium VM for the FreeIPA server
- 6 m3.large VMs for the HDP cluster

##Step 2: Setup FreeIPA Server
To setup the FreeIPA server, install the server software:
<br>
<code>
ipaserver:~ # yum -y install ipa-server
</code>

Once the IPA server software is installed, you must configure IPA. In the Field Openstack Cloud, DNS is already configured, so no need to use FreeIPA for DNS resolution:
<br>
<code>
ipaserver:~ # ipa-server-install<br>
...<i>copious amounts of output</i>
</code>

##Step 3: Prepare the HDP VMs

##Step 4: Install the cluster

##Step 5: &lt;Step 5 Title&gt;



Enabling security in HDP 2.5/Ambari 2.4 with FreeIPA and CentOS 7

1. Have to enable experimental mode parameter enableipa
    - http://ambari.server:8080/#/experimental -> enableipa = true
2. Have to modify the krb5.conf to not use KEYRING (FILE=/tmp/krb5cc_${uid}
3. Have to create a group ambari-managed-principals
4. Have to modify the Spark user’s principal to include toLower() (e.g. ${cluster_name|toLower()})
