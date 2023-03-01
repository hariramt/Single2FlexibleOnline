---
title: "Tutorial: Online Migration of Azure Database for PostgreSQL - Single Server to Flexible Server using the Azure CLI"
titleSuffix: "Online Migration : Azure Database for PostgreSQL Flexible Server"
description: "Learn about Online migration of your Single Server databases to Azure Database for PostgreSQL Flexible Server by using the Azure CLI."
author: hariramt
ms.author: hariramt
ms.service: postgresql
ms.topic: tutorial
ms.date: 03/03/2023
ms.custom: seo-lt-2023
---

# Tutorial: Online Migration of Azure Database for PostgreSQL - Single Server to Flexible Server using the Azure CLI

You can perform Online migration of an instance of Azure Database for PostgreSQL – Single Server to Azure Database for PostgreSQL – Flexible Server by using the Azure Command Line Interface (CLI). In this tutorial, we perform Online migration of a sample database from an Azure Database for PostgreSQL single server to a PostgreSQL flexible server using the Azure CLI.

>[!NOTE]
> The migration tool is in preview.

In this tutorial, you learn about:

> [!div class="checklist"]
>
> * Prerequisites
> * Getting started
> * Migration CLI commands
> * Monitor the migration
> * Cancel the migration
> * Migration best practices

## Prerequisites

To complete this tutorial, you need to:

