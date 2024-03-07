+++ 
draft = false
date = 2024-03-05T15:09:37+01:00
title = "Amazon MWAA"
description = "Working with Amazon MWAA"
slug = ""
authors = ["Fabrizio Regalli"]
tags = ["aws", "mwaa"]
categories = []
externalLink = ""
series = []
+++

Recently I started using [Amazon MWAA](https://aws.amazon.com/managed-workflows-for-apache-airflow/?nc1=h_ls)  (also know as Amazon Managed Workflow for Apache Airflow)  and I would like to share my experience.

# What is it?
It is a managed service for **Apache Airflow** that lets you use your current, familiar Apache Airflow platform to orchestrate your workflows. You gain improved scalability, availability, and security without the operational burden of managing underlying infrastructure.


# Envs and configuration
The creation of the environment is pretty simple (trough the GUI, not tried with the AWS CLI yet).
Once the environment will be ready, you will see an "**Open Ariflow UI**" link in your Airflow environments console.  

Here you can configure the tasks, the connections and many other things. I'm using [Snowflake](https://www.snowflake.com/en/) as data source, s3 as bucket (for dag), SecretManager for storing the credentials and Python as preferred language.

I don't want to spend many words around **IAM roles** needed for MWAA because it's [very well documented](https://docs.aws.amazon.com/mwaa/latest/userguide/security_iam_service-with-iam.html) but please take a closer look and be assured you have all the roles required for running DAG. (The execution role will be created automatically during the creation of the environment).

# Run it!
You can now connect to the Airflow gui and run your DAG. Take a look of the logs and what it says under "Runs" tab: if you see "Failed" there is a problem. Please check again your code and the configuration on the environment on MWAA Console.

# Common issue
First time I tried, I see an error related to the connection: "The conn_id '{conn_id}' isn't defined"

![python_error](https://i.postimg.cc/jdYzDHSC/2024-03-05-14-45.png)

Luckily it was a pretty simple fix. 
During the creation of the environment I forgot to specify [two important parameters](https://docs.aws.amazon.com/mwaa/latest/userguide/connections-secrets-manager.html#connections-sm-aa-configuration)   

    secrets.backend : airflow.providers.amazon.aws.secrets.secrets_manager.SecretsManagerBackend
and

    secrets.backend_kwargs : {"connections_prefix" : "airflow/connections", "variables_prefix" : "airflow/variables"}
like in the picture:
![Airflow_configuration_options](https://i.postimg.cc/3wLXWNLR/Screenshot-from-2024-03-05-15-02-47.png)

Now if you run again the DAG the error should be disappeard.  
Enjoy with MWAA ;-)  
