# Platform Deployment Manager

* [Design](#design)
* [Building](#building)
* [API Documentation](#api-documentation)
  * [Repository API](#repository-api)
  * [Packages API](#packages-api)
  * [Applications API](#applications-api)
  * [Environment Endpoints API](#environment-endpoints-api)
* [Deployment Manager Variables](#deployment-manager-variables)

# Design #

The Deployment Manager is a service that manages package deployment and application creation for a single PNDA cluster.

- It implements the Packages, Applications, Repository and EnvironmentEndpoints API used by operators.
- It parses and validates basic package structure.
- It interacts with a Repository and a Registrar to determine available & record currently deployed packages and applications.
- It includes a number of component specific Creator implementations that carry out the concrete steps necessary to set up different parts of the Core Platform.
- It is easily extensible to support additional component types and repository types.

The design consists of a main class that implements the APIs and coordinates between a Repository, a Registrar and an Application Creator that dynamically loads a number of component specific Creator classes as required by a particular package.

HTTP and Python bindings are provided for these APIs.

## Connecting

By default, the Deployment Manager is installed on `edge` node o. In order to access it go to: http://[cluster-name]-cdh-edge:5000

## Repository ##

Packages are made available via a repository. The Deployment Manager is configured with a client of this repository at instantiation time. The reference repository is implemented as a thin wrapper over an Openstack Swift container.

## Registrar ##

The details of package deployments for a given service instance are recorded by a registrar. The registrar stores information in HBase in the platform_packages and platform_applications tables.

## Application Creator ##

The Application Creator handles the creation and control of applications on behalf of the Deployment Manager. It implements business logic that is common to all components and delegates to a component specific Creator as required by a particular package. Creator subclasses are dynamically loaded as needed by the Application Creator.

## Creator ##

Each component type is associated with a subclass of Creator. Each Creator implements the specific steps necessary to perform the following functions:

### Validation ###

Each component type has a specific structure. Each Creator implements a validation function that checks that structure. All components are validated before the package is deployed. If any validation function fails, the package is deemed “bad” and package deployment fails. This provides an opportunity to catch simple package construction problems early in the deployment process.

### Application creation ###

Each component type has specific creation requirements and resource dependencies. Each Creator implements the process required to create components of a given type and returns “application_data”. The Deployment Manager aggregates the application data generated by the process of creating each of the components in the package, then persists an association between this and the package deployment using a Registrar.

### Application control ###

Applications may be paused and restarted. This leaves all the installed components in-place and temporarily stops the running processes associated with those components.

### Undeployment ###

Each Creator implements a specific set of steps to uninstall components of its associated type. The Creator is passed the application data associated with the package and component and uses this to execute those steps.


# Requirements

* [Maven](https://maven.apache.org/install.html)

# Building

To build the Deployment Manager, change to the `api` directory, which contains the `pom.xml` file. Type `mvn clean package` on the command line. Once the build is successful, the built package will be placed in the `target` folder.

# API Documentation

* [Repository API](#repository-api)
  * [GET /repository/packages](#list-packages-from-the-repository)
* [Packages API](#packages-api)
  * [GET /packages](#list-packages-currently-deployed-to-the-cluster)
  * [GET /packages/_package_/status](#get-the-status-for-package)
  * [GET /packages/_package_](#get-full-information-for-package)
  * [PUT /packages/_package_](#deploy-package-to-the-cluster)
  * [DELETE /packages/_package_](#undeploy-package-from-the-cluster)
* [Applications API](#applications-api)
  * [GET /applications](#list-all-applications)
  * [GET /packages/_package_/applications](#list-applications-that-have-been-created-from-package)
  * [GET /applications/_application_/status](#get-the-status-for-application)
  * [POST /applications/_application_/start](#start-application)
  * [POST /applications/_application_/stop](#stop-application)
  * [GET /applications/_application_](#get-full-information-for-application)
  * [PUT /applications/_application_](#create-application-from-package)
  * [DELETE /applications/_application_](#destroy-application)
* [Environment Endpoints API](#environment-endpoints-api)
  * [GET /environment/endpoints](#list-environment-variables-known-to-the-deployment-manager)

## Repository API

### List packages from the repository

?recency=n may be used to control how many versions of each package are listed, by default recency=1
````
GET /repository/packages

Response Codes:
200 - OK
500 - Server Error

Example response:
[
    {
	"latest_versions": [{
		"version": "1.0.23",
		"file": "spark-batch-example-app-1.0.23.tar.gz"
	}],
	"name": "spark-batch-example-app"
    }
]
````

## Packages API

### List packages currently deployed to the cluster
````
GET /packages

Response Codes:
200 - OK
500 - Server Error

Example response:
["spark-batch-example-app-1.0.23"]
````

### Get the status for _package_
````
GET /packages/<package>/status

Response Codes:
200 - OK
500 - Server Error

Example response:
{"status": "DEPLOYED", "information": "human readable error message or other information about this status"}

Possible values for status:
NOTDEPLOYED
DEPLOYING
DEPLOYED
UNDEPLOYING
````

### Get full information for _package_
````
GET /packages/<package>

Response Codes:
200 - OK
500 - Server Error

Example response:
{
	"status": "DEPLOYED",
	"version": "1.0.23",
	"name": "spark-batch-example-app",
	"defaults": {
		"oozie": {
			"example": {
				"end": "${deployment_end}",
				"start": "${deployment_start}",
				"driver_mem": "256M",
				"input_data": "/user/pnda/PNDA_datasets/datasets/source=test-src/year=*",
				"executors_num": "2",
				"executors_mem": "256M",
				"freq_in_mins": "180",
				"job_name": "batch_example"
			}
		}
	}
}
````

### Deploy _package_ to the cluster
````
PUT /packages/<package>

Response Codes:
202 - Accepted, poll /packages/<package>/status for status
404 - Package not found in repository
409 - Package already deployed
500 - Server Error
````

### Undeploy _package_ from the cluster
````
DELETE /packages/<package>

Response Codes:
202 - Accepted, poll /packages/<package>/status for status
404 - Package not deployed
500 - Server Error
````

## Applications API

### List all applications
````
GET /applications

Response Codes:
200 - OK
500 - Server Error

Example response:
["spark-batch-example-app-instance"]
````

### List applications that have been created from _package_
````
GET /packages/<package>/applications

Response Codes:
200 - OK
500 - Server Error

Example response:
["spark-batch-example-app-instance"]
````

### Get the status for _application_
````
GET /applications/<application>/status

Response Codes:
200 - OK
404 - Application not known
500 - Server Error

Example response:
{"status": "STARTED", "information": "human readible error message or other information about this status"}

Possible values for status:
NOTCREATED
CREATING
CREATED
STARTING
STARTED
STOPPING
DESTROYING
````

### Get run-time details for _application_
````
GET /applications/<application>/detail

Response Codes:
200 - OK
404 - Application not known
500 - Server Error

{
        "yarn_applications": {
		    "oozie-example": {
			    "type": "oozie",
				"yarn-id": "application_1479988623709_0015",
				"component": "example",
				"yarn-start-time": 1479992520527,
				"yarn-state": "FINISHED"
			}
		},
		"status": "STARTED",
		"name": "spark-batch-example-app-instance"
}

````

### Start _application_
````
POST /applications/<application>/start

Response Codes:
202 - Accepted, poll /applications/<application>/status for status
404 - Application not known
500 - Server Error
````

### Stop _application_
````
POST /applications/<application>/stop

Response Codes:
202 - Accepted, poll /applications/<application>/status for status
404 - Application not known
500 - Server Error
````

### Get full information for _application_
````
GET /applications/<application>

Response Codes:
200 - OK
404 - Application not known
500 - Server Error

Example response:
{
	"status": "CREATED",
	"overrides": {
        "user": "somebody",
		"package_name": "spark-batch-example-app-1.0.23",
		"oozie": {
			"example": {
				"executors_num": "5"
			}
		}
	},
	"package_name": "spark-batch-example-app-1.0.23",
	"name": "spark-batch-example-app-instance",
	"defaults": {
		"oozie": {
			"example": {
				"end": "${deployment_end}",
				"input_data": "/user/pnda/PNDA_datasets/datasets/source=test-src/year=*",
				"driver_mem": "256M",
				"start": "${deployment_start}",
				"executors_num": "2",
				"freq_in_mins": "180",
				"executors_mem": "256M",
				"job_name": "batch_example"
			}
		}
	}
}
````

### Create _application_ from _package_

````
PUT /applications/<application>
{
	"user": "<username>",
	"package": "<package>",
	"<componentType>": {
		"<componentName>": {
			"<property>": "<value>"
		}
	}
}

Response Codes:
202 - Accepted, poll /applications/<application>/status for status
400 - Request body failed validation
404 - Package not found
409 - Application already exists
500 - Server Error

Example body:
{
	"user": "somebody",
	"package": "<package>",
	"oozie": {
		"example": {
			"executors_num": "5"
		}
	}
}

Package and user are mandatory, property settings are optional
````

### Destroy _application_
````
DELETE /applications/<application>

Response Codes:
200 - OK
404 - Application not known
500 - Server Error
````

## Environment Endpoints API
### List environment variables known to the deployment manager
````
GET /environment/endpoints

Response Codes:
200 - OK
500 - Server Error

Example response:
{"zookeeper_port": "2181", "cluster_root_user": "cloud-user", ... }
````
# Deployment Manager Variables #

The following variables are made available for use in the configuration files for every component and injected as previously described.

## Component Variables ##
````
component_user_name     The user ID that this component will run as
component_application   unique application ID
component_name          name of component folder in package
component_job_name      application_id-component_name-job
component_xxx           setting xxx from properties.json
hdfspath_path_name      generated from entries in hdfs.json
````

## Environment Variables ##
These can be obtained with the [environment endpoints API](#environment-endpoints-api)
````
environment_hadoop_manager_host       192.168.1.2
environment_hadoop_manager_password   admin
environment_hadoop_manager_username   admin
environment_cluster_private_key         ./dm.pem
environment_cluster_root_user           cloud-user
environment_hbase_rest_port             20550
environment_hbase_rest_server           cluster-cdh-mgr1
environment_hive_port                   10000
environment_hive_server                 cluster-cdh-mgr1
environment_impala_host                 cluster-cdh-dn0
environment_impala_port                 21050
environment_kafka_brokers               192.168.1.3:9092, ...
environment_kafka_manager               https://192.168.1.4:443
environment_kafka_zookeeper             192.168.1.5:2181, ...
environment_metric_logger_url           hhtp://192.169.1.7:3001/metrics
environment_name_node                   hdfs://cluster-cdh-mgr1:8020
environment_namespace                   platform_app
environment_oozie_uri                   http://cluster-cdh-mgr1:11000/oozie
environment_opentsdb                    192.168.1.6:4242
environment_queue_name                  default
environment_webhdfs_host                cluster-cdh-mgr1
environment_webhdfs_port                50070
environment_yarn_node_managers          cluster-cdh-dn0
environment_yarn_resource_manager_host  cluster-cdh-mgr1
environment_yarn_resource_manager_mr_port 8032
environment_yarn_resource_manager_port  8088
environment_zookeeper_port              2181
environment_zookeeper_quorum            cluster-cdh-mgr1
````

## Oozie Specific Variables ##
The following varibles are only injected for Oozie components.

````
component_end                  2016-03-31T17:07Z
component_start                2016-03-24T17:07Z
mapreduce.job.user.name        hdfs
oozie.coord.application.path   hdfs://cluster-cdh-mgr1:8020/user/application_id/component_name/coordinator.xml
oozie.libpath                  /user/deployment/platform
oozie.use.system.libpath       true
user.name                      hdfs
````
