#HDP 2.5/Ambari 2.4 Kerberos with FreeIPA
This tutorial describes how to enable Kerberos using a FreeIPA server for LDAP and KDC functions on HDP 2.5. The following assumptions are made:<br>
- An existing HDP 2.5 cluster
- No existing IPA server
- There are sufficient resources to create an m3.medium VM to house the FreeIPA server
- The IPA Server will manage DNS for the cluster
- FreeIPA will run on RHEL/CentOS 7

##Step 1: Setup FreeIPA Server
###Create a VM for FreeIPA
On the Field Openstack environment, spin up an m3.medium VM with the CentOS 7.2 image. 

###Install Entropy Tools
Certain operations like generating encryption keys host entropy for creating random data. A fresh system with no processes running and no real device drivers can have issues generating enough random data for these types of operations. Install the rng-tools package and start rngd to help with this issue:
```
yum -y install rng-tools
systemctl start rngd
systemctl enable rngd
```

###Install FreeIPA Server
Install NTP and the FreeIPA software and start the NTP service:
```
yum -y install ntp ipa-server ipa-server-dns
systemctl enable ntpd
systemctl start ntpd
```

In order to use FreeIPA for domain resolution within the cluster, there are a few pieces of information that need to be collected:
- DNS servers for external lookups. These will be configured as "forwarders" in FreeIPA for handing off DNS resolution for external lookups.
- Reverse DNS Zone name. This is used for configuring reverse DNS lookups within FreeIPA. The FreeIPA server will calculate this based on the IP address and Netmask of the server if it is unknown.
- DNS domain to use for the cluster
- Kerberos realm to use for the cluster (by convention, usually the domain in uppercase)
- The hostname of the FreeIPA server
- The IP address to use for the FreeIPA server (if there is more than one on the host).

```
ipa-server-install --domain=example.domain.com \
    --realm=EXAMPLE.DOMAIN.COM \
    --hostname=ipaserver.example.domain.com \
    --ip-address=1.2.3.4
    --setup-dns \
    --forwarder=8.8.8.8 \
    --forwarder=8.8.8.4 \
    --reverse-zone=3.2.1.in-addr.arpa.
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
kinit admin@EXAMPLE.DOMAIN.COM
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
kinit hadoopadmin@EXAMPLE.DOMAIN.COM
```

##Step 2: Prepare the HDP Nodes
First, disable the `chronyd` service since it interferes with NTP (which FreeIPA prefers):
```
systemctl stop chronyd
systemctl disable chronyd
```

Configure the HDP nodes to use the FreeIPA server for DNS resolution:
```
echo "nameserver $ipaserver_ip_address" > /etc/resolv.conf
```

All nodes in the HDP cluster must have the ipa-client software installed and be joined to the FreeIPA server:
```
yum -y install ipa-client
ipa-client-install --domain=example.domain.com \
    --server=ipaserver.example.domain.com \
    --realm=EXAMPLE.DOMAIN.COM \
    --principal=hadoopadmin@EXAMPLE.DOMAIN.COM \
    --enable-dns-updates
```
On the Amberi server node, install the ipa-admintools package:
```
yum -y install ipa-admintools
```

##Step 3: Enable Experimental FreeIPA Support
Support for FreeIPA is not enabled by default in Ambari. You must enable the experimental functionality in Ambari before you can select FreeIPA as an option in the Kerberos wizard. In a browser, navigate to:
```
http://ambariserver.example.domain.com:8080/#/experimental
```
Check the box next to enableipa:

![Image](images/ambari-exp.png?raw=true)

##Step 4: Run the Kerberos Wizard
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
