#HDP 2.5 Security
##Create VMs
Enabling security in HDP 2.5/Ambari 2.4 with FreeIPA and CentOS 7

1. Have to enable experimental mode parameter enableipa
    - http://ambari.server:8080/#/experimental -> enableipa = true
2. Have to modify the krb5.conf to not use KEYRING (FILE=/tmp/krb5cc_${uid}
3. Have to create a group ambari-managed-principals
4. Have to modify the Spark user’s principal to include toLower() (e.g. ${cluster_name|toLower()})
