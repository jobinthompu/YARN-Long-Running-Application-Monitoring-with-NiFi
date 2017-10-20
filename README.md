# YARN Long Running Application Monitoring with NiFi

## Short Description:

Simple NiFi flow to monitor and Alert on a Long Running YARN Application.


## Introduction
- Here is a small demo how NiFi can help you monitor and alert on Long Running YARN Applications.

## TEST Will decide about Video Later

- Here you can view the screen recording that Demonstrates how it works!

[![YARN Application Monitoring with NiFi](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/YARN-Application-Monitoring-with-NiFi.jpg)](https://youtu.be/Ez9NPIS0XS8 "YARN Application Monitoring with NiFi - Click to Watch!")


## Prerequisite

- Make sure you have your HDF Sandbox up and running. you can download HDF Sandbox [here](https://hortonworks.com/downloads/)

- NiFi_1.2/HDF_3.0 is available on Sandbox and running. In case you are not using HDF sandbox makesure you have latest NiFi is downloaded on your machine or Installed via Ambari.

## Steps:

1) Assuming you have NiFi UI up and Available, lets drop GetHTTP processor to pull data from YARN REST API:

Configure Processor with URL given as below, which pulls all Applications in Running state. sandbox-hdf.hortonworks.com is my Resource Manager running on 18088 port:

```
http://sandbox-hdf.hortonworks.com:18088/ws/v1/cluster/apps?states=RUNNING
```

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/1.GetHTTP-processor.jpg)

Lets schedule the processor to run only every 10sec so that you don’t query too often.

2) To make sure the json file received is sent only down stream if its not empty (i.e no jobs are running on the cluster), add a RouteText processor to check null in the content as below:

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/1.GetHTTP-processor.jpg)

Route on unmatched relation, only when json is not empty. Auto terminate null connection created above. Connect GetHTTP processor to RouteText for success relation

3) As the Rest call outputs the application details in Json format, lets use a SplitJson processor to separate individualapplication details.
 
```
Provide “JsonPath Expression” value as  “$.apps.app”  in the configuration.
```

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/2.SplitJson.jpg)
 
4) Connect RouteText to SplitJson for 'unmatched' relation and auto terminate 'null' relation.

5) Lets add EvaluateJsonPath processor to extract required fields and add them to flow-file attribute: Configure it as below:

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/3.EvaluateJsonPath.jpg)

5) Connect SplitJson to EvaluateJsonPath for success relation.

6) Create and start two controller services: DistributedMapCacheClientService, DistributedMapCacheServer so that we keep track of all the applications and don’t sent out duplicate alerts for same application.

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/4.DistributedMapCacheClientService.jpg)

7) Add a PutDistributedMapCache processor to update the cache with latest apps that fails/killed. Configure it as below adding Distributed cache service.

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/5.PutDistributedMapCache.jpg)

8) Lets auto terminate Failure relationship and connect success relationship to PutEmail processorwhich will sent out email for any new failed/killed application.

9) Make sure you have formatted the email body and subject to have all information about the failed job:

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/6.PutEmail.jpg)

10) Auto terminate success and failure relationship for PutEmail processor. Once you start the Flow, you will get alerts for each Killed/Failed Yarn application. My Alert would look like below:

![alt tag](https://github.com/jobinthompu/YARN-Application-Monitoring-with-NiFi/blob/master/Resources/images/7.Email.jpg)

Note: Now you can configure your GetHTTP Processor to query YARN to find long running applications

* You can also refer my HCC Article [here](https://community.hortonworks.com/articles/34147/nifi-security-user-authentication-with-kerberos.html)

Thanks,

Jobin George

