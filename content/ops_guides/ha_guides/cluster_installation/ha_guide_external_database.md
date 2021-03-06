---
title: Cloudify High Availability - External Database
category: High Availabilty Guides
draft: false
weight: 100
---
## Environment

Cloudify HA cluster builds on three Cloudify managers. When creating a cluster using an external database, the join operation is run during the installation phase of the joining nodes.  
Therefore, 3 linux machines will need to be prepared each with the proper cloudify-manager-installer rpm, but only 1 node will have Cloudify Manager installed on it with an external database (follow this guide - [Installing Cloudify Manager with an external database]({{< relref "install_maintain/installation/installing-manager-with-an-external-db.md" >}})  
Once finished the remaining 2 nodes will be installed and join the master node 

### Network Ports Requirements

Before you begin, make sure you have the following prerequisites configured.
In addition to Cloudify Manager single node network requirements, the 3-node cluster has some network requirements as well.

**Cloudify Manager HA cluster:**

| Source | <-> | Target | Port | Description |
|--------|-----------|--------|------|-------------|
| Cloudify Manager | <-> | Cloudify Manager | 8300 | Internal port for the distributed key/value store. |
| Cloudify Manager | <-> | Cloudify Manager | 8301 | Internal port for TCP and UDP heartbeats. Must be accessible for both TCP and UDP. |
| Cloudify Manager | <-> | Cloudify Manager | 8500 | Port used for outage recovery in the event that half of the nodes in the cluster failed. |
| Cloudify Manager | <-> | Cloudify Manager | 22000 | Filesystem replication port. |

## Build Cloudify HA cluster

Once you finished installing the first Cloudify Manager and preparing the other 2 nodes, do the following:
When you run the `cfy cluster start` command on a first Cloudify Manager, high availability is configured automatically. Use the `cfy cluster join` command, following installation, to add more Cloudify Managers to the cluster. The Cloudify Managers that you join to the cluster must be in an empty state, otherwise the operation will fail.

1. On your first Cloudify Manager (master node) run `cfy cluster start --cluster-node-name <Leader name>`  
When you run the `cfy cluster start` command on a first Cloudify Manager, high availability is configured automatically.

2. Once the configuration is finished, login to the second prepared machine

3. There are 2 options to configure a machine to join to a cluster during bootstrap:  
    ##### Option 1:
      
    In case you haven't done already - update the following sections in the [config.yaml]({{< relref "install_maintain/installation/installing-manager.md#additional-cloudify-manager-settings" >}}) file as below:
       
      ```yaml
     ssl_inputs:
       postgresql_client_cert_path: '<PostgreSQL client certificate path>'
       postgresql_client_key_path: '<PostgreSQL client key path>'
       ca_cert_path: '<Root/Intermediate CA certificate path>'
       ca_key_path: '<Root/Intermediate CA certificate path>'
     ```
     
    Run:
    
      ```
      cfy_manager install --private-ip <Private IP> \
          --public-ip <Public IP> \
          --admin-password <Leader's admin password> \
          --join-cluster <Leader's Public/Private IP> \
          --database-ip <Leader's external database IP> \
          --postgres-password <Leader's external database postgres user's password>
      ```
{{% note %}}
Selecting this option will  
 - Make cluster-host-ip used for cluster communication be the Private IP.  
 - Generate a random cluster-node-name used for representing the node in the cluster.
{{% /note %}}
       
    ##### Option 2:
    Update the following sections in the [config.yaml]({{< relref "install_maintain/installation/installing-manager.md#additional-cloudify-manager-settings" >}}) file as below:

      ```yaml
     manager:
       private_ip: '<Private IP>'
       public_ip: '<Public IP>'
       security:
         admin_password: '<Master admin password>'
     cluster:
       master_ip: '<Master IP>'
       node_name: '<Joining machine node name represented in the cluster>'
       cluster_host_ip: '<Joining machine interface to use for cluster networking>'
     .
     .
     .
     postgresql_client:
       host: '<External database host[:<External database port>]>'
       postgres_password: '<postgres password configured on the external PostgreSQL server>'
       ssl_enabled: true
     .
     .
     .
     ssl_inputs:
       postgresql_client_cert_path: '<PostgreSQL client certificate path>'
       postgresql_client_key_path: '<PostgreSQL client key path>'
       ca_cert_path: '<Root/Intermediate CA certificate path>'
       ca_key_path: '<Root/Intermediate CA certificate path>'
     .
     .
     .
     services_to_install:
      - 'queue_service' 
      - 'composer_service' 
      - 'manager_service' 
     ```
    Run:
    
     ```
     cfy_manager install
     ```

{{% note %}}
 - Leaving cluster_host_ip and cluster-node-name empty will achieve the same behavior as in option 1  
{{% /note %}}

{{% warning title="Warning" %}}
The external database IP must be the same as the Leader's external database IP
{{% /warning %}}

4. Repeat steps 2-3 on the 3rd machine

## Cloudify HA cluster management

Cloudify HA cluster manages in the same way as a single Cloudify manager, but there are small differences when a leader changing. 

Cloudify CLI profile contains all information about managers of the HA Cluster and if the leader manager does not answer Cloudify CLI starts finding new leader.

If new profile is created for existing cluster, or new nodes joined to the cluster the command should be run to retrieve the information about all cluster managers and upgrade the profile:
```
cfy cluster update-profile
```

When using Cloudify WEB UI -  Cloudify HA cluster does not provide out of the box mechanism to update the WEB UI  to switch to a new leader due to Security limitations. Cloudify WEB UI User should make sure to have a mechanism to be aware which Cloudify Manager is the current leader. There are several well known mechanisms to achieve this, for example using a Load Balancer, using a Proxy such as HAProxy and configure it to poll the cluster IPs,  or using a CNAME instead of explicit IPs.

You can also implement a [load balancer health check]({{< relref "working_with/manager/high-availability-clusters.md#implementing-a-load-balancer-health-check" >}}).