# db2textaudit

DB2 audit feature: https://www.ibm.com/support/knowledgecenter/SSEPGG_11.5.0/com.ibm.db2.luw.admin.sec.doc/doc/c0005483.html

Another DB2 Audit solution: https://github.com/stanislawbartkowski/db2audit

This solution is much simpler and is focused only on audit data collection. It is implemented as several Ansible playbooks. https://www.ansible.com/

The solution assumes we have a number DB2 instances and want to enable DB2 audit feature for every environment and collect all audit logs in a single place ready for further analysis.

It covers two tasks:<br>
* Activate DB2 feature and prepare passwordless connection between DB2 instance and "audit master node".
* Collect all audit logs and move them to a single directory on "audit master node" for further analysis. 

# Prerequisities

Three types of nodes:<br>

* *Provision node*. This node is used only for enabling DB2 audit feature and configuring the connection between DB2 instances and "audit master node". A privileged user is necessary on this node having sudo access to DB2 instances and *Audit master node*. The provision node and  the privileged user are used only for configuring task, can be dismantled or deactivated when the task is done.
* *Audit master node*. The configuration task creates a *db2audit* user on this node. The *db2audit* user has a passwordless connection with *db2audit* user on DB2 nodes and is authorized to run DB2 audit-related command. The *db2audit* user does not have access to DB2 tables and data and is not allowed to execute any action outside auditing scope.
* DB2 instances under monitoring.

Collected audit log files are stored on the *Audit master node* in the directory:<br>

\<auditlogroot\>/\<auditdir\>/\<DB2 hostname\>/\<db2 instance owner\>\<database\><br>
  
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

The log files are stored as text delimited CSV files. Data can be loaded into a database or analyzed using a text searching tool.

*Ansible* utility is installed on *Provision node* and *Audit master node*. No additional dependency is required for DB2 instance nodes.

# Security

* Privileged access to DB2 instance nodes is necessary only to activate DB2 auditing and setting up the passwordless connection between *Audit master node* and DB2 instance nodes. 
* To collect audit data, a passwordless connection from *Audit master node* to DB instance node is used. The connection is between *db2audit* users only and the *db2audit* user on DB2 instance node is not privileged. It is authorized only to run DB2 audit-related commands and cannot conduct any action outside DB2 audit scope and does not have any access to DB2 tables and data.

# Configuration

> git clone \<url\> <br>
> cd \<directory\>
> mkdir conf<br>
> cp conf.template/* conf/<br>
> cp conf.template/hosts hosts<br>

Configure *conf/env_vars.yml* and *hosts*. 
<br>
In *conf/env_vars.yml* modify the value of *audithost* variable to keep the *Audit master node* hostname.<br>

# Ansible inventory, *hosts* file

Example:<br>
```
[db]
db2a1.sb.com auditdir=dba
db2a2.sb.com auditdir=dbabank
db2a3.sb.com auditdir=dba

[auditmaster]
db2host.sb.coms
```

Inventory group *db* specifies the list of DB2 instances where DB2 audit facility is to be enabled. *auditmaster* is the hostname of *Audit master node*.<br>

Every host in *[db]* group has two additional variables defined.
* *auditdir* Subdirectory name in the collected audit logs allowing a group of nodes to be clustered together.

Using the example above:<br>

DB2 nodes *db2a1.sb.com* and *db2a3.sb.com* contain a single database SAMPLE to be monitored. The audit log files will be stored in the following directory structure.<br>

* auditlog/dba/db2a1.sb.com/db2inst1/SAMPLE
* auditlog/dba/db2a3.sb.com/db2inst1/SAMPLE

DB2 node *db2a2.sb.com* contains two databases: *SAMPLE* and *BANK* to be monitored. The audit log files for this node will be stored:<br>

* auditlog/dbabank/db2a2.sb.com/db2inst1/SAMPLE
* auditlog/dbabank/db2a2.sb.com/db2inst1/BANK

# Running the tool

## Configuring

Run this task as privilege user on a *Provision node*.

Configure *conf/env_vars.yml* and *hosts* inventory file.

> ansible-playbook -i hosts installaudit.yml <br>

Activates DB2 audit features on all DB2 instance node. The following actions are conducted on very node.<br>
* Create *db2audit* user<br>
* Give *db2audit* user *SECADM* privilege to every database under monitoring
* Define *audit* directories:<br>
  * db2audit configure datapath \<data_path\> \<archide_path\>
* Define DB2 audit policy for every database:<br>
  * create audit policy audit_validate categories validate status both ERROR TYPE NORMAL;
  * AUDIT database USING POLICY audit_validate;

It is possible to run the tool more than once. Running the tool again does not make any change in the existing configuration.

**Important**: After running the tool, the audit records are not generated immediately. It is necessary to restart the instance or *deactivate/activate* database to trigger that. It is not covered by the tool.<br>

> ansible-playbook -i hosts setupauditmaster.yml<br>

Creates *db2audit* user on *Audit master node* and setup passwordless ssh connection from *Audit master node* to *db2audit* user on DB2 instance node. Running the tool for the second time does not change anything on nodes where *db2audit* user and ssh connection are already created.<br>

## Log data collecting

Run it on *Audit master node* as *db2audit* user. Cnfiguration file: *conf/env_vars.yml*, *env/env_db.yml* and *hosts* inventory files are the same as for a *Provision node*.

> ansible-playbook -i hosts auditcollect.yml<br>

The following action is executed on every DB2 instance node.<br>

* As *dbaudit* user run the command:<br>
   * CALL SYSPROC.AUDIT_ARCHIVE(NULL, NULL);
   * CALL SYSPROC.AUDIT_DELIM_EXTRACT(NULL, '{{temp_audit_dir.path}}', NULL, NULL, NULL);
* Fetch extracted log audit data to *Audit master node* and store them in the directory:
  *  \<auditlogroot\>/\<auditdir\>/\<DB2 hostname\>/\<db2 instance owner\> 

The task is tranfering all log data available from DB2 instances to *Audit master node* and is rewriting the existing content of audit directory on *Audit master node*. 