* Use an existing instance of Azure Database for PostgreSQL – Single Server (the source server)
* All extensions used on the Single Server (source) must be [allow-listed on the Flexible Server (target)](https://learn.microsoft.com/azure/postgresql/migrate/concepts-single-to-flexible#allow-list-required-extensions)

> [!IMPORTANT]
> To provide the best migration experience, performing migration using a burstable instance of Flexible server is not supported. Please use a general purpose or a memory optimized instance (4 VCore or higher) as your Target Flexible server to perform the migration. Once the migration is complete, you can downscale back to a burstable instance if necessary.

## Getting started

1. Install the latest Azure CLI for your operating system from the [Azure CLI installation page](https://learn.microsoft.com/cli/azure/install-azure-cli).

   If the Azure CLI is already installed, check the version by using the `az version` command. The version should be **2.45.0** or later to use the migration CLI commands. If not, [update your Azure CLI version](https://learn.microsoft.com/cli/azure/update-azure-cli).

2. Run the `az login` command:
   
   ```bash
   az login
   ```

   A browser window opens with the Azure sign-in page. Provide your Azure credentials to do a successful authentication.

## Migration CLI commands

The migration tool comes with easy-to-use CLI commands to do migration-related tasks. All the CLI commands start with  `az postgres flexible-server migration`.
Allow-list all required extensions as shown in [Migrate from Azure Database for PostgreSQL Single Server to Flexible Server](https://learn.microsoft.com/azure/postgresql/migrate/concepts-single-to-flexible#allow-list-required-extensions). It is important to allow-list the extensions before you initiate a migration using this tool.
For help with understanding the options associated with a command and with framing the right syntax, you can use the `help` parameter:

```azurecli-interactive
az postgres flexible-server migration --help
```

The above command gives you the following output:

:::image type="content" source="./media/az-postgres-flexible-server-migration-help.png" alt-text="Screenshot of Azure Command Line Interface help." lightbox="./media/az-postgres-flexible-server-migration-help.png":::

The output lists the supported migration commands, along with their actions. Let's look at these commands in detail.

### Create a migration using the Azure CLI

The `create` command helps in creating a migration from a source server to a target server:

```azurecli-interactive
az postgres flexible-server migration create -- help
```

The above command gives you the following result:

:::image type="content" source="./media/az-postgres-flexible-server-migration-create.png" alt-text="Screenshot of the command for creating a migration." lightbox="./media/az-postgres-flexible-server-migration-create.png":::

It lists the expected arguments and has an example syntax for successfully creating a migration from the source server to the target server. Here's the CLI command to create a new migration:

```azurecli
az postgres flexible-server migration create [--subscription]
                                            [--resource-group]
                                            [--name] 
                                            [--migration-name] 
                                            [--properties]
                                            [--migration-mode]
```

| Parameter | Description |
| ---- | ---- |
|`subscription` | Subscription ID of the Flexible Server target. |
|`resource-group` | Resource group of the Flexible Server target. |
|`name` | Name of the Flexible Server target. |
|`migration-name` | Unique identifier to migrations attempted to Flexible Server. This field accepts only alphanumeric characters and does not accept any special characters, except a hyphen (`-`). The name can't start with `-`, and no two migrations to a Flexible Server target can have the same name. |
|`properties` | Absolute path to a JSON file that has the information about the Single Server source. |
|`migration-mode` | 'offline' or 'online' are the accepted values. Use online for Online migration. Not including this (parameter,value) results in offline moigration by default. |

For example:

```azurecli-interactive
az postgres flexible-server migration create --subscription 11111111-1111-1111-1111-111111111111 --resource-group my-learning-rg --name myflexibleserver --migration-name CLIMigrationExample --properties "C:\Users\Administrator\Documents\migrationBody.JSON" --migration-mode online
```

The `migration-name` argument used in the `create` command will be used in other CLI commands, such as `update`, `delete`, and `show.` In all those commands, it uniquely identifies the migration attempt in the corresponding actions.

Finally, the `create` command needs a JSON file to be passed as part of its `properties` argument.

The structure of the JSON is:

```bash
{
"properties": {
 "SourceDBServerResourceId":"/subscriptions/<subscriptionid>/resourceGroups/<src_ rg_name>/providers/Microsoft.DBforPostgreSQL/servers/<source server name>",

"SecretParameters": {
    "AdminCredentials": 
    {
  "SourceServerPassword": "<password>",
  "TargetServerPassword": "<password>"
    }
},

"DBsToMigrate": 
   [
   "<db1>","<db2>"
   ],

"OverwriteDBsInTarget":"true"

}

}

```

The `create` parameters that go into the json file format are as shown below:

| Parameter | Type | Description |
| ---- | ---- | ---- |
| `SourceDBServerResourceId` | Required |  This parameter is the resource ID of the Single Server source and is mandatory. |
| `SecretParameters` | Required | This parameter lists passwords for admin users for both the Single Server source and the Flexible Server target, along with the Azure Active Directory app credentials. These passwords help to authenticate against the source and target servers. They also help in checking proper authorization access to the resources.
| `DBsToMigrate` | Required | Specify the list of databases that you want to migrate to Flexible Server. You can include a maximum of eight database names at a time. |
| `OverwriteDBsinTarget` | Required | When set to true (default), if the target server happens to have an existing database with the same name as the one you're trying to migrate, migration tool automatically overwrites the database. |
| `SetupLogicalReplicationOnSourceDBIfNeeded` | Optional | You can enable logical replication on the source server automatically by setting this property to `true`. This change in the server settings requires a server restart with a downtime of two to three minutes. |
| `SourceDBServerFullyQualifiedDomainName` | Optional |  Use it when a custom DNS server is used for name resolution for a virtual network. Provide the FQDN of the Single Server source according to the custom DNS server for this property. |
| `TargetDBServerFullyQualifiedDomainName` | Optional |  Use it when a custom DNS server is used for name resolution inside a virtual network. Provide the FQDN of the Flexible Server target according to the custom DNS server. <br> `SourceDBServerFullyQualifiedDomainName` and `TargetDBServerFullyQualifiedDomainName` are included as a part of the JSON only in the rare scenario that a custom DNS server is used for name resolution instead of Azure-provided DNS. Otherwise, don't include these parameters as a part of the JSON file. |

Note these important points for the command response:

- As soon as the `create` command is triggered, the migration moves to the `InProgress` state and the `PerformingPreRequisiteSteps` substate. The migration workflow takes a couple of minutes to deploy the migration infrastructure and setup connections between the source and target.
- If **Online migration** is selected, it requires **Logical replication** to be turned on in the source Single server. If it is not turned on, the migration movies into the `WaitingForUserAction` state and the `WaitingForLogicalReplicationSetupRequestOnSourceDB` substate. User should enable logical replication on the Source single server to advance from this state. Note that this action requires a restart of the Source single server. Logical replication can be turned on in the Single Server portal under `Replication` or through CLI using the [`--setup-replication` command](#setup-replication).
- After the `PerformingPreRequisiteSteps` substate is completed, the migration moves to the substate of `Migrating Data`, where the Cloning/Copying of the databases take place.
- Each database migrated has its own section with all migration details, such as table count, incremental inserts, deletions, and pending bytes.
- The time that the `Migrating Data` substate takes to finish depends on the size of databases that are migrated.
- After the copy/clone of the base data is complete, the migration moves to `WaitingForUserAction` state and `WaitingForCutoverTrigger` substate. In this state, user can trigger cutover from the portal by selecting the migration or through CLI using the [`--cutover` command](#cutover-the-migration).
- The migration moves to the `Succeeded` state as soon as the `Migrating Data` substate or the cutover (in case of Online) finishes successfully. If there's a problem at the `Migrating Data` substate, the migration moves into a `Failed` state.

>[!NOTE]
> Gentle reminder to [allow-list the extensions](https://learn.microsoft.com/azure/postgresql/migrate/concepts-single-to-flexible#allow-list-required-extensions) before you execute **Create** in case it is not yet done. It is important to allow-list the extensions before you initiate a migration using this tool.

### Setup replication

For Online migration, in case replication has not been setup at the source, it can be enabled using the below command. Please note that this command restarts the source Single server.

For example:

```azurecli-interactive
az postgres flexible-server migration update --subscription 11111111-1111-1111-1111-111111111111 --resource-group my-learning-rg --name myflexibleserver --migration-name CLIMigrationExample --setup-replication
```

### Cutover the migration

After the base data migration is complete, the migration task moves to `WaitingForCutoverTrigger` substate. In this state, user can trigger cutover from the portal by selecting the migration name in the migration grid or through CLI using the command below.

For example:

```azurecli-interactive
az postgres flexible-server migration update --subscription 11111111-1111-1111-1111-111111111111 --resource-group my-learning-rg --name myflexibleserver --migration-name CLIMigrationExample --cutover
```

Before initiating cutover it is important to ensure that writes to the source are stopped and Pending changes to be written to the Target are zero. This information can be obtained using the [migration show command](#monitor-the-migration).
Here's a snapshot of the response when initiating the cutover:

:::image type="content" source="./media/az-postgres-flexible-server-migration-cutover-command.png" alt-text="Screenshot of Command Line Interface migration Show." lightbox="./media/az-postgres-flexible-server-migration-cutover-command.png":::

After cutover is initiated, pending data captured during CDC is written to the target and migration is now complete.

:::image type="content" source="./media/az-postgres-flexible-server-migration-cutover-success.png" alt-text="Screenshot of Command Line Interface migration Show." lightbox="./media/az-postgres-flexible-server-migration-cutover-success.png":::

If the cutover is not successful, the migration moves to `Failed` state.

For more information about this command, use the `help` parameter:

```azurecli-interactive
 az postgres flexible-server migration update -- help
 ```

### List the migration(s)

The `list` command lists all the migration attempts made to a Flexible Server target:

```azurecli
az postgres flexible-server migration list [--subscription]
                                            [--resource-group]
                                            [--name]  
                                            [--filter] 
```

The `filter` parameter has two options:

- `Active`: Lists the current active migration attempts (in progress) into the target server. It does not include the migrations that have reached a failed, canceled, or succeeded state.
- `All`: Lists all the migration attempts into the target server. This includes both the active and past migrations, regardless of the state.

For more information about this command, use the `help` parameter:

```azurecli-interactive
az postgres flexible-server migration list -- help
```

## Monitor the migration

The `show` command helps you monitor ongoing migrations and gives the current state and substate of the migration.
These details include information on the current state and substate of the migration.

```azurecli
az postgres flexible-server migration show [--subscription]
                                            [--resource-group]
                                            [--name]  
                                            [--migration-name] 
```

The `migration_name` parameter is the name you have assigned to the migration during the `create` command. Here's a snapshot of the sample response from the CLI command for showing details:

:::image type="content" source="./media/az-postgres-flexible-server-migration-show.png" alt-text="Screenshot of Command Line Interface migration Show." lightbox="./media/az-postgres-flexible-server-migration-show.png":::

For more information about this command, use the `help` parameter:

```azurecli-interactive
 az postgres flexible-server migration show -- help
 ```

The following tables describe the migration states and substates.

| Migration state | Description |
| ---- | ---- |
| `InProgress` | The migration infrastructure is set up, or the actual data migration is in progress. |
| `WaitingForUserAction` | The migration task is waiting for user input/action. |
| `Canceled` | The migration is canceled or deleted. |
| `Failed` | The migration has failed. |
| `Succeeded` | The migration has succeeded and is complete. |

| Migration substate | Description |
| ----  | ---- |
| `PerformingPreRequisiteSteps` | Infrastructure is set up and is prepped for data migration. |
| `WaitingForLogicalReplicationSetupRequestOnSourceDB` | The migration task is waiting for the user to setup replication on the source Single server. |
| `WaitingForCutoverTrigger` | The migration task is waiting for the user to cutover the migration so it can move to completed state. |
| `MigratingData` | Data migration is in progress. |
| `CompletingMigration` | Migration cutover is in progress. |
| `Completed` | Cutover was successful, and migration is complete. |

## Cancel the migration

You can cancel any ongoing migration attempts by using the `cancel` command. This command stops the particular migration attempt, but it doesn't drop or roll back any changes on your target server. Here's the CLI command to delete a migration:

```azurecli
az postgres flexible-server migration update cancel [--subscription]
                                            [--resource-group]
                                            [--name]  
                                            [--migration-name]
```

For example:

```azurecli-interactive
az postgres flexible-server migration update cancel --subscription 11111111-1111-1111-1111-111111111111 --resource-group my-learning-rg --name myflexibleserver --migration-name CLIMigrationExample
```

For more information about this command, use the `help` parameter:

```azurecli-interactive
 az postgres flexible-server migration update cancel -- help
 ```

The command gives you the following output:

:::image type="content" source="./media/az-postgres-flexible-server-migration-update-cancel-help.png" alt-text="Screenshot of Azure Command Line Interface Cancel." lightbox="./media/az-postgres-flexible-server-migration-update-cancel-help.png":::

## Migration best practices

- For a successful end-to-end migration, follow the post-migration steps in [Migrate from Azure Database for PostgreSQL Single Server to Flexible Server](https://learn.microsoft.com/azure/postgresql/migrate/concepts-single-to-flexible#best-practices).