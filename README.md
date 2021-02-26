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

Ansible tool is installed on *Provisioning node* and *Audit master node*. No additional dependency is required on DB2 instance nodes.

# Security

* Privileged access to DB2 instance nodes is necessary only to activate DB2 auditing and setting up the passwordless connection between *Audit master node* and DB2 instance nodes.
