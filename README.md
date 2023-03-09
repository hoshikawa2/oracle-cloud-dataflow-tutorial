# Process large files in Autonomous Database and Kafka with Oracle Cloud Infrastructure Data Flow

## Introduction

Oracle Cloud Infrastructure (OCI) Data Flow is a fully managed service for running Apache Spark ™ applications. Data Flow is used for processing large files, streaming, database operations, and you can build a lot of applications with very high scalable processing. Apache Spark can scale and use clustered machines to parallelize jobs with minimum configuration.

Using Apache Spark as a managed service (Data Flow), you can add many scalable services to multiply the power of cloud processing and this tutorial shows you how to use:

- Object Storage: As low-cost and scalable a file repository
- Autonomous Database: As a scalable Database in the cloud
- Streaming: As a high scalable Kafka managed service

![dataflow-use-case.png](./images/dataflow-use-case.png?raw=true)

In this tutorial, you can see the most common activities used to process large files, querying database and merge/join the data to form another table in memory. You can write this massive data into your database and in a Kafka queue with very low-cost and high performance.

## Objectives

- Learn how Data Flow can be used to process a large amount of data
- Learn how to integrate scalable services: File Repository, Database and Queue

## Prerequisites

- An operational **Oracle Cloud** tenant: You can create a free Oracle Cloud account with US$ 300.00 for a month to try this tutorial. See [Create a Free Oracle Cloud Account](https://www.oracle.com/cloud/free/)

- **OCI CLI** (Oracle Cloud Command Line Interface) installed on your local machine: This is the link to install the [OCI CLI](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm).

- An **Apache Spark** application installed in your local machine. Review [Develop Oracle Cloud Infrastructure Data Flow Applications Locally, Deploy to The Cloud](https://docs.public.oneportal.content.oci.oraclecloud.com/en-us/iaas/data-flow/data-flow-tutorial/develop-apps-locally/front.htm) to understand how to develop locally and in Data Flow.

  >**Note**: This is the official page to install: [Apache Spark](https://spark.apache.org/downloads.html). There are alternative procedures to install Apache Spark for each type of Operational System (Linux/Mac OS/Windows).

- **Spark Submit CLI** installed. This is the link to install [Spark Submit CLI](https://docs.oracle.com/en-us/iaas/data-flow/data-flow-tutorial/spark-submit-cli/front.htm#front).

- **Maven** installed in your local machine.

- Knowledge of OCI Concepts:
  - Compartments
  - IAM Policies
  - Tenancy
  - OCID of your resources

## Task 1: Create the Object Storage structure

The Object Storage will be used as a default file repository. You can use other type of file repositories, but Object Storage is a simple and low-cost way to manipulate files with performance. In this tutorial, both applications will load a large CSV file from the object storage, showing how Apache Spark is fast and smart to process a high volume of data.

1. Create a compartment: Compartments are important to organize and isolate your cloud resources. You can isolate your resources by IAM Policies.

   - You can use this link to understand and setup the policies for compartments: [Managing Compartments](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcompartments.htm)

   - Create one compartment to host all resources of the 2 applications in this tutorial. Create a compartment named **analytics**.

   - Go to the Oracle Cloud main menu and search for: **Identity & Security**, **Compartments**. In the Compartments section, click **Create Compartment** and enter the name.

     ![create-compartment.png](./images/create-compartment.png?raw=true)

     >**Note**: You need to give the access to a group of users and include your user.

    - Click **Create Compartment** to include your compartment.

2. Create your bucket in the Object Storage: Buckets are logical containers for storing objects, so all files used for this demo will be stored in this bucket.

    - Go to the Oracle Cloud main menu and search for **Storage** and **Buckets**. In the Buckets section, select your compartment (analytics), created previously.

      ![select-compartment.png](./images/select-compartment.png?raw=true)

    - Click **Create Bucket**. Create 4 buckets: apps, data, dataflow-logs, Wallet

      ![create-bucket.png](./images/create-bucket.png?raw=true)

    - Enter the **Bucket Name** information with these 4 buckets and maintain the other parameters with the default selection.

    - For each bucket, click **Create**. You can see your buckets created.

      ![buckets-dataflow.png](./images/buckets-dataflow.png?raw=true)

>**Note:** Review the IAM Policies for the bucket. You must set up the policies if you want to use these buckets in your demo applications. You can review the concepts and setup here [Overview of Object Storage](https://docs.oracle.com/en-us/iaas/Content/Object/Concepts/objectstorageoverview.htm) and [IAM Policies](https://docs.oracle.com/en-us/iaas/Content/Security/Reference/objectstorage_security.htm#iam-policies).

## Task 2: Create the Autonomous Database

Oracle Cloud Autonomous Database is a managed service for the Oracle Database. For this tutorial, the applications will connect to the database through a Wallet for security reasons.

- Instantiate the Autonomous Database as described here: [Provision Autonomous Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/adbsa/autonomous-provision.html#GUID-0B230036-0A05-4CA3-AF9D-97A255AE0C08).

- From the Oracle Cloud main menu, select the **Data Warehouse** option, select **Oracle Database** and **Autonomous Data Warehouse**; select your compartment **analytics** and follow the tutorial to create the database instance.

- Name your instance **Processed Logs**, choose **logs** as the database name and you don't need to change any code in the applications.

- Enter the ADMIN password and download the Wallet zip file.

   ![create-adw.png](./images/create-adw.png?raw=true)

- After creating the database, you can setup the **ADMIN** user password and download the **Wallet** zip file.

   ![adw-admin-password.png](./images/adw-admin-password.png?raw=true)

   ![adw-connection.png](./images/adw-connection.png?raw=true)

   ![adw-wallet.png](./images/adw-wallet.png?raw=true)

- Save your Wallet zip file (`Wallet_logs.zip`) and annotate your **ADMIN** password, you will need to setup the application code.

- Go to **Storage**, **Buckets**. Change to **analytics** compartment and you will see the **Wallet** bucket. Click on it.

- To upload your Wallet zip file, just click **Upload** and attach the **Wallet_logs.zip** file.

   ![upload-wallet.png](./images/upload-wallet.png?raw=true)

>**Note:** Review IAM Policies for accessing the Autonomous Database here: [IAM Policy for Autonomous Database](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm#db-admins-manage-adb)

## Task 3: Upload the CSV sample files

To demonstrate the power of Apache Spark, the applications will read a CSV file with 1,000,000 lines. This data will be inserted in the Autonomous Data Warehouse database with just one command line and published on a Kafka streaming (Oracle Cloud Streaming). All these resources are scalable and perfect for high data volume.

- Download these 2 links and Upload to the **data** bucket:

  - [organizations.csv](./files/organizations.csv)

  - [organizations1M.csv](https://objectstorage.us-ashburn-1.oraclecloud.com/p/EqwMnzRwjtmes4okPLItLTOSNyq-KW9ktV5s1n4WCS_y1XpCWXDwJCkZ-PYKJeK0/n/idavixsf5sbx/b/data/o/organizations1M.csv)

    >**Note**:
    > - **organizations.csv** has only 100 lines, just to test the applications on your local machine.
    > - **organizations1M.csv** contains 1,000,000 lines and will be used to run on the Data Flow instance.

- From the Oracle Cloud main menu, go to **Storage** and **Buckets**. Click on the **data** bucket and upload the 2 files from the previous step.

   ![data-bucket-header.png](./images/data-bucket-header.png?raw=true)

   ![csv-files-data-bucket.png](./images/csv-files-data-bucket.png?raw=true)

- **Upload an auxiliary table to ADW Database**

   - Download this file to upload to the ADW Database: [GDP PER CAPTA COUNTRY.csv](./files/GDP_PER_CAPTA_COUNTRY.csv)

  - From the Oracle Cloud main menu, select **Oracle Database** and **Autonomous Data Warehouse**.

  - Click on the **Processed Logs** Instance to view the details.

  - Click **Database actions** to go to the database utilities.

    ![adw-actions.png](./images/adw-actions.png?raw=true)

  - Enter your credentials for the **ADMIN** user.

     ![adw-login.png](./images/adw-login.png?raw=true)

  - Click on the **SQL** option to go to the Query Utilities.

     ![adw-select-sql.png](./images/adw-select-sql.png?raw=true)

  - Click **Data Load**.

    ![adw-data-load.png](./images/adw-data-load.png?raw=true)

  - Drop the **GDP PER CAPTA COUNTRY.csv** file into the console panel and proceed to import the data into a table.

    ![adw-drag-file.png](./images/adw-drag-file.png?raw=true)

You can see your new table named **GDPPERCAPTA** imported successfully.

![adw-table-imported.png](./images/adw-table-imported.png?raw=true)


## Task 4: Create a Secret Vault for your ADW ADMIN password

For security reasons, the ADW ADMIN password will be saved on a Vault. Oracle Cloud Vault can host this password with security and can be accessed on your application with OCI Authentication.

 - Create your secret in a vault as described in the following documentation: [Add the database admin password to Vault](https://docs.oracle.com/en/learn/data-flow-analyze-logs/index.html#add-the-database-admin-password-to-vault)

- Create a variable named **PASSWORD_SECRET_OCID** in your applications and enter the OCID.

  ![vault-adw.png](./images/vault-adw.png?raw=true)

  ![vault-adw-detail.png](./images/vault-adw-detail.png?raw=true)

  ![secret-adw.png](./images/secret-adw.png?raw=true)

>**Note:** Review the IAM Policy for OCI Vault here: [OCI Vault IAM Policy](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm#sec-admins-manage-vaults-keys).

## Task 5: Create a Kafka Streaming with Oracle Cloud Streaming service

Oracle Cloud Streaming is a Kafka like managed streaming service. You can develop applications using the Kafka APIs and common SDKs. In this tutorial, you will create an instance of Streaming and configure it to execute in both applications to publish and consume a high volume of data.

1. From the Oracle Cloud main menu, go to **Analytics & AI**, **Streams**.

2. Change the compartment to **analytics**. Every resource in this demo will be created on this compartment. This is more secure and easy to control IAM.

3. Click **Create Stream**.

   ![create-stream.png](./images/create-stream.png?raw=true)

4. Enter the name as **kafka_like** (for example) and you can maintain all other parameters with the default values.

   ![save-create-stream.png](./images/save-create-stream.png?raw=true)

5. Click **Create** to initialize the instance.

6. Wait for the **Active** status. Now you can use the instance.

   >**Note:** In the streaming creation process, you can select the **Auto-Create a default stream pool** option to automatically create your default pool.

7. Click on the **DefaultPool** link.

   ![default-pool-option.png](./images/default-pool-option.png?raw=true)

8. View the connection setting.

   ![stream-conn-settings.png](./images/stream-conn-settings.png?raw=true)

   ![kafka-conn.png](./images/kafka-conn.png?raw=true)

9. Annotate this information as you will need it in the next step.

>**Note:** Review the IAM Policies for the OCI Streaming here: [IAM Policy for OCI Streaming](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm#streaming-manage-streams).

## Task 6: Generate a AUTH TOKEN to access Kafka

You can access OCI Streaming (Kafka API) and other resources in Oracle Cloud with an Auth Token associated to your user on OCI IAM. In Kafka Connection Settings, the SASL Connection Strings has a parameter named **password** and an **AUTH_TOKEN** value as described in the previous task. To enable access to OCI Streaming, you need to go to your user on OCI Console and create an AUTH TOKEN.

1. From the Oracle Cloud main menu, go to **Identity & Security**, **Users**.

    >**Note**: Remember that the user you need to create the **AUTH TOKEN** is the user configured with your **OCI CLI** and all the **IAM Policies** configuration for the resources created until now. The resources are:
    >  - Oracle Cloud Autonomous Data Warehouse
    >  - Oracle Cloud Streaming
    >  - Oracle Object Storage
    >  - Oracle Data Flow

2. Click on your username to view the details.

   ![auth_token_create.png](./images/auth_token_create.png?raw=true)

3. Click on the **Auth Tokens** option in the left side of the console and click **Generate Token**.

   >**Note**: The token will be generated only in this step and will not be visible after you complete the step. So, copy the value and save it. If you lose the token value, you must generate the auth token again.

   ![auth_token_1.png](./images/auth_token_1.png?raw=true)

   ![auth_token_2.png](./images/auth_token_2.png?raw=true)


## Task 7: Set up the Demo applications

This tutorial has 2 demo applications for which we will set up the required information:

- **Java-CSV-DB:** This application will read 1,000,000 lines of a csv file (organizations1M.csv) and execute some usual processes in a common scenario for integration with a database (Oracle Cloud Autonomous Data Warehouse) and a Kafka streaming (Oracle Cloud Streaming).

   The demo shows how a CSV dataset can be merged with a auxiliary table in database and crossing types of tables generating a third dataset in memory. After the execution, the dataset will be inserted on ADW and published on Kafka streaming.

- **JavaConsumeKafka:** This application will repeat some steps of the first application just to take CPU and memory for a high volume of processing. The difference is, the first application publishes to the Kafka streaming, whereas this application reads from the Streaming.

1. Download the applications using the following links:

   - [Java-CSV-DB.zip](./files/Java-CSV-DB.zip)

   - [JavaConsumeKafka.zip](./files/JavaConsumeKafka.zip)

2. Find the following details in your Oracle Cloud Console:

   - Tenancy Namespace

     ![tenancy-namespace-1.png](./images/tenancy-namespace-1.png?raw=true)

     ![tenancy-namespace-detail.png](./images/tenancy-namespace-detail.png?raw=true)

   - Password Secret

     ![vault-adw.png](./images/vault-adw.png?raw=true)

     ![vault-adw-detail.png](./images/vault-adw-detail.png?raw=true)

     ![secret-adw.png](./images/secret-adw.png?raw=true)

   - Streaming Connection Settings

      ![kafka-conn.png](./images/kafka-conn.png?raw=true)

   - Auth Token

     ![auth_token_create.png](./images/auth_token_create.png?raw=true)

     ![auth_token_2.png](./images/auth_token_2.png?raw=true)

3. Open the downloaded zip files (`Java-CSV-DB.zip` and `JavaConsumeKafka.zip`). Go to the **/src/main/java/example** folder and find the **Example.java** code.

    ![code-variables.png](./images/code-variables.png?raw=true)

    - These are the variables that need to be changed with your tenancy resources values.

      |VARIABLE NAME| RESOURCE NAME| INFORMATION TITLE|
      |-----|----|----|
      |NAMESPACE|TENANCY NAMESPACE|TENANCY|
      |OBJECT_STORAGE_NAMESPACE|TENANCY NAMESPACE|TENANCY|
      |PASSWORD_SECRET_OCID|PASSWORD_SECRET_OCID|OCID|
      |streamPoolId|Streaming Connection Settings|ocid1.streampool.oc1.iad..... value in SASL Connection String|
      |kafkaUsername|Streaming Connection Settings|value of usename inside " " in SASL Connection String|
      |kafkaPassword|Auth Token|The value is displayed only in the creation step|

>**Note:** All the resources created for this tutorial are in the US-ASHBURN-1 region. Check in what region you want to work. If you change the region, you need to change the following details in the 2 code files:
>
> - **Example.java**: Change the **bootstrapServers** variable, replacing the "us-ashburn-1" with your new region.
>
>
> - **OboTokenClientConfigurator.java**: Change the **CANONICAL_REGION_NAME** variable with your new region.

## Task 8: Understand the Java code

This tutorial was created in Java and this code can be ported to Python also. The tutorial is divided in 2 parts:

  - Application 1 to publish to Kafka Streaming

  - Application 2 to consume from Kafka Streaming

To prove the efficiency and scalability, both applications were developed to show some possibilities in a common use case of an integration process. So the code for both the applications show the following examples:

  - Read a CSV file with 1,000,000 lines

  - Prepare the ADW Wallet to connect through a JDBC Connection

  - Insert 1,000,000 lines of CSV data into the ADW database

  - Execute a SQL sentence to query an ADW table

  - Execute a SQL sentence to JOIN a CSV dataset with an ADW dataset table

  - Perform a loop of the CSV dataset to demonstrate an iteration with the data

  - Operate with Kafka Streaming

This demo can be executed in your local machine and can be deployed into the Data Flow instance to run as a job execution.

>**Note:** For both the Data Flow job and your local machine, use the OCI CLI configuration to access the OCI resources. In the Data Flow side, everything is pre-configured, so no need to change the parameters. In your local machine side, install the OCI CLI and configure the tenant, user and private key to access your OCI resources.

Let's show the `Example.java` code in sections:

- Apache Spark initialization: This part of the code represents Spark initialization. Most configurations to perform the execution processes are configured automatically, so it's very easy to work with the Spark engine.

   ![Spark-initilization-code](./images/spark-initialization-code.png?raw=true)

- Read a large file in many formats: The Apache Spark engine and SDK permit a fast load and write file formats. A high volume can be manipulated in seconds and even milliseconds. So you can MERGE, FILTER, JOIN datasets in memory and manipulate different data sources.

   ![read-csv-file-spark.png](./images/read-csv-file-spark.png?raw=true)

- Read the ADW Vault Secret: This part of the code accesses your vault to obtain the secret for your ADW instance.

  ![spark-adw-vault-secret.png](./images/spark-adw-vault-secret.png?raw=true)

- Read the `Wallet.zip` file to connect through JDBC: This section shows how to load the `Wallet.zip` file from Object Storage and configure the JDBC driver.

  ![spark-jdbc-connection.png](./images/spark-jdbc-connection.png?raw=true)

  ![spark-adw-get-secret.png](./images/spark-adw-get-secret.png?raw=true)

- Insert 1,000,000 lines of CSV dataset into ADW Database: From the CSV dataset, it is possible to batch insert into the ADW Database directly. Apache Spark can optimize the execution using all the power of machines clustered, CPUs and memory to obtain the best performance.

   ![spark-insert-adw.png](./images/spark-insert-adw.png?raw=true)

- Data Transformation: Imagine loading many CSVs files, querying some tables in the database in datasets, JOIN, filter, eliminate columns, calculate and many other operations in a few code lines, in a fraction of time and perform a write operation in any format. In this example, a new dataset named **oracleDF2** was created from a CSV dataset and an ADW Database dataset.

  ![spark-transform-datasets.png](./images/spark-transform-datasets.png?raw=true)

- Iterate with a dataset in a loop: This is an example of a loop iteration over the CSV dataset (1,000,000 lines). The **row** object contains the mapping of the CSV fields structure. So you can obtain the data of each line and can execute API calls and many other operations.

  ![spark-iteration.png](./images/spark-iteration.png?raw=true)

- Kafka Operations: This is the preparation for connecting to OCI Streaming using the Kafka API.

  >**Note:** Oracle Cloud Streaming is compatible with most Kafka APIs.

  ![kafka-connection.png](./images/kafka-connection.png?raw=true)

- After configuring the connection parameters, the code shows how to produce and consume the streaming.

  ![kafka-produce.png](./images/kafka-produce.png?raw=true)

  ![kafka-consume.png](./images/kafka-consume.png?raw=true)


## Task 9: Package your application with Maven

Before executing the job in Apache Spark, it is necessary to package your application with Maven. Maven is one of the most known utilities to package applications with libraries and plugins.

>**Note:**
>  - You can execute a fast test changing the CSV file with another with only 100 lines. To do this, just locate the following code in the **Example.java** file: **private static String INPUT_PATH = "oci://data@" + OBJECT_STORAGE_NAMESPACE + "/organizations1M.csv";**
>
>  - Replace `organizations1M.csv` with `organizations.csv` and the execution will be significantly faster.


1. **Java-CSV-DB Package**

   1. Go to **/Java-CSV-DB** folder and execute this command:

      `mvn package`

   2. You can see **Maven** starting the packaging.

      ![maven-package-1a.png](./images/maven-package-1a.png?raw=true)

   3. If everything is correct, you can see the **Success** message.

      ![maven-success-1a.png](./images/maven-success-1a.png?raw=true)

   4. To test your application in your local Apache Spark machine, execute this command:

       `spark-submit --class example.Example target/loadadw-1.0-SNAPSHOT.jar`

2. **JavaConsumeKafka Package**

   1. Go to the **/JavaConsumeKafka** folder and execute this command:

      `mvn package`

   2. You can see **Maven** starting the packaging.

      ![maven-package-2a.png](./images/maven-package-2a.png?raw=true)

   3. If everything is correct, you can see the **Success** message.

      ![maven-success-2a.png](./images/maven-success-2a.png?raw=true)

   4. To test your application in yourr local Apache Spark machine, execute this command:

       `spark-submit --class example.Example target/loadkafka-1.0-SNAPSHOT.jar`

## Task 10: Verify the execution

1. **Confirm ADW Insertions**

   1. Go to the Oracle Cloud main menu, select **Oracle Database** and **Autonomous Data Warehouse**.

   2. Click on the **Processed Logs** Instance to view the details.

   3. Click **Database actions** to go to the database utilities.

      ![adw-actions.png](./images/adw-actions.png?raw=true)

   4. Enter your credentials for the **ADMIN** user.

      ![adw-login.png](./images/adw-login.png?raw=true)

   5. Click on the **SQL** option to go to the Query Utilities.

      ![adw-select-sql.png](./images/adw-select-sql.png?raw=true)

   6. Execute a query to see the 1,000,000 of lines in your table.

      ![ADW-query-organizations.png](./images/ADW-query-organizations.png?raw=true)

2. **Confirm Execution Logs**

   - You can see in the execution logs if the job can access and load the datasets.

     ![spark-csv-results.png](./images/spark-csv-results.png?raw=true)

## Task 11: Create and execute a Data Flow job

Now, with both applications running successfully in your local Apache Spark machine, you can deploy them into the **Oracle Cloud Data Flow** in your tenancy.


1. From the Oracle Cloud main menu, go to **Analytics & AI** and **Data Flow**.

2. Be sure to select your **analytics** compartment before create a Data Flow Application.

3. Click **Create application**.

   ![create-dataflow-app.png](./images/create-dataflow-app.png?raw=true)

4. Complete the parameters as shown in the following image:

   ![dataflow-app.png](./images/dataflow-app.png?raw=true)

5. Click **Create**.

6. After creation, click on the **Scale Demo** link to view details.

7. Click **Run** to execute the job.

8. Confirm the parameters and click **Run** again.

   ![dataflow-run-job.png](./images/dataflow-run-job.png?raw=true)

9. View the Status of the job, wait until the Status changes to **Succeeded** and you can see the results.


   ![dataflow-run-status.png](./images/dataflow-run-status.png?raw=true)


   ![dataflow-run-success.png](./images/dataflow-run-success.png?raw=true)

## Next Steps

The first application publishes data into Kafka Streaming. The second application consumes this data from Kafka.

 - Create another **Data Flow Application** using the same steps when you created the first Data Flow application.

 - You must change the **Name** of your application and change the package, from **loadadw-1.0-SNAPSHOT.jar** to **loadkafka-1.0-SNAPSHOT.jar**.

 - You can retain the other parameters to be the same as the first Data Flow application and RUN the job.

## Related Links

- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)

- [Data Flow Documentation](https://docs.oracle.com/en-us/iaas/data-flow/data-flow-tutorial/getting-started/dfs_tut_get_started.htm#get_started)

- [Data Flow Prerequisites](https://docs.oracle.com/en-us/iaas/data-flow/using/dfs_getting_started.htm#set_up_admin)

- [Develop Oracle Cloud Infrastructure Data Flow Applications Locally, Deploy to The Cloud](https://docs.public.oneportal.content.oci.oraclecloud.com/en-us/iaas/data-flow/data-flow-tutorial/develop-apps-locally/front.htm)

- [Spark submit CLI](https://docs.oracle.com/en-us/iaas/data-flow/data-flow-tutorial/spark-submit-cli/front.htm#front)

- [OCI CLI](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm)

- [Install Apache Spark](https://spark.apache.org/downloads.html)

- [Compartments](https://docs.oracle.com/pt-br/iaas/Content/Identity/Tasks/managingcompartments.htm)

- [Provision Autonomous Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/adbsa/autonomous-provision.html#GUID-0B230036-0A05-4CA3-AF9D-97A255AE0C08)

- [Add the database admin password to Vault](https://docs.oracle.com/en/learn/data-flow-analyze-logs/index.html#add-the-database-admin-password-to-vault)

- [Getting Started With Oracle Streaming Service](https://blogs.oracle.com/developers/post/getting-started-with-oracle-streaming-service-oss)

## Acknowledgments

- **Author** - Cristiano Hoshikawa (Oracle LAD A-Team Solution Engineer)
