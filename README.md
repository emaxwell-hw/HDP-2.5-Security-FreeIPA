#HDP 2.5 Security
##Create VMs
On the Field Openstack environment, create the following VMs:
- 1 m3.medium VM for the FreeIPA server
- 6 m3.large VMs for the HDP cluster

##Setup FreeIPA Server
To setup the FreeIPA server, install the server software:
<br>
<code>
ipaserver:~ # yum -y install ipa-server
</code>

Once the IPA server software is installed, you must configure IPA. In the Field Openstack Cloud, DNS is already configured, so no need to use FreeIPA for DNS resolution:
<br>
<code>
ipaserver:~ # ipa-server-install
...<i>copious amounts of output</i>
</code>

Enabling security in HDP 2.5/Ambari 2.4 with FreeIPA and CentOS 7

1. Have to enable experimental mode parameter enableipa
    - http://ambari.server:8080/#/experimental -> enableipa = true
2. Have to modify the krb5.conf to not use KEYRING (FILE=/tmp/krb5cc_${uid}
3. Have to create a group ambari-managed-principals
4. Have to modify the Spark user’s principal to include toLower() (e.g. ${cluster_name|toLower()})
