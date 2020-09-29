# Atlas Workshop
Purpose of this workshop is to understand Atlas architecture, configuration and features. We will also look into some use cases (Liineage, Ranger Tag policies).
In this workshop, we will understand the working of Atlas by decoupling each of the dependent service. First part introduces users to Atlas dependent services and how failure/misconfiguration of these services effects function of Atlas. At the end of the first part we will make sure Atlas is functional and summarize the role of each dependent service.

## *Prerequisites for Atlas workshop :*

* CDP DC cluster built on Squadron cluster with Data-Operational template.
* Kerberize cluster using One-click scripts.
* Update Atlas Configuration to use File Based Authentication (atlas.authentication.method.file).

## *Atlas Architecture for reference :*

We will discuss and understand all the components in this architecture throughout our workshop while reviewing configuration and troubleshooting in some areas.

![Atlas Architecture](Atlas_Architecture.png)

## *LAB 1 : Understanding Atlas service pre-requisites, configuration and troubleshooting the setup in squadron*
 
 
### *LAB 1.1 : Hbase configurations for Atlas service :*

**Step 1 :**  Once cluster is setup and Kerberize, try accessing Atlas UI and verify if it is up and running after restart. Atlas web UI should give 503 exception. 

**Step 2 :** Review the /var/log/atlas/application.log on atlas logs for further troubleshooting.

```
# tail -f /var/log/atlas/application.log
[..]
Caused by: org.janusgraph.diskstorage.TemporaryBackendException: Temporary failure in storage backend
at org.janusgraph.diskstorage.hbase2.HBaseStoreManager.ensureTableExists(HBaseStoreManager.java:732)
```

Prior to troubleshooting the issue, lets understand the atlas service pre-requisites.

Atlas depends on Hbase/HDFS,Kafka,Solr service. Before starting Atlas service , all these services must be up and running with permissions set on resource to allow Atlas user access.

---
**Step 3:** As configuring Atlas is already done by CM, we will review hbase related configs and fix some common issue.

