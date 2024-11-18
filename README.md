# Real-Time Data Streaming Using Apache NiFi, AWS S3, Snowpipe, Stream, and Task

## Overview

This project demonstrates the real time data streaming pipeline by leveraging the capabilities of Apache Nifi , AWS S3, Snowflake, Docker. The pipeline focuses on the generation on data using Python and load the data into AWS S3 using Apache Nifi, a data integration tool , and further load the data into Snowflake tables using Stream and Tasks.The pipeline demonstrates the use of SCD 1 and SCD 2 and also implements Change Data Capture (CDC).

## Architecture Diagram

![image](https://github.com/user-attachments/assets/9ba1df3e-6ab5-481a-8e7d-753347c4b034)

## Technologies Used

* **AWS S3** : It is used for storage of data into buckets in Amazon
* **Snowflake** : A cloud based data warehousing tool for data storage and analysis
* **Docker**: To containerize and manage the Apache NiFi and Jupyter notebook environments
* **Apache Nifi** : A data integration tool to manage, automate, and monitor data flows
* **Amazon EC2** : To host the docker containers

  ## Configuration Steps
  1. EC2 Setup :
     - Setup a EC2 Instance of type t2.large in AWS.
     - Generate the Key Pair if not generated and attach to the instance
     - Setup the Storage of 100 GB
     - Allow SSH and HTTP access
     - Allow ports 4000 - 38888
     - Connect to EC2 via SSH
       
  2. Docker Setup :
     # Run the below commands to setup docker on EC2 machine.
     ```
      sudo yum update -y
      sudo yum install docker
      sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
      sudo chmod +x /usr/local/bin/docker-compose
      sudo gpasswd -a $USER docker
      newgrp docker
      sudo yum install python-pip
      sudo pip install docker-compose
     ```
     # Copy files to EC2
       scp -r -i snowflake-project.pem docker-exp ec2-user@ec2-13-232-189-6.ap-south-1.compute.amazonaws.com:/home/ec2-user/docker_exp

     # Start Docker: 
      sudo systemctl start docker
     # Stop Docker:
      sudo systemctl stop docker
     # Start the containers and installs dependencies
      docker compose up
         
  3. Data Generation:
     - Data is generated using Python Faker Library to generate Fake Data for data processing.
     - Open the JupyterLab inside the docker machine of EC2 using EC2's http://public address:4888
       
  4. Apache Nifi Setup:
     - Open the Apache Nifi inside the docker machine of EC2 using EC2's http://public address:2080
     - Go inside the docker image of apache nifi
      ```
       docker exec -i -t nifi bash
      ```
    - Add processor in Apache Nifi to ListFile, FetchFile and PutS3Object
      ![image](https://github.com/user-attachments/assets/be15f0ac-15e4-45a7-8f43-f858bf6e3009)
    - Run the Nifi Flow which will load the data into Amazon S3 bucket.

  5. Snowflake Setup:
     - Create three tables in Snowflake:
         - Customer Table : SCD1 Table
         - Customer History Table : SCD2 Table
         - Raw Table : Staging Table
    - Create a stream to track of changes made on Customer table
      ```
        CREATE OR REPLACE STAGE Schema_name.stage_name
          url='s3://bucket-name'
          credentials=(aws_key_id='' aws_secret_key='');
      ```
    - Load the data into stage from AWS S3
    - Create a file format
      ```
        CREATE OR REPLACE FILE FORMAT Schema_name.file_format_name
        TYPE = CSV,
        FIELD_DELIMITER = ","
        SKIP_HEADER = 1;
      ```
    - Create a pipe to load data from stage
      ```
      CREATE OR REPLACE PIPE pipe_name
        auto_ingest = true
        AS
        COPY INTO customer_raw
        FROM @stage_name
        FILE_FORMAT = CSV
        ;
      
      show pipes;
      select SYSTEM$PIPE_STATUS('pipe_name');
      ```
    -  To capture the CDC , use merge statement. Schedule it to run every minute
   ```
      CREATE OR REPLACE PROCEDURE pdr_scd_demo()
        returns string not null
        language javascript
        as
            $$
              var cmd = `
                         merge into customer c 
                         using customer_raw cr
                            on  c.customer_id = cr.customer_id
                         when matched and c.customer_id <> cr.customer_id or
                                          c.first_name  <> cr.first_name  or
                                          c.last_name   <> cr.last_name   or
                                          c.email       <> cr.email       or
                                          c.street      <> cr.street      or
                                          c.city        <> cr.city        or
                                          c.state       <> cr.state       or
                                          c.country     <> cr.country then update
                             set c.customer_id = cr.customer_id
                                 ,c.first_name  = cr.first_name 
                                 ,c.last_name   = cr.last_name  
                                 ,c.email       = cr.email      
                                 ,c.street      = cr.street     
                                 ,c.city        = cr.city       
                                 ,c.state       = cr.state      
                                 ,c.country     = cr.country  
                                 ,update_timestamp = current_timestamp()
                         when not matched then insert
                                    (c.customer_id,c.first_name,c.last_name,c.email,c.street,c.city,c.state,c.country)
                             values (cr.customer_id,cr.first_name,cr.last_name,cr.email,cr.street,cr.city,cr.state,cr.country);
              `
              var cmd1 = "truncate table Schema_name.customer_raw;"
              var sql = snowflake.createStatement({sqlText: cmd});
              var sql1 = snowflake.createStatement({sqlText: cmd1});
              var result = sql.execute();
              var result1 = sql1.execute();
            return cmd+'\n'+cmd1;
            $$;
        call pdr_scd_demo();   

  ```
  - Create a task to automate the merge statement
    ```
      #Set up TASKADMIN role
      use role securityadmin;
      create or replace role taskadmin;
      #Set the active role to ACCOUNTADMIN before granting the EXECUTE TASK privilege to TASKADMIN
      use role accountadmin;
      grant execute task on account to role taskadmin;

      create or replace task tsk_scd_hist warehouse= COMPUTE_WH schedule='1 minute'
      ERROR_ON_NONDETERMINISTIC_MERGE=FALSE
      as
      call pdr_scd_demo();
      
    ```
  - Alter Task
    ```
    show tasks;
    alter task tsk_scd_raw suspend;--resume --suspend
    ```

    ## Conclusion
    This project demonstrates the real time streaming , extracting data from Python code and loading it into S3 and transforming it in Snowflake which enables to manage, analyse large volumes of data. 
