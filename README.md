# Transition Workflow

Configure Alias (dislike typing)
```
alias ap=ansible-playbook
alias ape=ansible-playbook --extra-vars
alias apt=ansible-playbook --tags
```

## Assumptions
- [ ] Source Cluster (Ambari) is NOT Kerberized.  That's because we are getting a Kerberos ticket for any of the playbooks run against the Ambari Cluster.  It 'can' be done, just hasn't for this version of the process.
- [ ] Running CDP-DC (Ambari 7.1.0+)

## Prep
**Build Local Repo**
[Repo Sync](devops/repo_sync.yaml)
`ap devops/repo_sync.yaml`
Then check and unpack CM tarball

### Before you Start
- Ensure all hosts are consistent
- Backup the Databases

### AM2CM Tooling
__SKIP IF USING MOD'd version__
**[AM2CM Build-out](tooling/am2cm_build.yaml)**
`ap tooling/am2cm_build.yaml`

**[AM2CM Install](tooling/am2cm_install.yaml)**
`ap tooling/am2cm_install.yaml`

**[Deploy Ranger Migration Tool](tooling/ranger_migration_install.yaml)**
`ap tooling/ranger_migration_install.yaml`

**[Configure AM2CM for Environment](tooling/am2cm_configure.yaml)**
`ap tooling/am2cm_configure.yaml`

## Step #1 - Cloudera Manager Installation
**[Install Cloudera Manager](cm/cm_install.yaml)**

- [ ] `ap cm/cm_install.yaml`

This installs CM and creates the databases for CM and Report Manager and deploys JDBC jars and MariaDB Clients where needed.

### Setup Cloudera Manager for Transition
- [ ] Add License Key
- [ ] Skip Cluster Wizard
- [ ] Set Networking/Repo Locations for Local Repos
- [ ] Adjust Distribution Threads 
- [ ] Set CM TLS
    - Restart CM
- [ ] Add Hosts
> Need to have all hosts added that are used in Ambari Cluster
- [ ] Add Cloudera Management Service
    - [ ] Start the CMS Service

## Step #2 - Transition to Cloudera Manager

**NOTE: Waiting for silent mode in am2cm Tool**

- [ ] [Get an Ambari Blueprint](tooling/ambari_blueprint.yaml)

`ap tooling/ambari_blueprint.yaml`

- [ ] Goto edge and run:
```
cd am2cm-conversion/
../am2cm-1.0-SNAPSHOT/am2cm.sh -bp HOME90-blueprint.json  -dt HOME90-cm_template.json
```
then on the ansible host...

- [ ] [Submit Template to CM](tooling/transition.yaml)

`apt "submit" tooling/transition.yaml`


#### Or with Silent Mode
Once the `-s` mode is available for am2cm run the entire `transition.yaml`:
[Transition](tooling/transition.yaml)

`ap tooling/transition.yaml` 

## Step #3 - Deployment in Cloudera Manager

In Cloudera Manager:

- [ ] 'Download' Cloudera Runtime Parcel
- [ ] 'Distribute' the Parcel. Do not 'Activate' yet.
 
- [ ] [Save HDFS Namespace on Running Cluster](devops/save_namespace.yaml)

`ap devops/save_namespace.yaml`

The **takeover** starts NOW.

In Ambari:

- [ ] Shutdown all Services 
- [ ] [Disable Ambari Agents](ambari/agent_disable.yaml)

`ap ambari/agent_disable.yaml`

In Cloudera Manager:

- [ ] Activate the Parcel on all hosts.  
> This will reconfigure all the symlinks for the cluster configurations.

## Step #4 - Reseting the Hadoop Environment
The goal here is to NOT start services until required and to minimize HDFS restarts to manage time.

