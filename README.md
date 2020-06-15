# Transition Workflow

Configure Alias (dislike typing)
```
alias ap=ansible-playbook
alias ape=ansible-playbook --extra-vars
alias apt=ansible-playbook --tags
```

## Prep
**Build Local Repo**
`ap devops/repo_sync.yaml`
Then check and unpack CM tarball

### Before you Start
- Ensure all hosts are consistent
- Backup the Databases

### AM2CM Tooling
**Build and Deploy Tools**
__SKIP IF USING MOD'd version__
`ap tooling/am2cm_build.yaml`

**Deploy AM2CM on to an Edge Node**
`ap tooling/am2cm_install.yaml`

**Deploy the Ranger Policy Migration Tool**
`ap tooling/ranger_migration_install.yaml`

**Configure AM2CM**
`ap tooling/am2cm_configure.yaml`

## CM Installation
**Install Cloudera Manager**
`ap cm/cm_install.yaml`
This installs CM and creates the databases for CM and Report Manager and deploys JDBC jars and MariaDB Clients where needed.

### Setup Cloudera Manager for Transition
- Set Networking/Repo Locations for Local Repos
- Adjust Distribution Threads 
- Set CM TLS
- Configure Kerberos
- Add Hosts
> Need to have all hosts added that are used in Ambari Cluster
- Add Cloudera Management Service
    - Start the CMS Service

## Transition to Cloudera Manager (Step #1)
**NOTE: Waiting for silent mode in am2cm Tool**

`ap tooling/ambari_blueprint.yaml`

Goto edge and run:
```
cd am2cm-conversion/
../am2cm-1.0-SNAPSHOT/am2cm.sh -bp HOME90-blueprint.json  -dt HOME90-cm_template.json
```
then on the ansible host...
`apt "submit" tooling/transition.yaml`


#### Or with Silent Mode
Once the `-s` mode is available for am2cm run the entire `transition.yaml`:
`ap tooling/transition.yaml` 

## Deployment in Cloudera Manager (Step #2)

In Cloudera Manager 'Download' and 'Distribute' the Parcel. Do not 'Activate' yet.

`ap devops/save_namespace.yaml`
Ensure Ambari Cluster is Health and that HDFS is in good shape:
- Save HDFS Namespace
- Ensure no underreplicated blocks
- Check the 'Exclude Hosts' list is empty. IE: No dead/inactive datanodes.

In Ambari, shutdown all Services and 'Stop' and 'Disable' all **Ambari Agents**.  The **takeover** starts NOW.

Disable Ambari Agents
`ap ambari/agent_disable.yaml`

In Cloudera Manager, 'Activate' the Parcel on all hosts.  This will reconfigure all the symlinks for the cluster configurations.

## Reseting the Hadoop Environment (Step #3)
The goal here is to NOT start services until required and to minimize HDFS restarts to manage time.

### First Phase Cleanup
- Reset Kafka and ZooKeeper Id's
> `ap reset/cm_reset_phase_1.yaml`  
- Start ZooKeeper
- Remove /hadoop-ha znode
- Deploy the HDFS Clients
- From Namenode as `hdfs` user:
    - formatZK
    > `hdfs zkfc -formatZK`

## Configuring Basic Services (Step #4)
### Enable TLS Security
- TLS

### Add Service
Yarn Queue Manager 
- Set YARN Dependency in YARN Config.
KNOX

