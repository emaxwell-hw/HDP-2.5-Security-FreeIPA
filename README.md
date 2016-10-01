#HDP 2.5 Security
##Create VMs
On the Field Openstack environment, create the following VMs:
- 1 m3.medium VM for the FreeIPA server
- 6 m3.large VMs for the HDP cluster

##Setup FreeIPA Server
To setup the FreeIPA server, install the server software:
<code>
ipaserver:~ # yum -y install ipa-server
...<i>copious amounts of output</i> 
</code>

Once the IPA server software is installed, you must configure IPA. In the Field Openstack Cloud, DNS is already configured 
<code>
ipaserver:~ # ipa-server-install
</code>

Enabling security in HDP 2.5/Ambari 2.4 with FreeIPA and CentOS 7

1. Have to enable experimental mode parameter enableipa
    - http://ambari.server:8080/#/experimental -> enableipa = true
2. Have to modify the krb5.conf to not use KEYRING (FILE=/tmp/krb5cc_${uid}
3. Have to create a group ambari-managed-principals
4. Have to modify the Spark user’s principal to include toLower() (e.g. ${cluster_name|toLower()})
