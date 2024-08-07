# OMOPonFHIR Setup Guide

This repository is an installable server deployment package that locks down the version of included components (using submodules). Please use the following step to compile and deploy the server. You may need to customize some of the environment variables.

Note: If you want to see current snapshots of OMOPonFHIR components or other versions of FHIR and OMOP, please visit the OMOPonFHIR GitHub organization at https://github.com/omoponfhir/

This package is tested with Google Big Query instance. And, the server supports mapping for FHIR R4 and OMOP v5.4. Please see omopv5_4_setup/ folder for some help on ddls for extra tables/views. Database ddls are also included.

**Note:** This repository contains submodules of anoter repositories that are needed. If you want to participate in 
development and contribute, please use the repositories directly as this submodule points to a certain commit point. 
Refer the follow repositories and use the latest to work on the code.

```
- path = omoponfhir-omopv5-sql
- url = https://github.com/omoponfhir/omoponfhir-omopv5-sql.git
- branch = 5.4

- path = omoponfhir-omopv5-r4-mapping
- url = https://github.com/omoponfhir/omoponfhir-omopv5-r4-mapping.git
- branch = sqlRender

- path = omoponfhir-r4-server
- url = https://github.com/omoponfhir/omoponfhir-r4-server.git
- branch = sqlRender
```

## 1. Setup the repository and submodules

```bash
# Clone the repository, including submodules with --recurse
git clone --recurse https://github.com/omoponfhir/omoponfhir-main-v54-r4.git
# Change directory to the cloned repository
cd omoponfhir-main-v54-r4
```
You will need `Apache Maven` installed to proceed with the next steps, you can check if you have it in `$PATH` with the following command:
```bash
which mvn # Should return the path to the maven executable
```
If you have it installed, you can skip the next step. If not, please install it before proceeding.

### 1.1 Maven Installation
If you don't have it installed, please download it from [here](https://maven.apache.org/download.cgi).

The package seems to be targeted at openJDK 21 or later, check your java version with:
```bash
java -version
```
If you don't have it installed, you can download it from [here](https://jdk.java.net/22/) and follow the installation instructions.

### 1.2 Build the project
After `Maven` is installed, in the cloned repository, run the following command to build the project:
```bash
mvn clean install
```

## 2. Setup the OMOP database

Clone this repository with:
```bash
git clone git@github.com:omoponfhir/omopv5_4_setup.git
```

If you already have a running omop database instance, start from step 2.6.

