# Welcome to OMOPonFHIR Github Repository

This repository is an installable server deployment package that locks down the version of included components (using submodules). Please use the following step to compile and deploy the server. You may need to customize some of the environment variables. 

**Note:** If you want to see current snapshots of OMOPonFHIR components or other versions of FHIR and OMOP, please visit the OMOPonFHIR GitHub organization at https://github.com/omoponfhir/

This package is tested with Google Big Query instance. And, the server supports mapping for FHIR R4 and OMOP v5.4. Please see omopv5_4_setup/ folder for some help on ddls for extra tables/views. Database ddls are also included. 

```
**Note:** This repository contains submodules of anoter repositories that are needed. If you want to participate in 
development and contribute, please use the repositories directly as this submodule points to a certain commit point. 
Refer the follow repositories and use the latest to work on the code.

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
        
## Download the package
```
git clone --recurse https://github.com/omoponfhir/omoponfhir-main-v54-r4.git
cd omoponfhir-main-v54-r4/
mvn clean install
cd omoponfhir-r4-server/target/
scp omoponfhir-r4-server.war <vm url>:omoponfhir-r4-server.war
```

## Prepare your OMOP database

If you don't have a OMOP database yet, you will need to create one. In case you already have one you will need to add additional tables for it to work with this tool. For both these cases you can use https://github.com/omoponfhir/omopv5_4_setup as a reference and start at either step 1 or 6.

## Setting up vocabulary schema 

If you want to setup (or have) vocabulary schema, you can make OMOPonFHIR to point to the vocabulary schema for all vocabulary tables. In the vocabulary schema, the tables defined here must exist (https://github.com/SmartChartSuite/Registry-Database/blob/main/omoponfhir_v5_4_setup/OMOPCDM_postgresql_5.4_vocabulary_ONLY_ddl.sql) 

With this setting, you can run multiple OMOPonFHIR instances with the same vocabulary schema. This makes the vocabulary management easier. Please note that All schemas must exist in the same database. 

Once the schemas are set up, you need the following environment variables to specify schemas for data and vocabulary.

```
export JDBC_DATA_SCHEMA="omopv54"
export JDBC_VOCABS_SCHEMA="vocab"
```

The variables above assume that vocabulary related tables are in "vocab" schema, and data are in "omopv54" schema.  

## Deploy to tomcat ##
```
cp omoponfhir-r4-server.war <tomcat_directory>/webapps/
cd <tomcat_directory>/bin
vi setenv.sh
```
## Deploy using Docker
```
sudo docker build -t omoponfhir .
sudo docker run --name omoponfhir -p 8080:8080 -d omoponfhir:latest
```

OMOPonFHIR is a web application. If it's deployed using Docker, then you need to get how war file is deployed in the dockerfile. Default is set to ROOT.war, which means it's running at the / path. If you are running omoponfhir on your local PC, you can use your browser with URL, http://localhost:8080/, to launch test overlay HAPI FHIR GUI. Please note that the port number depends one the docker run command above. If you are running omoponfhir on the remote server, then you need to use the remote server's URL. In both cases, please make sure your firewall is not blocking the traffic.

OMOPonFHIR provides FHIR APIs and operations. The HAPI FHIR GUI is a FHIR client running separately. So, even if this is not running correctly, the services may be running. The FHIR API URL for OMOPonFHIR http://localhost(or server hostname):8080/fhir. You can do GET http://localhost(or server hostname):8080/fhir/metadata to see if OMOPonFHIR is running OK. In most cases, if services are not running, it's because of the database connection and its credential. Please check this in your environment variable. 

## Configuration of webapp
In setenv.sh file, add the following environment variables. Change the values for your environment 
```
export JDBC_URL="jdbc:postgresql://url"
export JDBC_USERNAME="<your username of JDBC DB instance>"
export JDBC_PASSWORD="<your password of JDBC DB instance>"
# export JDBC_DRIVER="org.postgresql.Driver"
export JDBC_DATASOURCENAME="org.postgresql.ds.PGSimpleDataSource"
export JDBC_POOLSIZE=5
# And, belows are schema naems for data and vocabulary in the database specified above.
export JDBC_DATA_SCHEMA="public"
export JDBC_VOCABS_SCHEMA="public"
export SMART_INTROSPECTURL="<your_omoponfhir_root_server_NOT_a_fhir_url_base, eg: localhost:8080/omoponfhir-dstu2-server/>/smart/introspect"
export SMART_AUTHSERVERURL="<your_omoponfhir_root_server_NOT_a_fhir_url_base>/smart/authorize"
export SMART_TOKENSERVERURL="<your_omoponfhir_root_server_NOT_a_fhir_url_base>/smart/token"
export AUTH_BEARER="<any value>"
export AUTH_BASIC="<username_you_want>:<password_you_want>"
export FHIR_READONLY="<True or False>"
export SERVERBASE_URL="<your fhir base url, eg: http://localhost:8080/omoponfhir-dstu2-server/fhir/>"
export LOCAL_CODEMAPPING_FILE_PATH="<whatever the path you want to put your local mapping file. eg: /temp/my_local_code or none>"
export MEDICATION_TYPE="code"
export TARGETDATABASE="<SqlRenderTargetDialect value such as bigquery or postgresql. If you leave this empty, it will be postgresql. Use string from sqlRender>"
export OMOPONFHIR_NAME="OMOP v5.4 on FHIR R4"
export BIGQUERYDATASET="<BigQuery Dataset Name. It will be ignored if TARGETDATABASE is not bigquery>"
export BIGQUERYPROJECT="<BigQuery Project Name. It will be ignored if TARGETDATABASE is not bigquery>"
```

## Configuration of Docker deployment
This is not yet tested and validated. The same environment variables are applied here. The environment variable must be defined either using Dockerfile or docker command line when the docker image is instantiated to start running. One way to do this would be to create a env.list file containing the variables from above and then running the following command to start your deployment:
```
docker run --env-file env.list --name omoponfhir -p 8080:8080 -d omoponfhir:latest
```
