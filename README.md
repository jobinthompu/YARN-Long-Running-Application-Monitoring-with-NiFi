# YARN Long Running Application Monitoring with NiFi

## Short Description:

Simple NiFi flow to monitor and Alert on a Long Running YARN Application.


## Introduction

- Here is a small demo how NiFi can help you monitor and alert on Long Running YARN Applications.
- You can download the flow XML I have created [here](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/flow/YARN-Long-Running-Jobs.xml)

## Prerequisite

- Make sure you have your HDF Sandbox up and running. You can download HDF Sandbox [here](https://hortonworks.com/downloads/)

- NiFi_1.2/HDF_3.0 is available on Sandbox and running. In case you are not using HDF sandbox makesure you have latest NiFi downloaded on your machine or Installed via Ambari.

## Steps:

1) Assuming you have NiFi UI up and Available, lets drop GetHTTP processor to pull data from YARN REST API:

Configure Processor with URL given as below, which pulls all Applications in Running state. sandbox-hdf.hortonworks.com is my Resource Manager running on 18088 port:

```
http://sandbox-hdf.hortonworks.com:18088/ws/v1/cluster/apps?states=RUNNING
```

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/1.Query_ResourceManager.jpg)

Lets schedule the processor to run only every 10sec so that you don’t query too often.

2) To make sure the json file received is sent only down stream if its not empty (i.e no jobs are running on the cluster), add a RouteText processor to check null in the content as below:

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/2.Check_For_Empty_Json.jpg)

Route on unmatched relation, only when json is not empty. Auto terminate null connection created above. Connect GetHTTP processor to RouteText for success relation

3) As the Rest call outputs the application details in Json format, lets use a SplitJson processor to separate individualapplication details.
 
```
Provide “JsonPath Expression” value as  “$.apps.app”  in the configuration.
```

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/3.Separate_Jobs.jpg)
 
4) Connect RouteText to SplitJson for 'unmatched' relation and auto terminate 'null' relation.

5) Lets add EvaluateJsonPath processor to extract required fields and add them to flow-file attribute: Configure it as below:

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/4.Extract_Job_Info.jpg)

 Extracted Attribute 'ElapsedTime' is the key player here which tell how long the application was running.

6) Connect SplitJson to EvaluateJsonPath for 'split' relation.

7) Add a RouteOnAttribute processor to the canvas to check value of 'ElapsedTime', here lets check and alert on all application passed 1 Hour and 10 hours. 

```
10hr 	: ${ElapsedTime:gt(36000000)}
1hr 	: ${ElapsedTime:gt(3600000)}
```
![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/5.Check_Elapsedtime_1hr_10hr.jpg)

8) Create and start two sets of DistributedMapCache controller services: 2 DistributedMapCacheClientService, 2 DistributedMapCacheServer for 1hr jobs and 10hr jobs so that we keep track of all the applications crossed 1hr and 10hr and no duplicate alerts for same are sent.

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/6.DistributedMapCacheServers.jpg)

 Make sure those DistributedMapCacheServer run on different ports
 
7) Add a two PutDistributedMapCache processors to update the cache with jobs ran past 1hr and 10hrs respectively. Configure it as below adding Distributed cache service.

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/7.Save_hr_Alerted_Jobs.jpg)

8) Lets auto terminate Failure relationship and connect success relationship to PutEmail processor which will sent out email for any new Yarn Application which crossed the threshold of 1hr and 10hr .

9) Make sure you have formatted the email body and subject to have all information about the failed job:

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/8.Alert_hr_long_Job.jpg)

10) Auto terminate success and failure relationship for PutEmail processor. Once flow is completed, it would look something like below:

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/9.Completed_Flow.jpg)

11) Once you start the Flow, you will get alerts for each Yarn application which cross the threshold set which is 1hr and 10hr. My Email Alert would look like below:

![alt tag](https://github.com/jobinthompu/YARN-Long-Running-Application-Monitoring-with-NiFi/blob/master/Resources/images/10.Email.jpg)


Thanks,

Jobin George