### 2.1. Fetch Vocabulary and inject CPT codes
**DO NOT** download vocabularies frin Athena, instead use the provided vocabulary files.
```bash
#TODO Add the vocabulary files to the repository
```
You will need a UMLS API key to perform the CPT code injection. You can get one from [here](https://documentation.uts.nlm.nih.gov/rest/authentication.html).

After you have the API key, you can inject the CPT codes with the following command:
```bash
# In the downloaded vocabulary repository
# Make the script executable
chmod +x cpt.sh
# Run the script with the API key
./cpt.sh <API_KEY>
```
This takes a while (about 1 hour and a half), so be patient.
### 2.2 Prepare Docker container
Pull the image of the PostgreSQL:
```bash
docker pull postgres
```
Run the container with the following command:
```bash
docker run --name <CONTAINER_NAME> -d -p <PORT_ON_HOST_MACHINE>:<PORT_POSTGRES> -e POSTGRES_PASSWORD=<PASSWORD> --restart unless-stopped postgres:latest

# Example
# docker run --name omopv54 -d -p 5432:5432 -e POSTGRES_PASSWORD=mypsw --restart unless-stopped postgres:latest
```

### 2.3 Create new database
Connect to the container with the following command:
```bash
# Connect to the container
docker exec -it <CONTAINER_NAME> 
# Run psql
psql -U postgres
```
Create a new database with the following command:
```sql
CREATE DATABASE <DATABASE_NAME>;
```
*Optional*: You can create a schema for the database with the following command:
```sql
CREATE SCHEMA <SCHEMA_NAME>;
```
If no schema is created, the default schema is `public`.
### 2.4 Get DDLs
**DO NOT** go to the official OHDSI Github repository to download the DDLs, instead use the provided DDLs at `omopv5_4_setup/CommonDataModel-5.4.0/inst/ddl/5.4/postgresql/`.
There're 4 files in the folder:
- `OMOPCDM_postgresql_5.4_constraints.sql`
- `OMOPCDM_postgresql_5.4_ddl.sql`
- `OMOPCDM_postgresql_5.4_indices.sql`
- `OMOPCDM_postgresql_5.4_primary_keys.sql`

In each of them, make sure the schema is correct, and if you have created a schema in the previous step, replace the current schema with the schema name, otherwise make sure the schema is `public`.

### 2.5 Create DDL
Run **ONLY** the `OMOPCDM_postgresql_5.4_ddl.sql` file to create the tables in the database with:
```bash
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f OMOPCDM_postgresql_5.4_ddl.
```
This should create tables in the database `<DATABASE_NAME>`.
You can verify the tables in the database shell with the following command:
```sql
-- Connect to the database
\c <DATABASE_NAME>
-- List all tables
\dt
```

### 2.6 Import Vocabulary
The `sql` file for importing the vocabulary is provided in the `omopv5_4_setup/VocabImport/OMOP CDM vocabulary load - PostgreSQL.sql`. You should modify this file so that the path points to the right location of the vocabulary files generated in step 2.1, e.g.:
```sql
-- Original line:
\copy DRUG_STRENGTH FROM '<your path to vocabularies>/DRUG_STRENGTH.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;

-- Modified line:
\copy DRUG_STRENGTH FROM '/path/to/vocabularies/DRUG_STRENGTH.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;
```

Run the file with the following command:
```bash
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f path/to/OMOP\ CDM\ vocabulary\ load\ -\ PostgreSQL.sql
```

This should output:
```
COPY 2981807
COPY 7407686
COPY 46545874
COPY 80436514
COPY 2741949
COPY 91
COPY 696
COPY 423
COPY 50
```

Verify the vocabulary tables with the following command:
```sql
-- Connect to the database
\c <DATABASE_NAME>

-- List all tables
\dt

-- List Size of the tables
select
  table_name,
  pg_size_pretty(pg_total_relation_size(quote_ident(table_name))),
  pg_total_relation_size(quote_ident(table_name))
from information_schema.tables
where table_schema = 'public' -- Modify this if you have a different schema
order by 3 desc;

-- OR List all tables with their number of live rows
SELECT schemaname,relname,n_live_tup 
  FROM pg_stat_user_tables 
ORDER BY n_live_tup DESC;

-- If all of the tables are empty, something must be wrong you can try to re-import the vocabulary
```

### 2.7 Finish other DDLs
In the `omopv5_4_setup` directory, there are 4 other text files that you should run in the database:
```bash
# Run the following command in order

# person
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f omoponfhir_f_person_ddl.txt

# observation size change
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f observation_column_size_change_ddl.txt

# observation
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f omoponfhir_v5.4_f_observation_ddl.txt

# immunization
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f omoponfhir_v5.2_f_immunization_view_ddl.txt
```

### 2.8 Create CDM primary keys, constraints, and indices
In the `omopv5_4_setup/CommonDataModel-5.4.0/inst/ddl/5.4/postgresql` directory, there are 3 other text files that you should run in the database:
```bash
# Run the following command in order

# primary keys
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f OMOPCDM_postgresql_5.4_primary_keys.sql

# constraints
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f OMOPCDM_postgresql_5.4_constraints.sql
# There are some errors when running the constraints file #TODO fix the constraints file

# indices
psql -h localhost -p <PORT_POSTGRES> -U <POSTGRES_USER_NAME> -W -d <DATABASE_NAME> -f OMOPCDM_postgresql_5.4_indices.sql
```