### Setting up Governance Components
When other services aren't running, this will fail during 'setup'.  When it does, click on Cloudera Manager Logo (upper left) and acknowledge 'leave'.  The service will still be installed.
- Solr (will error when hdfs not running)
- Atlas (will rename Solr Service to CDP-INFRA-SOLR) (will also fail when services aren't running.)
- Ranger

### Validate / Check Safety Value Settings in:
- HDFS
- YARN
- Hive
- Hive On Tez
- Spark
- Oozie
- TEZ
- Kafka
- ZooKeeper

### Tweak other Service Settings
#### Hive
##### #1
Remove port from metastore db url.

#### YARN
##### #1
Issue with yarn.nodemanager.runtime.linux.allowed-runtimes (remove docker)

##### #2
Issue Yarn NodeManagers fail to start with error:
 /var/lib/yarn-ce/bin/container-executor error=13, permission denied.
dstreev@w01 /var/lib/yarn-ce/bin
---Sr-s--- 1 root yarn 414398 May 20 16:33 container-executor
Fix:
yarn.nodemanager.linux-container-executor.group (reset to 'yarn').  Is set to 'hadoop' in HDP.

##### #3
Check/Fix path for: yarm.nodemanager.linux-container.executor.group=yarn

##### #4
yarn.nodemanager.linux-container.executor.resources-handler.class ....

#### ZooKeeper
Add `4lw.commands.whitelist=mntr,conf,ruok` to zoo.cfg safety value to support Solr inspection and monitoring.

## Configuring Governance (Step #5)

### Ranger Admin
Start Ranger Services
- Action - Setup Ranger Plugin Services. (Adds linked repos to Ranger Services)

### Migrate Ranger Policies
`ap tooling/ranger_service_migration.yaml`
Failures expected when service repos aren't available on either side.
- HDFS
- YARN
- Hive
- HBase
- Kafka
- Atlas
- Knox
- Solr

### Ranger HDFS Policy Modifications
- Add `hue` as 'SuperUser' to HDFS Policy ''
> Allows CM to complete following installations.
- Create policy for `/solr-infra` resource (hdfs path) and grant full rights to `solr` user.

### Activate Ranger for Each Service
**NOTE: Do NOT activate Ranger for services not previously under Ranger control in Ambari.  This may cause service disruption.  We suggest addressing this after the migration.**
- HDFS
- YARN
- Hive
- Hive on Tez (HS2)
- HBase
- Kafka
- Atlas
- Knox
- Solr

### Enable Kerberos
- Kerberos
> Add CM kdc user to 'retrieve keytab' for the HTTP/m01.streever.local service principal. (Needed when using IPA host as a host in the cluster)

## Start Bringing the rest of the Cluster Up (Step #6)
### Restart / Start Core Services
- ZooKeeper
- HDFS
    - Create `/solr-infra` HDFS directory
- YARN
- CLDR-INFRA-SOLR

### Setting Up Service Dependencies
* YARN
    - Create Jobs History Dir
    - Create NodeManager Remote Application Log Directory
    - Install YARN MapReduce Framework JARs
    - Install YARN Service Dependencies
    - Create Ranger Plugin Audit Directory
* Hive
    - Create Hive User Directory
    - Create Hive Warehouse Directory
    - Create Hive Warehouse External Directory
    - Create Ranger Plugin Audit Directory
* Hive On Tez
    - Create Ranger Plugin Audit Directory
* TEZ
    - Upload Tez tar file to HDFS
* Oozie
    - Install Oozie ShareLib
* Kafka
    - Create Ranger Kafka Plugin Audit Directory
* HBase
    - Create Ranger Plugin Audit Directory
* Solr
    - Create HDFS Home Directory
    - Create Ranger Plugin Audit Directory

### Start Remaining Services
- Kafka
- Start Hive Service
    - Validate Hive Metastore Schema
    > To ensure all previous schema upgrades have been applied.
- Hive on TEZ
- HBase
- Oozie
- Spark
- Atlas
- Hue
- Knox
          
---


## RESET (soft - don't kill)

### CM to Ambari

1. Stop ALL services in CM
2. Disable CM Agents
    `ap cm/agent_disable.yaml`
3. Enable and Configure Ambari HDP
    `ap ambari/agent_enable.yaml`
    `ap reset/hdp_service_reset.yaml`
4. Goto Ambari and Start Services    

### Ambari to CM

1. Stop ALL services in Ambari
2. Disable Ambari Agents
    `ap ambari/agent_disable.yaml`
3. Enable and Configure CM
    `ap cm/agent_enable.yaml`
    ? ...

## RESET (HARD - Kill Environment)

### CM

1. Stop all components in CM
2. Turn off, disable, and remove CM
    `ap cm/cm_remove.yaml`




