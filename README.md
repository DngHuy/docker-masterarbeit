# Masterarbeit - Bewertung und Analyse von Softwareartefakten - Stack
Docker Compose to start the respective services. 

<!-- TOC -->
* [Masterarbeit - Bewertung und Analyse von Softwareartefakten - Stack](#masterarbeit---bewertung-und-analyse-von-softwareartefakten---stack)
  * [Local Testing](#local-testing)
  * [Requirements](#requirements)
    * [Host setup](#host-setup)
      * [Windows](#windows)
      * [macOS](#macos)
    * [Bringing up the stack](#bringing-up-the-stack)
    * [Initial setup for Elasticsearch and Kibana](#initial-setup-for-elasticsearch-and-kibana)
      * [Setting up user authentication](#setting-up-user-authentication)
      * [Injecting data](#injecting-data)
    * [Setup for SonarQube](#setup-for-sonarqube)
      * [Generate Access Tokens](#generate-access-tokens)
    * [Setup for Gitlab-Service](#setup-for-gitlab-service)
    * [Setup for Sonar-Service](#setup-for-sonar-service)
      * [Env file](#env-file)
      * [Local analysis](#local-analysis)
      * [Maven](#maven)
    * [Docker Image](#docker-image)
      * [Evaluation for a specific date](#evaluation-for-a-specific-date)
    * [Setup for Eval-Service](#setup-for-eval-service)
      * [Env file](#env-file-1)
      * [Evaluation for a specific date](#evaluation-for-a-specific-date-1)
      * [Configuration](#configuration)
    * [projects/default/project.properties](#projectsdefaultprojectproperties)
    * [projects/default/params](#projectsdefaultparams)
    * [projects/default/metrics](#projectsdefaultmetrics)
    * [projects/default/level2s.properties](#projectsdefaultlevel2sproperties)
    * [projects/default/level3s.properties](#projectsdefaultlevel3sproperties)
  * [Source](#source)
<!-- TOC -->

## Local Testing
It's probably for the best if the sonar-services and eval-service are only run once and stopped afterwards. The services work by defining an interval. After the given duration, the service will fetch or compute on the newest data and then go back to their idle state. The gitlab-service will do fetch the data on startup of the service. To synchronize the real time data with elasticsearch, the webhook in the gitlab-service is listening for events.

## Requirements
### Host setup

* [Docker Engine][docker-install] version **18.06.0** or newer
* [Docker Compose][compose-install] version **1.28.0** or newer (including [Compose V2][compose-v2])
* 1.5 GB of RAM

> [!NOTE]
> Especially on Linux, make sure your user has the [required permissions][linux-postinstall] to interact with the Docker
> daemon.

By default, the stack exposes the following ports:

* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

#### Windows
If you are using the legacy Hyper-V mode of _Docker Desktop for Windows_, ensure [File Sharing][win-filesharing] is
enabled for the `C:` drive.

#### macOS
The default configuration of _Docker Desktop for Mac_ allows mounting files from `/Users/`, `/Volume/`, `/private/`,
`/tmp` and `/var/folders` exclusively. Make sure the repository is cloned in one of those locations or follow the
instructions from the [documentation][mac-filesharing] to add more locations.


### .env file
Here's the whole .env file. Create a `.env` file and copy/past this content. 
```properties
ELASTIC_VERSION=8.12.1

## Passwords for stack users
#

# User 'elastic' (built-in)
#
# Superuser role, full access to cluster management and data indices.
# https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
ELASTIC_PASSWORD='elastic'

# User 'kibana_system' (built-in)
#
# The user Kibana uses to connect and communicate with Elasticsearch.
# https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
KIBANA_SYSTEM_PASSWORD='elastic'

# Users 'metricbeat_internal', 'filebeat_internal' and 'heartbeat_internal' (custom)
#
# The users Beats use to connect and send data to Elasticsearch.
# https://www.elastic.co/guide/en/beats/metricbeat/current/feature-roles.html
METRICBEAT_INTERNAL_PASSWORD=''
FILEBEAT_INTERNAL_PASSWORD=''
HEARTBEAT_INTERNAL_PASSWORD=''

# User 'monitoring_internal' (custom)
#
# The user Metricbeat uses to collect monitoring data from stack components.
# https://www.elastic.co/guide/en/elasticsearch/reference/current/how-monitoring-works.html
MONITORING_INTERNAL_PASSWORD=''

# User 'beats_system' (built-in)
#
# The user the Beats use when storing monitoring information in Elasticsearch.
# https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
BEATS_SYSTEM_PASSWORD=''

# Sonar-Service
SONAR_URL=<url>
SONAR_TOKEN=<user_token>
SONAR_USER=admin
SONAR_PASS=sonar
# comma seperated project_keys
SONAR_COMPONENT_KEYS=<keys>
# comma seperated project_keys
SONAR_PROJECT_KEY=<keys>
# metrics 
SONAR_METRIC_KEYS=comment_lines_density,complexity,violations,duplicated_lines_density,security_review_rating,sqale_index,new_technical_debt,sqale_rating,reliability_rating,classes,functions,test_success_density
SONAR_INTERVAL_SECONDS=10m

# Elasticsearch
ELASTICSEARCH_IP=elasticsearch
ELASTICSEARCH_APIKEY=<api_key>

# Eval-Service
EVAL_INTERVAL=10m

# Gitlab-Service
# GITLAB_TOKEN=796963
GITLAB_URL=<url>
GITLAB_USERNAME=<username>
GITLAB_PASSWORD=<password>
GITLAB_WEBHOOK=<webhook_secret>

# Sonarqube & Database
SONAR_JDBC_PASSWORD=sonar
POSTGRES_PASSWORD=sonar
```

### Bringing up the stack
Clone this repository onto the Docker host that will run the stack with the command below:

```sh
git@github.com:DngHuy/docker-masterarbeit.git
```

Then, initialize the Elasticsearch users and groups required by docker-elk by executing the command:

```sh
docker-compose up setup
```

If everything went well and the setup completed without error, start the other stack components:

```sh
docker-compose up

If everything went well and the setup completed without error, start the other stack components:

```sh
docker-compose up
```

Give Kibana a bit to initialize. The Kibana web UI can be accessed by going to  <http://localhost:5601>, use the following (default) credentials to log in:

* user: *elastic*
* password: *changeme*

> [!NOTE]
> Upon the initial startup, the `elastic` and `kibana_system` Elasticsearch users are intialized
> with the values of the passwords defined in the [`.env`](.env) file (_"changeme"_ by default). The first one is the
> [built-in superuser][builtin-users], the other two is used by Kibana to communicate with
> Elasticsearch. This task is only performed during the _initial_ startup of the stack. To change users' passwords
> _after_ they have been initialized, please refer to the instructions in the next section.

> [!NOTE] You might have to set the time frame in a dashboard to see actual data after inserting them.

### Initial setup for Elasticsearch and Kibana

#### Setting up user authentication

The _"changeme"_ password set by default for all aforementioned users is **unsecure**. For increased security, we will
reset the passwords of all aforementioned Elasticsearch users to random secrets.

1. Reset passwords for default users

   The commands below reset the passwords of the `elastic` and `kibana_system` users. Take note
   of them.

    ```sh
    docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic
    ```

    ```sh
    docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system
    ```

1. Replace usernames and passwords in configuration files

   Replace the value for `ELASTIC_PASSWORD` inside the `.env` file with the password generated in the previous step or delete it after the stack has been initialized.

   Replace the password of the `kibana_system` user inside the `.env` file with the password generated in the previous
   step. Its value is referenced inside the Kibana configuration file (`kibana/config/kibana.yml`).

   See the [Configuration](#configuration) section below for more information about these configuration files.

1. Restart Kibana to re-connect to Elasticsearch using the new passwords

    ```sh
    docker-compose up -d kibana
    ```

#### Injecting data

Open the url <http://localhost:5601> and use the following credentials to log
in:
* user: *elastic*
* password: *\<your generated elastic password>*

Navigate to `Management > Stack Management > Kibana > Saved Objects`.
Click on `Import` and choose the `saved_objects.njson` file from the root directory of this folder. Afterwards all data views, dashboards and visualizations should have been imported.

#### Create the API Key for Elasticsearch
To manage API keys, open the main menu, then click Stack Management > Security > API Keys.

### Setup for SonarQube
The username is defined in the compose file for the respective service. The password for the user is inside the `.env` file.
```properties
SONAR_JDBC_PASSWORD=<password>
POSTGRES_PASSWORD=<password>
```
The following command starts the sonarqube server and the dependent postgresql database.
```sh
docker-compose up -d sonarqube
```

Give Sonarqube about a minute to initialize, then access the Sonarqube web UI by opening <http://localhost:9000> in a web
browser and use the following (default) credentials to log in:

* user: *admin*
* password: *admin*

After entering the (default) credentials you are required to set a new password.

#### Generate Access Tokens
In the web UI navigate to: My Account > Security > Generate Tokens.
Create two tokens: 
- Global Analysis Token - used to create analysis of the project (see [Local analysis](#local-analysis))
- User Token - used to make API requests in the sonar-service (see [Env file](#env-file))

If you directly create a project in the UI, you'll have to set a project key. This project key is required for the eval-service and sonar-service. 
It's defined as `bcKey`, `SONAR_COMPONENT_KEYS` or `SONAR_PROJECT_KEY`.

### Setup for Gitlab-Service
Set the required .env variables in the `.env` file. The `GITLAB_TOKEN` variable is not used yet, the gitlab-service is currently using the user:password to authenticate with Gitlab.

```properties
GITLAB_URL=<url>
GITLAB_USERNAME=<username>
GITLAB_PASSWORD=<password>
GITLAB_WEBHOOK=<webhook_secret>
```

The following command starts the gitlab-service.
```sh
docker-compose up -d gitlab
```

The Gitlab-Service will fetch all all Gitlab to this date and listen to webhook events. Received data will be inserted into elasticsearch.

### Setup for Sonar-Service
This service fetches data from the before initialised sonarqube-container and inserts them into the elasticsearch. The sonar-service periodically fetches data and inserts them into elasticsearch. The interval can be defined with the `SONAR_INTERVAL_SECONDS` property in the `.env` file.

#### Env file
Set the required .env variables in the `.env` file.

```properties
SONAR_URL=<url>
SONAR_TOKEN=<user_token>
SONAR_COMPONENT_KEYS=List of <project_keys> comma seperated 
SONAR_PROJECT_KEY=List of <project_keys> comma seperated 
SONAR_METRIC_KEYS=comment_lines_density,complexity,violations,duplicated_lines_density,security_review_rating,sqale_index,new_technical_debt,sqale_rating,reliability_rating,classes,functions,test_success_density
SONAR_INTERVAL_SECONDS=<number><time_unit> e.g. 10s, 10m, 24h, etc.
```

#### Local analysis
The code analysis can technically run via webhooks, since there are still software and hardware limitations, the option of running a local code analysis is used to push the analysis into the sonarqube-server.

> [Note] You might need to set the permission to run a code analysis or to create a project from below-mentioned commands. 
> The `project_key` can also be chosen here, in case that the project does not exists yet.
> You can create the project in the web UI first. In that case copy the project_key and name into the respective command below.

#### Maven
Go into the folder where the `pom.xml` is.

To include the coverage we are using the `jacoco-maven-plugin`. Add this to the `pom.xml`

```xml
<profile>
  <id>coverage</id>
  <build>
   <plugins>
    <plugin>
      <groupId>org.jacoco</groupId>
     <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.12</version>
      <executions>
        <execution>
          <id>prepare-agent</id>
          <goals>
            <goal>prepare-agent</goal>
          </goals>
        </execution>
        <execution>
          <id>report</id>
          <goals>
            <goal>report</goal>
          </goals>
          <configuration>
            <formats>
              <format>XML</format>
            </formats>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
   </plugins>
  </build>
</profile>

```

Execute the following command: 

```sh 
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=<project_key> \
  -Dsonar.projectName='project_name' \
  -Dsonar.host.url=<url | defaults to http://localhost:9000> \
  -Dsonar.token=<token> -Pcoverage
```

In case the the first command is not working and the configuration is alright, try the second approach. 

### Docker Image

```sh
docker run --network docker-masterarbeit_default  \
  --rm -v "<PATH_TO_FOLDER_WHERE_POM>:/usr/src" \
  sonarsource/sonar-scanner-cli \
  -D sonar.sources=. \
  -D sonar.java.binaries=target \
  -D sonar.projectKey=<project_key> \
  -D sonar.host.url=<url | defaults to http://localhost:9000> \
  -D sonar.token=<token>
```

#### Evaluation for a specific date
By setting the command line argument in the `docker-compose.yml` you can specify a date in which the metrics should be evaluated. Date format is `yyyy-mm-dd`.

```properties  
args:
    evaluationDate: 2024-05-03
```

The following command starts the sonar-service.
```sh
docker-compose up -d sonar
```

### Setup for Eval-Service
This service fetches data from the before initialised elasticsearch-container, calculates the level 1, level 2 and level 3 metrics and inserts them into elasticsearch again.
This service should be run at last after the data from the defined sources are fetched and inserted in to elasticsearch. The eval-service periodically fetches data, calculates and inserts them into elasticsearch. The interval can be defined with the `EVAL_INTERVAL` property in the `.env` file.

#### Env file
Set the required .env variables in the `.env` file.
```properties
EVAL_INTERVAL=<number><time_unit> e.g. 10s, 10m, 24h, etc.
```

#### Evaluation for a specific date
By setting the command line argument in the `docker-compose.yml` you can specify a date in which the metrics should be evaluated. Date format is `yyyy-mm-dd`.

#### Configuration
The Eval-Service is configured via a pair of `.query` and .properties` files. The files are are placed in a specific folder structure. The top-folder is named 'projects'. Each subfolder defines a single project, which contains the metrics which are to be evaluated.

The following folder structure defines one project 'default' (rename it to the project name) with sample metrics.  
To add more projects copy the whole 'default' folder and configure the `project.properties` file and metrics as necessary.

```
+---projects
    +---default
    |   +---level2s
    |   |     level2.properties
    |   |     level2.query
    |   +---level3s
    |   |     level3.properties
    |   |     level3.query
    |   +---metrics
    |   |     comments.properties
    |   |     comments.query
    |   |     complexity.properties
    |   |     complexity.query
    |   |     estimation.properties
    |   |     estimation.query
    |   +---params
    |   |     01_lastSnapshotDate.properties
    |   |     01_lastSnapshotDate.query
    |   |     02_filesInSnapshot.properties
    |   |     02_filesInSnapshot.query
    |   |
    |   |  level2s.properties
    |   |  level3s.properties
    |   |  project.properties
    |   eval.properties
```

### projects/default/project.properties
This property file is a special case and acts as a replacement to then usual `.env` file.  
Replace the value for:
- `project.name` with the actual name of the project
- `sonarqube.measures.bcKey` with the <sonar_project_key>

Error handling takes place when the computation of metrics, level2s, or level3s fails. This can happen because of missing data, errors in formulas (e.g. division by 0) and for other reasons. The onError property allows to set a project-wide default (which can be overwritten for metrics, level2s etc.) how to handle these errors.
+ The 'drop' option just drops the metrics/level2s/level3s item that can't be computed, no record is stored.
+ The 'set0' option stores a record with value 0.

```properties
# project name
# must be lowercase since it becomes part of the metrics/level2s/level3s/relations index names, mandatory, change to the gitlab name, keep the "-"
project.name=default

# Elasticsearch source data, mandatory
elasticsearch.source.ip=elasticsearch

# Elasticsearch target data (metrics, level2s, level3s, relations, ...), mandatory
# Could be same as source
elasticsearch.target.ip=elasticsearch

########################
#### SOURCE INDEXES ####
########################

# sonarqube measures index
sonarqube.measures.index=sonarqube_measures
sonarqube.measures.bcKey=<your-sonarqube-base-component-key>

# sonarqube issues index
sonarqube.issues.index=sonarqube_issues
sonarqube.issues.project=<your-sonarqube-project-key>

########################
#### TARGET INDEXES ####
########################

# rules for index names: lowercase, no special chars (dot "." allowed), no leading numbers, 

# metrics index, mandatory
metrics.index=metrics
metrics.index.type=metrics

# level2s index, mandatory
level2s.index.type=level2s

# impacts index, mandatory
relations.index=relations
relations.index.type=relations

# level2s index, mandatory
level3s.index=level3s
level3s.index.type=level3s

# global error handling default: 'drop' or 'set0', default is 'drop'.
# Error handling takes place when the computation of a metric/level2/level3/relation fails.
# Strategy 'drop' doesn't store the item, 'set0' sets the item's value to 0.
# The setting can be overwritten for specific metrics, level2s, and level3s
onError=set0
```

### projects/default/params
Eval-Service is a tool designed to facilitate project evaluation processes. In the initial phase, it executes queries located in the params folder, specifically referred to as params queries. These queries don't directly calculate metrics or level2s but serve the purpose of querying arbitrary values denoted with the prefix result. These queried values can then be utilized as parameters in subsequent params and metrics queries.
Unlike values found in project.properties, where declaration is mandatory, the results of params queries can be seamlessly employed in subsequent queries without the need for explicit declaration in the associated property files.

The params queries are executed sequentially, following alphabetical order. Therefore, adhering to the suggested naming convention for parameter queries is essential. It's advisable to initiate the query names with a sequence of numbers (e.g., 01_query_name, 02_other_name). This practice ensures proper ordering as params queries typically build upon each other. Maintaining this order is crucial for the coherent execution of project evaluation processes.

A query consists of a pair of files:
* A .properties file, which declares the index the query should run on, as well as parameters and results of the query
* A .query file that contains the actual query in Elasticsearch syntax (see [Elasticsearch DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html))

__Example (01_lastShnapshotDate)__

01_lastSnapshotDate.properties

```properties
index=$$sonarqube.measures.index
param.bcKey=$$sonarqube.measures.bcKey
result.lastSnapshotDate=hits.hits[0]._source.snapshotDate
```
+ The index property is read from the project.properties files ($$-notation).
+ The query uses one parameter (bcKey), which is also read from the project properties file. Parameters of a query are declared with prefix 'param.'
+ The query defines one result (lastSnapshotDate), that is specified as a path within the query result delivered by elasticsearch. Results are declared with prefix 'result.'
  All results computed by params queries can be used as parameters (without declaration) in subsequent params- and metrics queries. Make sure that the names of the results of params queries are unique, otherwise they will get overwritten.

__Query Parameters__

Eval-Service internally uses [Elasticsearch search templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-template.html) to perform *params, metrics*, and other queries. Search templates can receive parameters with the mustache annotation (double curly braces: {{parameter}} ). The parameters are replaced by actual values, before the query is executed. The replacement is done verbatim and doesn't care about data types. Thus, if you want a string parameter, you'll have to add quotes around the parameter yourself (as seen below with the evaluationDate parameter).
+ The evaluationDate is available to all *params* and *metrics* queries without declaration. Eval-Service started without command-line options sets the evaluationDate to the date of today (string, format yyyy-mm-dd).
+ Elements of the *project.properties* can be declared as a parameter with the $$-notation, as seen above (param.bcKey)
+ Literals (numbers and strings) can be used after declaration as parameters (e.g by *param.myThreshold=15*)
+ Results (noted with prefix 'result.') of *params queries* can be used as parameters in succeeding *params* and *metrics* queries without declaration.

01_lastSnapshotDate.query

```
{
	"query": {
		"bool": {
			"must" : [
				{ "term" : { "bcKey" : "{{bcKey}}" } },
				{ "range" : { "snapshotDate" : { "lte" : "{{evaluationDate}}", "format": "yyyy-MM-dd" } } }
      		]
		}
	},
	"size": 1,
	"sort": [
    	{ "snapshotDate": { "order": "desc" 	} }
	]
}
```

The lastSnapshotDate query is a [bool query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html). It defines two conditions that have to evaluate to TRUE for matching documents:
+ The documents must have the supplied parameter {{bcKey}} as value of the field bcKey (match only records of the specified project)
+ The value of the field snapshotDate must be lower or equal to the evaluationDate. The {{evaluationDate}} parameter is available to all queries without declaration and typically contains the todays date in the yyyy-MM-dd format. The evaluationDate can be supplied via command-line (see command-line-options).

The query limits the size of the result to one and sorts in descending order.

Example query result:

```json
{
  "took" : 31,
  "timed_out" : false,
  "_shards" : {...} ,
  "hits" : {
    "total" : 144491,
    "max_score" : null,
    "hits" : [
      {
        "_index" : "sonarqube_measures",
        "_type" : "sonarqube",
        "_id" : "sonarqube_measures_1",
        "_score" : null,
        "_source" : {
          ...
          "snapshotDate" : "2024-05-03",
          "bcKey" : "bcKey",
          ...
        },
        "sort" : [
          1543881600000
        ]
      }
    ]
  }
}
```

The result of the query is specified as path in the returned json: __"hits" -> "hits" [0] -> "_source" -> "snapshotDate" = "2024-05-03"__

### projects/default/metrics
Within the folder, you'll find the metrics definitions of a project. Similar to params queries, metrics queries consist of a pair of files, namely a .properties file and a .query file. However, metrics queries go beyond params queries by computing a metric value defined by a specific formula.

After execution, the computed metric value is stored in the metrics index, as defined in the project.properties file. These computed metrics are then aggregated into level2s. To ensure accurate computation, it's imperative to specify the level2s that a metric is expected to influence.

Each metric can influence one or more level2s, which are indicated as a comma-separated list of level2-ids along with the weight that describes the strength of the influence. For instance, in the example provided below, the metric 'complexity' influences two level2s (codequality and other) with weights of 2.0 for codequality and 1.0 for other. The value of a level2 is subsequently computed as a weighted sum of all metrics influencing that particular level2.

__Example: complexity query__

complexity.properties

```properties
# index the query runs on, mandatory
# values starting with $$ are looked up in project.properties
index=$$sonarqube.measures.index

# metric props, mandatory
enabled=true
name=Complexity
description=Percentage of files that do not exceed a defined average complexity per function
level2s=codequality,other
weights=2.0,1.0

# query parameter
param.bcKey=$$sonarqube.measures.bcKey
param.avgcplx.threshold=15

# query results (can be used in metric calculation)
result.complexity.good=aggregations.goodBad.buckets[0].doc_count
result.complexity.bad=aggregations.goodBad.buckets[1].doc_count

# metric defines a formula based on execution results of params- and metrics-queries
metric=complexity.good / ( complexity.good + complexity.bad )
onError=set0
```

__Note:__ The onError property can be set to 'drop' or 'set0' and overwrites to setting in project.properties.

complextiy.query

```
{ 
  "size" : 0,
  "query": {
    "bool": {
      "must" : [
			{ "term" : { "bcKey" : "{{bcKey}}" } },
			{ "term" : { "snapshotDate" : "{{lastSnapshotDate}}" } },
			{ "term" : { "metric" : "function_complexity"} },
			{ "term" : { "qualifier" : "FIL" } }
      ]
    }
  },
  "aggs": {
    "goodBad" : {
      "range" : {
        "field" : "floatvalue",
        "ranges" : [
          { "to" : {{avgcplx.threshold}} }, 
          { "from" : {{avgcplx.threshold}} }
        ]
      }
    }
  }
}
```

The complexity query is based on a [bool query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) and uses a [bucket range aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-range-aggregation.html) to derive its results.
The query considers documents/records that fulfill the following conditions:
+ Only documents with a specific {{bcKey}} (only files of this project)
+ Only documents with a specific {{snapshotDate}} (parameter derived in *params query* 01_snapshotDate)
+ Only documents for metric "function_complexity"
+ Only documents with qualifier "FIL" (analyze only files, not folders etc.)

In the bucket range aggregation, the matching documents are divided into two buckets:
+ Files with function_complexity < avgcplx.threshold
+ Files with function_complexity >= avgcplx.threshold

__Example result__

```
{
  "took" : 22,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : {
    "total" : 53,
    ...
  },
  "aggregations" : {
    "goodBad" : {
      "buckets" : [
        { "key" : "*-15.0", "to" : 15.0, "doc_count" : 53 },
        { "key" : "15.0-*", "from" : 15.0, "doc_count" : 0 }
      ]
    }
  }
}
```

The metric (percentage of files having tolerable complexity) is then computed as:

```
metric=complexity.good / ( complexity.good + complexity.bad ) = 53 / ( 53 + 0 ) = 100%
```

### projects/default/level2s.properties
The level2s.properties file serves to define level2s for computation, along with their respective properties. Unlike performing complex calculations, level2s primarily function as aggregation points for metric values.
As level2s are subsequently aggregated into level3s, it's necessary to specify the level3s they influence, along with the weights of their influence. This specification is denoted in the format factorid.property=value. This notation ensures clarity and consistency in understanding the influence of level2s on level3s.

+ The *enabled* attribute enables/disables a level2 (no records written for a level2 when disabled)
+ The *name* property supplies a user-friendly name of a level2
+ The *decription* attribute describes the intention of the level2
+ The *level3s* attribute contains a list of influenced level3s (which are defined in a separate properties file).
+ The *weights* attribute sets the strength of the influence. Obviously, the lists in 'level3s' and 'weights' have to have the same length!
+ The *onError* attribute tells Eval-Service what to do in case of level2 computation errors (e.g. no metrics influence a level2, which results in a division by zero)

Example level2 definition (codequality):

```properties
codequality.enabled=true
codequality.name=Sprint Score
codequality.description=It measures how well the sprint is going. Specifically, ...
codequality.level3s=productquality
codequality.weights=1.0
codequality.onError=set0
```

>[Note] The onError property can be set to 'drop' or 'set0' and overwrites to setting in project.properties.

### projects/default/level3s.properties
The level3s.properties file defines the level3s for a project. The parents- and weights-attribute currently have no effect, but could define an additional level of aggregation in future.

```properties
productquality.enabled=true
productquality.name=Product Quality
productquality.description=Quality of the Product
productquality.parents=meta

### projects/default/level2s
Defines the query for aggregation of metrics into level2s, based on relations index.

### projects/default/level3s
Defines the query for aggregation of level2s into level3s, based on relations index. 

```properties  
args:
    evaluationDate: 2024-05-03
```

## Source
[Elasticsearch API Keys](https://www.elastic.co/guide/en/kibana/current/api-keys.html)
[SonarQube Java Test Coverage](https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/test-coverage/java-test-coverage/)
[Q-Rapids-Eval](https://github.com/q-rapids/qrapids-eval)
[Docker-ELK](https://github.com/deviantony/docker-elk)