Atlas is an application that uses graph database, we use janus graph(titan DB prior to HDP 3.x) database which relies on backend storage like Hbase or Cassandra.
Ref: [https://docs.janusgraph.org/](https://docs.janusgraph.org/) 

**Step 3.1 :** Review the atlas configuration to verify if all the configured backend services are available and accessible with atlas service principal. 
Login to atlas host and verify the configuration.

```
 # export ATLAS_PROCESS_DIR= ` ls -1dtr /var/run/cloudera-scm-agent/process/*ATLAS_SERVER | tail -1 `
 # egrep 'hbase|storage' $ATLAS_PROCESS_DIR/conf/atlas-application.properties
atlas.audit.hbase.zookeeper.quorum=c416-node4.coelab.cloudera.com,c416-node2.coelab.cloudera.com,c416-node3.coelab.cloudera.com
atlas.graph.storage.hostname=c416-node4.coelab.cloudera.com,c416-node2.coelab.cloudera.com,c416-node3.coelab.cloudera.com
atlas.graph.storage.hbase.table=atlas_janus
atlas.audit.hbase.tablename=ATLAS_ENTITY_AUDIT_EVENTS
```

/From the config we see that atlas.graph.storage is using Hbase table with name atlas_janus/
 
 
**Observation:** Atlas uses janus graph database which is configured to use Hbase as backend storage. By default atlas configured to use Hbase tables /atlas_janus/ and /ATLAS_ENTITY_AUDIT_EVENTS./
/‘Atlas_janus’ table used for storing actual graph data, ATLAS_ENTITY_AUDIT_EVENTS to maintain atlas audit events related to any CRUD operations on atlas entities./

**Step 3.2:** Access Hbase table to verify the mentioned table name exists and accessible for atlas user.  

```
 # kinit -kt ${ATLAS_PROCESS_DIR}/atlas.keytab atlas/ ` hostname -f `
 # klist
 # echo 'list' | hbase shell -n | grep -I atlas
```

**Step 3.3 :** If table atlas_janus doesn’t exist, verify same using Hbase keytab. Login to any other of the Hbase host to verify the same with Hbase key tab.

```
 # HBASE_KEYTAB= ` ls -1drt /var/run/cloudera-scm-agent/process/*hbase-REGIONSERVER/hbase.keytab | tail -1 `
 # kinit -kt $HBASE_KEYTAB hbase/ ` hostname -f `
 # klist
 # echo 'list' | hbase shell -n | grep -i atlas
ATLAS_ENTITY_AUDIT_EVENTS
atlas_janus
["ATLAS_ENTITY_AUDIT_EVENTS", "atlas_janus”]
```

/Note that Atlas backend DB should already exist as CM will auto create these resources while installing Atlas./

**Step 3.4 :** When kerberos is enabled, authorization is enabled by default on Hbase. Make sure that atlas user has permissions on Hbase tables /atlas_janus/ and /ATLAS_ENTITY_AUDIT_EVENTS/.

First step to verify what class name is set for authorization on Hbase service.

```
 # grep -C1 hbase.coprocessor.master.classes $ATLAS_PROCESS_DIR/hbase-conf/hbase-site.xml

<property>

<name>hbase.coprocessor.master.classes</name>
<value>org.apache.hadoop.hbase.security.access.AccessController</value>
```

If co-processor is set to /org.apache.hadoop.hbase.security.access.AccessController/, the class name for Hbase native authorization, Atlas user should be given access using HBase commands. 

/Note : Class name would org.apache.ranger.authorization.hbase.RangerAuthorizationCoprocessor if Ranger plugin is enabled on Hbase. In which case Ranger policies for Hbase are already configured by CM to allow atlas user access to these tables. Unless any hbase ranger plugin issues, there is no action needed./

**Step 3.5:** Configure permissions on hbase table : 

```
 # kinit -kt $HBASE_KEYTAB hbase/ ` hostname -f `
 # echo "user_permission 'atlas_janus'" |hbase shell -n
 # echo "user_permission 'ATLAS_ENTITY_AUDIT_EVENTS'" |hbase shell -n
```

If permissions are not set for above tables, make sure to grant RWX permissions to atlas user.

```
 # echo "grant 'atlas','RWXCA','atlas_janus'" | hbase shell -n
 # echo "grant 'atlas','RWXCA','ATLAS_ENTITY_AUDIT_EVENTS'" | hbase shell -n
 # echo "user_permission 'atlas_janus'" |hbase shell -n
 # echo "user_permission 'ATLAS_ENTITY_AUDIT_EVENTS'" |hbase shell -n
```

Restart Atlas service and verify if UI is accessible. (Squadron sets /‘hadoop12345!'/ as default local login password for /‘admin'/ user.)
Atlas should be up and running with primary pre-requisite, Hbase table access.

**Observations :** For atlas to be initialized and web ui to be accessible, Hbase tables configured must be accessible and Hbase table must be in consistent state.
Few common issues related to Hbase that can cause Atlas startup/initialization failures :
* Atlas user access issue with zookeeper, as this is primary entry point for accessing hbase tables. Fix any issues with zookeeper connection to start troubleshooting the issues
* Hbase table inconsistencies. Hbase Master UI should give the status of regions.
* Atlas Hbase access issues ( can be related to Ranger Plugin or permission on hbase tables)

### **LAB 1.2: Kafka configuration for Atlas, Atlas Hook and bridges:** 
 
Kafka is used as notification/messaging system for automated way to populate metadata into Atlas. Prior to reviewing how Kafka is used with Atlas, we will start with Atlas Bridges concept, which are helpful to populate existing metadata in the source prior to Atlas installation.
 
 
We will explore other pre-requisite services Solr and Kafka while understanding how metadata is populated to Atlas.
 
 
**LAB 1.2.1 : Understanding Atlas Hook and Bridges:**

We have 3 methods to populate metadata to Atlas:

**Atlas Bridge :** Atlas bridge is a simple class built on AtlasClientV2 class, which does a POST method to ATLAS API with the metadata fetched from service (like for hive we fetch metadata from Hive Metastore ).
With the atlas installation a shell script is provided as add-on per service.

/Code reference [https://github.infra.cloudera.com/CDH/atlas/tree/cdpd-master/addons](https://github.infra.cloudera.com/CDH/atlas/tree/cdpd-master/addons)/

These shell scripts call Atlas bridge code in backend , which further has implementation to fetch metadata from that respective service and push it to ATLAS via Atlas API.

**Step 1 :** Lets install Hive On Tez Service to understand how these Atlas bridge are used.
Install Hive on Tez service on cluster. (Hive is the very first service where Atlas integration was implemented.)
/Note: Hive on Tez depends on service YARN/HIVE/Tez, make sure to add these service in the order prior to adding Hive on Tez./

**Step 2 :** Create table on Hive to create some metadata for hive. Metadata for hive is created when hive objects like table/Db are created.

```
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "create table atlas_test_table(col1 string,col2 int);”
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "describe formatted atlas_test_table”
```

Output from describe command is the metadata we should see in Atlas UI once it gets written to Atlas.

**Step 3.1 :** Use import-hive.sh script to import hive metadata : 

```
 # kinit -kt $ATLAS_PROCESS_DIR/atlas.keytab atlas/ ` hostname -f `
 # export JAVA_HOME=/usr/java/jdk1.8.0_232-cloudera/
 # echo ‘hadoop’ | kinit admin/admin
 # /opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-hive.sh
Using Hive configuration directory [/etc/hive/conf]
[…]
Hive Meta Data imported successfully!!!
```

**Step 3.2 :** Login to atlas UI and use basic search with "Search by Type”: /hive_table/ and /"Search by text”/: /atlas_*/

**Step 3.3 :** Review the metadata and compare it with the above output of /“describe formatted atlas_test_table”./

**Step 3.4 :** Check the Audits tab for the atlas_test_table entity to see how this metadata got populated. Which should show user as ‘admin’ as we have executed import-hive.sh while authenticated as admin/admin user principal. 

**Observations:** Atlas Bridge is provided as a shell script per service, which gets the metadata from the service and writes to Atlas via POST API call.

**Step 4.1 :** Create another table using CTAS statement using above created table. 

```
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "create table atlas_test_ctas_bridge as (select * from atlas_test_table)”
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "show tables"
 # /opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-hive.sh
```

**Step 4.2 :** From atlas UI "Search by Type”: hive_table and "Search by text” : atlas_*
**Step 4.3 :** Review the Properties, Lineage and Audits for table atlas_test_ctas_bridge in Atlas.

**Observations:** import-hive/Atlas bridge will import the metadata but will not capture the Lineage. Lineage when captured, would show the parent table details for any CTAS operation.
We will review same operation while using Atlas hive Hook.

**Step 5:** Drop atlas_test_ctas_bridge table and re-execute import-hive script. 

```
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "drop table atlas_test_ctas_bridge;show tables; show tables;”
 # /opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-hive.sh
```

**Step 5.1 :** From atlas UI "Search by Type”: hive_table and "Search by text” : atlas_*

** Observations :** atlas_test_ctas_bridge and atlas_test_table metadata still exists. Import-hive script (or Atlas bridge) will import metadata but won’t act on existing atlas metadata which is already remove/dropped in source .

**Misc :** Below are the services and the respective import script paths: 

```
/opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-hbase.sh
/opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-hive.sh
/opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-kafka.sh
```

As we already got Hbase installed, lets execute import-hbase.sh script to populate HBase metadata.

```
 # kinit -kt $ATLAS_PROCESS_DIR/atlas.keytab atlas/ ` hostname -f `
 # /opt/cloudera/parcels/CDH-7.2.0-1.cdh7.2.0.p0.3758356/lib/atlas/hook-bin/import-hbase.sh
```

From Atlas UI “Search by Type”: hbase_table and make sure metadata is available for HBase tables.
Troubleshoot any issues with import-hbase.sh. (Hint: check for Ranger policy).

** Observations:**  For import scripts to work, user executing the script must have access to metadata on the source service and also on Atlas to create new entities.

* **Atlas Hooks :**  Main feature of Atlas and most of the use cases buillt based on Atlas Lineage feature. With Lineage, Atlas maintains history/changes to metadata. Lineage can be captured during runtime when there is any update to metadata which is possible with Atlas Hooks.

Atlas Hooks as name implies are hooks which are embedded into the service JVM with configurable option. If enabled on a service, this hook code is triggered when there is any change to metadata or any metadata is created/deleted. Atlas Hooks relies on kafka notification system to achieve this. Below is the flow for a typical Atlas Hook.

![Atlas Hook notification flow](Atlas_hook_Kafka.png)


**Step 1:** From CM UI, enable Atlas hook on hive_on_tez service(by selecting Atlas service is configuration), restart the services and verify configurations from HiveServer2: 
**Step 2:** Login to HiveServer2 to review the configurations for hook to work . 

```
 # export HIVE_PROCESS_DIR= ` ls -1dtr /var/run/cloudera-scm-agent/process/*hive_on_tez-HIVESERVER2 | tail -1 `
 # grep -C1 hive.exec.post.hooks $HIVE_PROCESS_DIR/hive-site.xml
<property>
<name>hive.exec.post.hooks</name>
<value>org.apache.hadoop.hive.ql.hooks.HiveProtoLoggingHook,org.apache.atlas.hive.hook.HiveHook</value>
```

**Step 3.1 :** With the hook enabled, class org.apache.atlas.hive.hook.HiveHook will be invoked for every metadata operation. Repeat the table creation exercise to see the difference. 

```
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "create table atlas_hook_table(col1 string,col2 int);show tables;"
```

**Step 3.2:** From atlas UI /"Search by Type”: hive_table/ and /"Search by text” : atlas_*/

**Step 3.3 :**  Metadata might not be available yet on Atlas UI. Review hiveserver2.log and see if any issues related to kafk.

```
＃grep 'Atlas Notifier' /var/log/hive/hadoop-cmf-hive_on_tez-HIVESERVER2-c416-node2.coelab.cloudera.com.log.out
```

**Step 4 :** Starting point for Atlas hooks is access to Kafka topic ATLAS_HOOK. Lets review the Kafka configuration, verify kafka health and fix any issues.

```
＃less $HIVE_PROCESS_DIR/atlas-application.properties
[…]
atlas.notification.topics=ATLAS_HOOK,ATLAS_ENTITIES

atlas.kafka.bootstrap.servers=c416-node4.coelab.cloudera.com:9092
atlas.kafka.security.protocol=SASL_PLAINTEXT
```

**Step 4.1 :** Create kafka client JAAS and config file to prepare for accessing Kafka broker.  
```
 # echo 'security.protocol=SASL_PLAINTEXT' > /var/tmp/kafka.config
 # cat > /var/tmp/kafka_jaas.conf
KafkaClient {
com.sun.security.auth.module.Krb5LoginModule required
useTicketCache=true
renewTicket=true
serviceName="kafka";
};

CTRL+D
```

**Step 4.2 :** Query Kafka to verify topics configured in atlas-application.properties : 

```
 # export KAFKA_OPTS='-Djava.security.auth.login.config=/var/tmp/kafka_jaas.conf'
 # kafka-topics --list --bootstrap-server c416-node4.coelab.cloudera.com:9092 --command-config /var/tmp/kafka.config
ATLAS_ENTITIES
ATLAS_HOOK
ATLAS_SPARK_HOOK
```

**Step 4.3 :** Make sure kafka topics exits and confirm atlas service is subscribed to ‘atlas’ kafka group ID. 

```
 # kafka-consumer-groups --list --bootstrap-server c416-node4.coelab.cloudera.com:9092 --command-config /var/tmp/kafka.config
```

If atlas group ID is not visible, review the kafka broker logs for any issues .

**Step 4.4 :** On Kafka host : 

```
 # tail -f /var/log/kafka/kafka-broker- ` hostname -f ` .log
[…]
2020-09-07 04:24:28,566 ERROR kafka.server.KafkaApis: [KafkaApi-1546332648] Number of alive brokers '1' does not meet the required replication factor '3' for the offsets topic (configured via 'offsets.topic.replication.factor'). This error can be ignored if the cluster is starting up and not all brokers are up yet.

```
As error shows, we don’t have enough brokers to match the replication factor. /*Add additional kafka brokers if required (or) change the replication.factor to match number of brokers.*/

**Step 4.5 :** Re-execute the Kafka-groups command to confirm atlas, group id exits. 

```
 # kafka-consumer-groups --list --bootstrap-server c416-node4.coelab.cloudera.com:9092 --command-config /var/tmp/kafka.config
```

**Step 4.6:** Review if there is any lag on consumer group ‘atlas’

```
 # kafka-consumer-groups --describe --group atlas --bootstrap-server c416-node4.coelab.cloudera.com:9092 --command-config /var/tmp/kafka.config
[…]
GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID
atlas ATLAS_SPARK_HOOK 0 - 0 - consumer-atlas-2-0807b7d8-99c2-48fc-9534-d77a58c9d511 /172.25.42.68 consumer-atlas-2
atlas ATLAS_HOOK 0 15 15 0 consumer-atlas-1-2f978b95-896f-4da7-9dba-02d63b057626 /172.25.42.68 consumer-atlas-1

```
From the above output we can see CURRENT-OFFSET and LAG, 0 LAG represents that consumer has consumed all the messages. In our case consumer here for ATLAS_HOOK is our Atlas service.

**Step 4.7 :** Login to Atlas UI and verify if the tables that were created are available. From atlas UI "Search by Type”: hive_table and "Search by text” : atlas_* 

Observations:  Atlas service subscribes to Kafka topic ATLAS_HOOK, for which Kafka service should be operational. Hive hook will publish any metadata change events during runtime to Kafka topic ATLAS_HOOK.

* **Atlas API :** We will use this method in second part of Lab with a custom type. 

**Task :** Referring to above command, try to see all the notifications published on ATLAS_HOOK topic.

### **LAB 1.3 : Atlas Search and Solr configurations :**
 
In this LAB we will review the configurations related to Basic search, the feature used by external clients for data discovery.
Atlas provides two options for search *Basic Search* and *Advanced/DSL search*: Basic search is faster as it gets the search results from Solr collections configured for Atlas. 

**Step 1:** From the Atlas service config, review the search related configurations. 

```
 # cat $ATLAS_PROCESS_DIR/conf/atlas-application.properties | grep atlas.graph.index
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.wait-searcher=true
atlas.graph.index.search.solr.zookeeper-url=c416-node4.coelab.cloudera.com:2181,c416-node2.coelab.cloudera.com:2181,c416-node3.coelab.cloudera.com:2181/solr
```

**Step 2:** Janus graph DB by defaults uses the solr collections with name *edge_index,vertex_index,fulltext_index.* Each index has its own purpose for indexing. With Atlas we rely more on vertex_index collection. 
Review the collections created on solr with above names.

```
 # solrctl --zk c416-node4.coelab.cloudera.com:2181,c416-node2.coelab.cloudera.com:2181,c416-node3.coelab.cloudera.com:2181/solr --jaas $ATLAS_PROCESS_DIR/conf/atlas_jaas.conf collection --list
vertex_index (5)
edge_index (5)
fulltext_index (5)
＃solrctl --zk c416-node4.coelab.cloudera.com:2181,c416-node2.coelab.cloudera.com:2181,c416-node3.coelab.cloudera.com:2181/solr --jaas $ATLAS_PROCESS_DIR/conf/atlas_jaas.conf cluster --get-clusterstate /tmp/test.out
 # less /tmp/test.out
 # curl --negotiate -u : 'http://c416-node2.coelab.cloudera.com:8983/solr/vertex_index/select?q=%28cn9_t%3A%28hive_table%29%29&wt=json&rows=100' | python -mjson.tool
```

**Step 3:** Equivalent query from frontend atlas to compare the results : 

```
 # curl -u admin:'hadoop12345!' -X POST -H 'content-type: application/json' -d '{"attributes":["qualifiedName"],"query":"atlas_*","limit":25,"offset":0,"typeName":"hive_table"}' http://c416-node4.coelab.cloudera.com:31000/api/atlas/v2/search/basic | python -mjson.tool
```

**Step 4:** Advanced search or DSL search accepts sql like syntax but is slower as this query is executed against the graph DB on HBase. This feature has limitations like using special characters. 
From Solr UI select Advanced search and try searching same table “atlas_*” and see the response time :

### **LAB 1.4 :Atlas Lineage**
In this LAB we will understand the Atlas Lineage feature and how a lineage is built using Atlas Hooks. After completing above steps, we should have fully functional Atlas and should be able to use the core features of Atlas.

**Step 1:**
Lineage for metadata is created when there is any change on existing metadata while Hook is enabled.

**Step 1.1 :** Create a new table with CTAS to see this feature. 

```
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "create table atlas_hook_table_ctas as (select * from atlas_hook_table);show tables;"
```

**Step 1.2 :** Login to Atlas UI and verify if the tables that were created are available. From atlas UI "Search by Type”: hive_table and "Search by text” : atlas_*. 
Review properties, Lineage and Audits for table atlas_hook_table_ctas

**Observations:** Lineage is created when metadata is created via hook. Audit captured in Atlas shows the actual user who created/updated the entity metadata. 

**Step 2.1:** Drop the intermediate table to see changes in Lineage and existing metadata.

```
 # beeline --silent=true -u 'jdbc:hive2://c416-node2.coelab.cloudera.com:10000/default;principal=hive/_HOST@COELAB.CLOUDERA.COM' -e "drop table atlas_hook_table;"
```

**Step 2.2:** Recreate the table with same name, and observe the changes to search results and Lineage of /atlas_hook_table_ctas/. 

**Step 2.3 :** Review the Kafka lag on atlas kafka group.

```
 # kafka-consumer-groups --describe --group atlas --bootstrap-server c416-node4.coelab.cloudera.com:9092 --command-config /var/tmp/kafka.config
[...]
GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID
atlas ATLAS_SPARK_HOOK 0 - 0 - consumer-atlas-2-0807b7d8-99c2-48fc-9534-d77a58c9d511 /172.25.42.68 consumer-atlas-2
atlas ATLAS_HOOK 0 19 19 0 consumer-atlas-1-2f978b95-896f-4da7-9dba-02d63b057626 /172.25.42.68 consumer-atlas-1
```

**Observations** : Dropping any table while Hook enabled, will mark the table as deleted and won’t show up in search (unless selected to show historical entities). 

**Summary of LAB 1 :** 
* Understanding configurations in Atlas for the dependency services Hbase,Kafka,Solr.
* Fixed issues faced with initial setup.
* Atlas Hook and Atlas bridge, difference in how these two methods populate metadata to Atlas.
* Atlas search feature (Basic and Advanced search)
* Atlas Lineage and how lineage gets created with the changes in metadata.
  
 With the fully functioning Atlas service, in next LAB we will explore more on core concepts of Atlas(Type system, entities, classifications, relationship, Glossary),Authentication/Authorization and one use case of Atlas Tags with dynamic tag based ranger policies.

 
 
