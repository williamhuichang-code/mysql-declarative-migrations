# MySQL Declarative Schema & Data Migrations

Versioned schema and data migrations for MySQL using **[schema-data-migration (sdm)](https://github.com/Beim/schema-data-migration)**, with separate production and development databases running in **Docker**.

Completed as a technical exercise. It was my first time using Docker and declarative migrations, so I learned the concepts and syntax along the way.

## What this shows

- **Two environments in Docker:** a production MySQL 5.7 container (port 3306) and a development one (port 3307).
- **Declarative schema changes:** describe the desired table in `schema/user.sql`, and sdm works out the change and records it as a versioned migration plan.
- **Data migrations with rollback:** a seed-data migration with both `forward` and `backward` SQL.
- **Full rollback:** from version `0002` back to `0000`, with every step recorded in the migration history.
- **Version control:** migration plans are tracked with Git.

## Migration timeline

| Version | Type | Change | Result |
|---|---|---|---|
| `0000` | schema | Initial snapshot of `awesome_db` | Applied |
| `0001` | schema | Add `address` column to `user` | Applied, then rolled back |
| `0002` | data | Seed `user` with one row | Applied, then rolled back |

The exported [`history/`](history/) tables confirm the end state: only `0000` remains in `_migration_history`, while `_migration_history_log` keeps the full trail of creates, successes and rollbacks.

## Repository structure

```
mysql-declarative-migrations/
├── awesome_project/            ← sdm project
│   ├── schema/user.sql         ← desired table definition (declarative)
│   ├── migration_plan/         ← versioned plans: 0000, 0001, 0002
│   ├── .schema_store/          ← schema snapshots referenced by the plans
│   └── .env.example            ← copy to .env and add your password
└── history/                    ← exported migration history tables (CSV)
```

Secrets are not committed: `.env` is ignored, and the password in the commands below is a placeholder.

## Task environment

For this exercise, I used **MySQL 5.7** as the database engine and **beim/schema-data-migration:latest** (sdm) for handling the schema and data migrations, both run as **Docker** containers following the step-by-step guide. I set up two environments: a prod container on port **3306** and a dev container on port **3307**. 

And instead of hardcoding these values throughout my script, I declared them upfront as variables (`$MysqlImage`, `$SdmImage`, `$ProdPort`, `$DevPort`) at the top, so every command later in the script references them consistently.

## How sdm works

[SDM](https://github.com/Beim/schema-data-migration) is a tool for managing database migrations in both development and production environments.

Basically, you tell it what you want the database to look like, and it compares that with the current database to work out what needs to change. It then turns those changes into a migration plan, applies them to the database, and keeps track of what happened. 

The idea is that the database has a history of versions as the database evolves, so you can move between versions when needed, rather than just move from one command execution to another.

## One challenge I encountered and solved

One challenge I ran into was while testing sdm data migration tool against a public "world" database. I hit a foreign key constraint error when inserting a row into the city table, which left that migration stuck in a "PROCESSING" state in _migration_history instead of "SUCCESSFUL". And once stuck, sdm wouldn't let me run any new migration or rollback until it was resolved. 

I tried Docker's built-in AI assistant and asked ChatGPT, but neither gave me a working fix. Eventually I went back to sdm's own documentation and found the answer in the "Fix migration and rollback" section. Running "sdm fix migrate dev --fake" cleared the stuck state and let me move on. 

The takeaway: it's always worth checking the official documentation first, rather than assuming a general-purpose AI assistant will know the specifics of a specialised tool.

---

## Full workflow

The following walks through my exercise end-to-end, from environment setup through rollback.

### 0. Set up Docker and define environment details

Run the commands in PowerShell with Docker Desktop running.

#### 0.1. Define several variables

Adjust `$Operator`, `$RootPassword`, and `$ExerciseRoot` to your own setup —
the rest (container names, ports, and image tags) can stay as-is.

```powershell
$Operator      = "williamhuichang"
$RootPassword  = "<your-password>"
$ExerciseRoot  = "D:\DS\Xtracta_exercise"
$ProjectName   = "awesome_project"
$ProjectDir    = Join-Path $ExerciseRoot $ProjectName
$ProdContainer = "awesome-prod"
$ProdPort      = 3306
$DevContainer  = "awesome-dev"
$DevPort       = 3307
$MysqlImage    = "mysql:5.7"
$SdmImage      = "beim/schema-data-migration:latest"
```

#### 0.2. Pull two docker images

```bash
docker pull $MysqlImage
docker pull $SdmImage
```

#### 0.3. Create two docker containers

```bash
docker run -d --name $ProdContainer -e MYSQL_ROOT_PASSWORD=$RootPassword -p "${ProdPort}:3306" $MysqlImage
docker run -d --name $DevContainer -e MYSQL_ROOT_PASSWORD=$RootPassword -p "${DevPort}:3306" $MysqlImage
```

### 1. Prepare databases

Created a production database with an existing table, and an empty dev database, to simulate migrating a real schema onto a fresh environment.

#### 1.1. Prepare production database

Dive into a MySQL prompt inside the production container.

```bash
docker exec -it $ProdContainer mysql -uroot "-p$RootPassword"
```

Create awesome database following the command in the guide.

```sql
CREATE DATABASE `awesome_db`;
USE `awesome_db`;
CREATE TABLE `user` (
  `id` int(11) NOT NULL,
  `name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

Confirm results for tables, schemas and potential data.

```sql
SHOW TABLES;
DESCRIBE user;
SELECT * FROM user;
```

Type exit to leave the MySQL prompt.

```bash
exit
```

#### 1.2. Prepare dev database

Dive into a MySQL prompt inside the dev container.

```bash
docker exec -it $DevContainer mysql -uroot "-p$RootPassword"
```

Create another dev database (it starts empty), following the command in the original guide.

```sql
CREATE DATABASE `awesome_db`;
```

Confirm results for tables.

```sql
SHOW TABLES;
```

Type exit to leave the MySQL prompt.

```bash
exit
```

### 2. Initialize project

#### 2.1. Create the project directory

Create the project directory and move into it.

```powershell
mkdir $ProjectDir && cd $ProjectDir
```

#### 2.2. Git Initialisation

Initialize a git repository in the project directory to track migration version history.

```bash
git init
```

#### 2.3. Project Initialisation

Define an Invoke-DockerSdm helper and use it all along in the following steps.

```powershell
function Invoke-DockerSdm {
    docker run `
        -v "${ProjectDir}:/workspace" `
        -e "MYSQL_PWD=$RootPassword" `
        -it --rm $SdmImage `
    sdm @args
}
```

Initialize the awesome project.

```powershell
Invoke-DockerSdm init --host host.docker.internal --port $ProdPort -u root --schema awesome_db
```

### 3. Add environment

Register the dev environment in the project config, so later commands can reference it by name instead of repeating its connection details.

```powershell
Invoke-DockerSdm add-env --host host.docker.internal --port $DevPort -u root dev
```

### 4. Migrate

`Invoke-DockerSdm migrate dev` (from original `sdm migrate dev`) reconstructs the dev database's schema to match production. It's a one-time, manually-triggered action — it doesn't run continuously or watch for changes, it only acts when we invoke it.

#### 4.1. Run the migration

```powershell
Invoke-DockerSdm migrate dev "-o$Operator"
```

#### 4.2. Verifying results

Check that `awesome_db` in dev, and it should now have tables (_migration_history, _migration_history_log and user).

```bash
docker exec -it $DevContainer mysql -uroot "-p$RootPassword" -e "USE awesome_db; SHOW TABLES;"
```

Check the table structure, which should match production.

```bash
docker exec -it $DevContainer mysql -uroot "-p$RootPassword" -e "USE awesome_db; DESCRIBE user;"
```

Confirm no rows came across — `migrate` syncs schema only, not data, should result in 0 count.

```bash
docker exec -it $DevContainer mysql -uroot "-p$RootPassword" -e "USE awesome_db; SELECT COUNT(*) FROM user;"
```

### 5. Make a schema migration plan

Schema migrations are declarative: edit the project's schema file to describe the table's desired
end state, then generate a migration plan that captures the diff.

#### 5.1. Declare what the schema should look like

Following the original step-by-step guide, add an `address` column to the `user` table, after `name`, in
`awesome_project/schema/user.sql`, so that the content becomes:

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL,
  `name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `address` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### 5.2. Generate the schema migration plan

Name and generate a migration plan for this change.

```powershell
Invoke-DockerSdm make-schema add_address_to_user_tab
```

This writes the plan out as a JSON file in
`awesome_project/migration_plan/`:
`0001_add_address_to_user_tab.json`.

#### 5.3. Apply the migration to dev

Apply the generated plan to the dev database, using the same `migrate dev` command as before.

```powershell
Invoke-DockerSdm migrate dev "-o$Operator"
```

#### 5.4. Verifying results

Check that the `user` table's schema now includes the new `address` column.

```powershell
docker exec -it $DevContainer mysql `
    -uroot `
    "-p$RootPassword" `
    -e "USE awesome_db; DESCRIBE user;"
```

Check the migration history — migration `0001` should show a `successful` state.

```powershell
Invoke-DockerSdm info dev
```

Check the migration history log for the full trail of past changes.

```powershell
docker exec -it $DevContainer mysql `
    -uroot `
    "-p$RootPassword" `
    -e "USE awesome_db; SELECT * FROM _migration_history_log;"
```

### 6. Make a data migration plan

This time, we make migration plan for data instances.

#### 6.1. Declare what the data should look like

This time, create the plan json file first. The `0002_seed_user_table.json` plan will be generated in
`awesome_project/migration_plan/`.

```powershell
Invoke-DockerSdm make-data seed_user_table sql
```

Then, make changes by editing the json template directly. Alter forward section, and add backward section (optional but recommended).

```json
{
    "version": "0002",
    "name": "seed_user_table",
    "author": "",
    "type": "data",
    "change": {
        "forward": {
            "type": "sql",
            "sql": "INSERT INTO `user` (`id`, `name`, `address`) VALUES (1, 'foo', 'bar');"
        },
        "backward": {
            "type": "sql",
            "sql": "DELETE FROM `user` WHERE `id`=1;"
        }
    },
    "dependencies": [
        {
            "version": "0001",
            "name": "add_address_to_user_tab"
        }
    ]
}
```

#### 6.2. Apply the migration to dev

Apply the generated plan to the dev database, using the same `migrate dev` command as before.

```powershell
Invoke-DockerSdm migrate dev "-o$Operator"
```

#### 6.3. Verifying results

Check that the `user` table now has an instance with `id` 1.

```powershell
docker exec -it $DevContainer mysql `
    -uroot `
    "-p$RootPassword" `
    -e "USE awesome_db; SELECT * FROM user;"
```

Check the migration history — migration `0002` should show a `successful` state.

```powershell
Invoke-DockerSdm info dev
```

Check the migration history log for the full trail of past changes.

```powershell
docker exec -it $DevContainer mysql `
    -uroot `
    "-p$RootPassword" `
    -e "USE awesome_db; SELECT * FROM _migration_history_log;"
```

### 7. Roll back

#### 7.1. Roll back to version 0000

Roll back to version `0000`, just like the original guide. This rollback only works because `0002` has a `backward` block as noted above. Otherwise this will raise an error.

```powershell
Invoke-DockerSdm rollback --version 0000 dev "-o$Operator"
```

#### 7.2. Verifying results

Check the migration history — `0002` and `0001` are gone; only the initial `0000` migration remains.

```powershell
Invoke-DockerSdm info dev
```

Check the migration history log for the full trail of past changes.

```powershell
docker exec -it $DevContainer mysql `
    -uroot `
    "-p$RootPassword" `
    -e "USE awesome_db; SELECT * FROM _migration_history_log;"
```