### First Phase Cleanup
- [ ] [Reset Kafka and ZooKeeper Id's](reset/cm_reset_phase_1.yaml)

`ap reset/cm_reset_phase_1.yaml`
  
- [ ] Add `-Dzookeeper.skipACL=yes` to 'Java Configuration Options for Zookeeper Server'
> This allow zNode adjustments without auth on the zNode.  Helpful for maintenance when zNodes had been created and secured by a SASL user.
- [ ] Start ZooKeeper
- [ ] Remove /hadoop-ha zNode
> HDFS HA needs to be reset for Cloudera Manager
- [ ] Deploy the HDFS Clients
From Namenode as `hdfs` user:
- [ ] formatZK `hdfs zkfc -formatZK`

## Step #5 - Configuring Basic Services
### Enable TLS Security
- [ ] TLS

### Add Services
- [ ] Yarn Queue Manager 
- [ ] Set YARN Dependency in YARN Config.
- [ ] KNOX

### Add Governance Components
When other services aren't running, this will fail during 'setup'.  When it does, click on Cloudera Manager Logo (upper left) and acknowledge 'leave'.  The service will still be installed.

- [ ] Solr (will error when hdfs not running)
- [ ] Atlas (will rename Solr Service to CDP-INFRA-SOLR) (will also fail when services aren't running.)
- [ ] Ranger

### Validate / Check Safety Value Settings in:
- [ ] HDFS
- [ ] YARN
- [ ] Hive
- [ ] Hive On Tez
- [ ] Spark
- [ ] Oozie
- [ ] TEZ
- [ ] Kafka
- [ ] ZooKeeper

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

## Step # 6 - Configuring Governance

### Ranger Admin
- [ ] Start Ranger Services
- [ ] Create Service Repos in Ranger. 
> Ranger->Action->Setup Ranger Plugin Services. (Adds linked repos to Ranger Services)

### Migrate Ranger Policies
- [ ] [Ranger Service Upsert to new Service Repos](tooling/ranger_service_migration.yaml)

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
- [ ] Add `hue` as 'SuperUser' to HDFS Policy ''
> Allows CM to complete following installations.
- [ ] Create policy for `/solr-infra` resource (hdfs path) and grant full rights to `solr` user.

### Activate Ranger for Each Service
**NOTE: Do NOT activate Ranger for services not previously under Ranger control in Ambari.  This may cause service disruption.  We suggest addressing this after the migration.**

- [ ] HDFS
- [ ] YARN
- [ ] Hive
- [ ] Hive on Tez (HS2)
- [ ] HBase
- [ ] Kafka
- [ ] Atlas
- [ ] Knox
- [ ] Solr

### Enable Kerberos
- [ ] Kerberos
> This will trigger a FULL cluster 'START'.  It's important to have all the previous configurations done before this to avoid having to 'restart' HDFS, which will take a long time for large clusters.
> Add CM kdc user to 'retrieve keytab' for the HTTP/m01.streever.local service principal. (Needed when using IPA host as a host in the cluster)

## Step #7 - Complete Component Platform Dependencies

### All Service Started

All services should be started at this point.  If they aren't, research the error(s) and bring them online.

### Setting Up Service Dependencies
* YARN (Action Menu)
    - [ ] Create Jobs History Dir
    - [ ] Create NodeManager Remote Application Log Directory
    - [ ] Install YARN MapReduce Framework JARs
    - [ ] Install YARN Service Dependencies
    - [ ] Create Ranger Plugin Audit Directory
* Hive (Action Menu)
    - [ ] Create Hive User Directory
    - [ ] Create Hive Warehouse Directory
    - [ ] Create Hive Warehouse External Directory
    - [ ] Create Ranger Plugin Audit Directory
* Hive On Tez (Action Menu)
    - [ ] Create Ranger Plugin Audit Directory
* TEZ (Action Menu)
    - [ ] Upload Tez tar file to HDFS
* Oozie (Action Menu)
    - [ ] Install Oozie ShareLib
* Kafka (Action Menu)
    - [ ] Create Ranger Kafka Plugin Audit Directory
* HBase (Action Menu)
    - [ ] Create Ranger Plugin Audit Directory
* Solr (Action Menu)
    - [ ] Create HDFS Home Directory
    - [ ] Create Ranger Plugin Audit Directory
  
## Step #8 - Smoke Tests
TBD
        
---


## RESET (soft - don't kill)

### CM to Ambari

- [ ] Stop ALL services in CM
- [ ] Disable CM Agents

    `ap cm/agent_disable.yaml`

- [ ] Enable and Configure Ambari HDP

    `ap ambari/agent_enable.yaml`

    `ap reset/hdp_service_reset.yaml`

- [ ] Goto Ambari and Start Services    

### Ambari to CM

- [ ] Stop ALL services in Ambari
- [ ] Disable Ambari Agents

    `ap ambari/agent_disable.yaml`
    
- [ ] Enable and Configure CM
    
    `ap cm/agent_enable.yaml`
    
    ? need to reset links for binaries and configs. looking for hdp-select like function in CM

## RESET (HARD - Kill CM Environment)

### CM

- [ ] Stop all components in CM
- [ ] Turn off, disable, and remove CM

    `ap cm/cm_remove.yaml`
    
### Ambari:
- [ ] Enable and Configure Ambari HDP

    `ap ambari/agent_enable.yaml`
    
    `ap reset/hdp_service_reset.yaml`
    
- [ ] Goto Ambari and Start Services    




