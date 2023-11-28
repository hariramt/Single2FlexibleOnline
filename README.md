# Migrate Azure Database for Postgresql Single Server to Flexible Server

To understand how the migration tool works, please refer to the Concept documentation [**here**](https://learn.microsoft.com/azure/postgresql/migrate/concepts-single-to-flexible)

## Prerequisite - Setup Logical Replication

- Set "Azure replication support" to "Logical" (don't restart yet)
- Set max_wal_senders and max_replication_slots to the number of databases you want to migrate and now restart the server.
- For PostreSQL 9.5 and 9.6 we have to explicitly allow replication connection. To enable that, add a firewall entry to allowlist connection from target. Make sure the firewall rule name has _replrule suffix. This step isn't required PostgreSQL 10 and 11.

## Start your Online migration
Two avenues are possible to perform the Online migration. Select the option that suits your requirement and follow the steps necessary:

[**Using the Azure Portal**](./single2flexibleonline-migrate-using-the-azure-portal.md)

[**Using the Azure CLI**](./single2flexibleonline-migrate-using-the-azure-cli.md)

## Limitations

* As with Online PostgreSQL migrations, all logical replication restrictions apply. Read more [here](https://www.postgresql.org/docs/current/logical-replication-restrictions.html).
* You can have only one active migration to your flexible server at a time.
* You can select a max of eight databases in one migration attempt. If you have more than eight databases, you must wait for the first migration to be complete before initiating another migration for the rest of the databases. Support for migration of more than eight databases in a single migration will be introduced later.
* The source and target server must be in the same Azure region. Cross region migrations are supported only in China regions.
* The tool migrates the data and schema. It doesn't migrate managed service features such as server parameters, connection security details, firewall rules, users, roles and permissions. These [post-migration steps](https://learn.microsoft.com/azure/postgresql/migrate/concepts-single-to-flexible#post-migration) will help with that.
* The migration tool shows the number of tables copied from source to target server. You need to validate the data in target server post migration.
* The tool only migrates user databases and not system databases such as template_0, template_1, azure_sys and azure_maintenance.