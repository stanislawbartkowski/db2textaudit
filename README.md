# db2textaudit

DB2 audit feature: https://www.ibm.com/support/knowledgecenter/SSEPGG_11.5.0/com.ibm.db2.luw.admin.sec.doc/doc/c0005483.html

Another DB2 Audit solution: https://github.com/stanislawbartkowski/db2audit

This solution is much simpler and is focused only on audit data collection. It is implemented as a number of Ansible playbooks. https://www.ansible.com/

The solution assumes we have a number DB2 instances and want to enable DB2 audit feature for every environment and collect all audit logs in a single place ready for further analysis.

It covers two taks:<br>
* Activate DB2 feature and prepare passwordless connection between DB2 instance and "audit master node".
* Collect all audit logs and move them to a single directory on "audit master node" for further analysis. 

# Prerequisities

Three type of nodes:<br>

* Provision node. This node is used only for enabling DB2 audit feature and configuring connection between DB2 instances and "audit master node". A privileged user is necessary on this node having sudo access to DB2 instances and "audit master node". The provision node and privileged user is used only for configuring task, can be dismantled or deactivated when the task is done.
* Audit master node. The configuration task creates a *db2audit* user on this node. The *db2audit* user has a passwordless connection with *db2audit* user on DB2 nodes and is authorized to run DB2 audit-related command. The *db2audit* user does not have access to DB2 tables and data and is not allowed to execute any action outside auditing.
* DB2 instances under monitoring.

Collected audit log files are stored on the *Audit master node* in the directory:<br>

\<auditlogroot\>/\<auditdir\>/\<DB2 hostname\>/\<db2 instance owner\><br>
  
Example:
* auditlog/dba/db2a1.sb.com/db2inst1 
  * audit.del
  * auditlobs
  * checking.del
  * context.del
  * execute.del
  * objmaint.del
  * secmaint.del
  * sysadmin.del
  * validate.del

The log files are stored as text delimited CSV files. Data can be loaded into database or analyzed using text searching tool.

Ansible tool is installed on *Provisioning node* and *Audit master node*. No additional dependency is required on DB2 instance nodes.

# Security

* Privileged access to DB2 instance nodes is necessary only to activate DB2 auditing and setting up the passwordless connection between *Audit master node* and DB2 instance nodes. 
* To collect audit data, passwordless connection from *Audit master node* to DB instance node is used. The connection is between *db2audit* users only and the *db2audit* user on DB2 instance node is not privileged. It is authorized only to run DB2 audit related commands and cannot execute any action outside DB2 audit scope and does not have any access to DB2 tables and data.

# Configuration

> git clone \<url\> <br>
> cd \<directory\>
> mkdir conf<br>
> cp conf.template/* conf/<br>
> cp conf.template/hosts hosts<br>

Configure *conf/env_db.yml* and *hosts*. 
<br>
In *conf/env_vars.yml* modify the value of *audithost* variable to keep the *Audit master node* hostname.<br>

# List of databases, *conf/env_db.yml*

The file contains a list of databases to be monitored. DB2 audit feature is enabled at the database level and every database. Example:<br>

```YML
sample:
  - SAMPLE
samplebank:
  - SAMPLE
  - BANK
```

# Ansible inventory, *hosts* file

Example:<br>
```
[db]
db2a1.sb.com dbset=sample auditdir=dba
db2a2.sb.com dbset=samplebank auditdir=dbabank
db2a3.sb.com dbset=sample auditdir=dba

[auditmaster]
db2host.sb.coms
```

Inventory group *db* specifies the list of DB2 instances where DB2 audit facility is to be enabled. *auditmaster* is the hostname of *Audit master node*.<br>

Every host in *[db]* group has two additional variables defined.
* *dbset* A key in *conf/env_db.yml* file giving the list of databases to be monitored.
* *auditdir* Subdirectory name in the collected audit logs allowing a group of nodes to be clustered together.

In the example below:<br>

DB2 nodes *db2a1.sb.com* and *db2a3.sb.com* contains a single database SAMPLE to be monitored. The audit log files will be stored in the following directory structure.<br>
* auditlog/dba/db2a1.sb.com/db2inst1 
* auditlog/dba/db2a3.sb.com/db2inst1 

DB2 node *db2a2.sb.com* contains two databases: *SAMPLE* and *BANK* to be monitored. The audit log files for this node will be stored:<br>
* auditlog/dbabank/db2a2.sb.com/db2inst1 







