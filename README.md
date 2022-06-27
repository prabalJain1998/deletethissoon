
# Spot PC Data Analysis for Cross Tenant Data 
Spot PC Data Insights allows you to visualize average data across all the VM's (Virtual Machines)/ Tenants. 

<img alt="PyPI - Python Version" src="https://img.shields.io/badge/python-3.9.7-yellow"><img alt="PyPI - FastAPI" src="https://img.shields.io/badge/FastAPI-0.45.0-green">

### Table of Contents

**[Technologies](#0-technologies)**<br>
**[Setup](#1-setup)**<br>
**[Overview](#2-overview)**<br>
**[Directory Structure](#3-directory-structure)**<br>
**[Schema](#4-schema)**<br>

## 0. Technologies
![image](/Images/techstack.png)

## 1. Setup

### 1.1 How to Setup in Visual Studio Code
1. Install latest python version 3.9.7 from [here](https://www.python.org/downloads/), Install visual Studio code from [here](https://code.visualstudio.com/download/)
2. Create a new directory 
   ```bash
   $ mkdir SpotCrossTenantInsights
   ```
3. clone the git repository
   ```bash
   $ git clone https://github.com/Virtual-Desktop-Service/spotpc-research-development.git
   ```
4. Move to spotpc-research-development directory
   ```bash
   $ cd spotpc-research-development
   ```
5. Create a virtual environment using the following command
   ```bash
   $ python -m venv ./venv
   ```
6. Open spotpc-research-development in Visual Studio Code and activate the virtual environment 
   ```bash
   $ cd venv/Scripts & activate 
   ```
7. Move back to spotpc-research-development
8. Install the Packages
   ```bash
   $ pip install -r requirements.txt
   ```
9. run startup.bat 
   ```bash
   $ startup.bat
   ```

## 2. Overview
Without accessing cross tenant data, a customer is unable to visualize and get the data on how their VM’s are performing w.r.t other VM’s inside Tenant as well as across Tenants. Having all the data of all tenants in one place allows us for monitoring the behavior of the entire system as a whole and the current load on the systems. Cloud Insights data is only stored per tenant so there is no way to visualize the data of the rest **n-1** tenants in the SPOT PC Dashboard. For any future work like : Prediction of usage and Login-logout time of machines, the system would need a lot of data to identify patterns. This aggregation of cross tenant data would allow for future endeavors in machine learning. 

The remote agent inside of each VM (Virtual Machine) is updated to push the windows log data to a Azure Service Bus.

The project is divided into 5 subdivisions based on the requirement:
   - Remote agent modification for Data Manipulation to compute averages per minute w.r.t a VM (Virtual Machine) present in a tenant. 
   - Creation of an ETL(Extract-Transform-Load) pipeline to handle incoming VDS (Virtual Desktop Services) data i.e Streaming data. 
   - Minute-wise average calculation from centralized Spot PC Database and writing it to Report Table with the help of Azure Function App as a schedular. 
   - FastAPI Creation to access the data from Report table. 
   - Azure services which are being used to serve a purpose in the pipeline. Since several options are available for the same task, we had to do a cost analysis          which would yield our desired throughput for a low price.

### 2.1 Remote Agent modification  
 
Previously remote agent collects the data for performance counters in every 10 seconds, As we need to compute averages across the tenants it would be inaccurate to consider the raw timestamps as for specific performance counters that depends on user processes may lead to add a delay in getting the data, so  value of that performance counters will not be available during that Timestamp. The remote agent pushes data to the data lake every 15 minutes. The name of the Json files is in the format of “MaskedTenantName_MaskedVMName_TimeStamp.”   
#### Solution :
We know that the data for any Performance counter will not be delayed by a minute (i.e. we can get t least one value of Permormance counter in a minute)
Now the remote agent is modified in such a way that for every counters there will be multiple (>=1) values which can be replaced by a single value by taking average/sum/min/max (depends on the performance counter) and the raw-timestamp will be replaced by an Identifier Timestamp which will be the 30th second of that minute.

**SPOC** : SPOC is a service where all the VM's push their 15 minutes of data, SPOC will perform aggregation of the data and push the JSON data to the Service Bus.

Sample JSON Format for incoming Data 
```python
{ "metrics": [{ "timestamp": 1648637647, 
                "MaskedTenantId": "wYAVC3oWQU+ozhgAAg5f/w==637841132085596403", 
                "SiteName": null, 
                "MaskedVMName": "Sharedfk6hZ4v5Xkmu0clBsg+aYw==63784", 
                "ResourcePoolName": "Shared-ngXXH", 
                "Region": "centralindia", 
                "MachineSize": "Standard_D4s_v4", 
                "PerfName": "WinDefend", 
                "PerfValue": 1, 
                "PerfType": "win_service" 
    }]} 
```


### 2.2. ETL Data Pipeline

![image](/Images/pipeline.gif)

The ETL Data Pipeline is responsible for continous flow of VDS Data from remote agent to SPOT Report Table.

The ETL Pipeline works in 3 Steps :

**Step 1 :** A listener is implemented and hosted along with the FastAPI's which listens to Service bus and as soon as there is a message in service bus it will deque the message, deserialize the message and then push it into the Centralized Spot PC Database.

**Step 2 :** Timer Trigger is used as a Schedular in Azure Function App which will invoke in every n minutes and it is responsible to compute averages across tenants and writing it in the Report SPOT PC table.

**Step 3 :** GET API's are implemented in order to retrieve the average data per minute across the tenant from the Report SPOT PC Table using Fast API's .

### 2.3. SPOC 
   - SPOC runs on the SpotPCManager1 and aggregates all the data of the tenant, i.e all the VM's under the tenant in which it is deployed. Each of the VM's therefore do not have access to the service bus queue. SPOC pushes the messages to the service bus for each tenant. It is also the only point of connection to the global queue.

### 2.4. Azure Service Bus 
   - The service bus holds each of the messages from SPOC. The messaging service ensures that none of the messages are lost in the event that the service bus listener is unable to process the message and put the data onto the central cosmos db. It is currently using quees as there is only a single consumer for the data that is coming onto the service bus. The service bus is size limited and the messages can have a maximum size of 256kb so SPOC will have to make sure the message size limit is maintained. 
   
### 2.5. Azure Cosmos Database 
   - Currently there are 2 Tables inside Spot Cosmos Database :
      - Spot Cosmos Table : It contains Processed Structured Data from all the VM's (Virtual Machines) across all Tenants for every Performance Counters. 
      - Spot Report Cosmos Table : It contains Averages for desired Performance Counters for every Timestamps (Unit : Minutes) and for every counters across all Tenants/VM's.  
    
### 2.5. Azure Function
   - Azure Trigger Function is used which gets triggered in every *x minutes* (we can modify it as per our convenience).   
   - Azure function triggers in every x minute, it finds the current Timestamp available in the report table for desired Performance Counters and it queries the Spot Cosmos Centralized Table by running an average query for the next Timestamps available and writed the result back into Spot Report Table. Hence it works as a Trigger so that our Report table can be updated frequently to make the near real time experience for the user. 

### 2.6. FastAPI 
   - We are using FastAPI for implementing API's to retreive the desired data from Spot Report Table.
   - For Testing Purposes :
      - Implemented API to get specific Tenant and VM specific data along with the averages to plot the graph that will be shown in the UI. 

#### 2.6.1 : Endpoints :

- **/v1/get-data/?key=win_disk.percent_idle_time&minutes=5&aggregationInterval=1&aggregationBy="avg"**
   - **key** : It is the concatination of PerfType and PerfName concatenated by "."
   - **minutes** : To get the Report data for latest n minutes.
   - **aggregationInterval** : To get the aggregated Data its value represent how many Items should get aggregated, default value = 1.
   - **aggregationBy** : It accepts 3 values : ("avg", "min", "max") to decide the criterion for aggregating the values.
               
- **/v1/get-tenant-data?key=win_swap.percent_usage&minutes=50&aggregationInterval=1&tenant=Tenant1&vm=VM1&aggregationBy="avg"**
   - **key** : It is the concatination of PerfType and PerfName concatenated by "."
   - **minutes** : To get the Report data for latest n minutes.
   - **aggregationInterval** : To get the aggregated Data its value represent how many Items should get aggregated, default value = 1.
   - **aggregationBy** : It accepts 3 values : ("avg", "min", "max") to decide the criterion for aggregating the values.
   - **tenant**: Masked Name of the Tenant.
   - **vm** : Masked Name of the VM inside the Tenant.

## 3. Directory Structure
    .
    ├── ...
    ├── routers                     # Contains FastAPI specific files.
        ├── SpotDataAggregate
            ├── api.py           
            ├── routers.py          # Contains routers/endpoints implementation
    ├── spotCosmosTimerTrigger                    
    │   ├── spotCosmosTimerTrigger  # Contains all Files necessary for Azure Timer Trigger 
    │       ├── init.py             # Entry Point for Timer Trigger
    │       ├── function.json           
    │       ├── sample.dat 
    │   ├── config.yaml             # Contains Credentials for Database and Service Bus Connectivity
    │   ├── host.json
    │   ├── requirements.txt        # Contains necessary package to install
    │   ├── spot.py                 # Entry Point for instantiating Objects
    │   ├── spotCosmosDB.py         # Contains Methods for getting instance of database/collection.
    │   ├── spotReport.py           # Contains Logic for Calculating Averages/minute across all tenants
    ├── spotGraphUI
    │   ├── data.js
    │   ├── index.html              # WebPage to Showcase Graph UI
    │   ├── package-lock.json
    │   ├── package.json
    ├── .gitignore  
    ├── .funcignore  
    ├── init.py                  
    ├── app.py 
    ├── cleanup.py                  # Cleans Table inside the Database.
    ├── config.yaml                 # Contains Credentials for Database and Service Bus Connectivity
    ├── createSampleData.py         # Script to generate Mock Data
    ├── requirements.txt            # Contains necessary package to install
    ├── spotCosmosDB.py             # Contains Methods for getting instance of database/collection.
    ├── spotInsert.py               # Testing file for database connectivity
    ├── startup.bat                 # contains startup command for API's
    └── ...
 
## 4. Schema 
There are 2 tables which are being maintained for the pipeline. 
### 4.1 SPOT COSMOS : Centralized Table || Structured Average per minute Data.
```python
 {              
      "timestamp": 1656209500, 
      "MaskedTenantId": "wYAVC3oWQU+ozhgAAg5f/w==637841132085596403", 
       "SiteName": "Site1, 
       "MaskedVMName": "Sharedfk6hZ4v5Xkmu0clBsg+aYw==63784", 
       "ResourcePoolName": "Shared-ngXXH", 
       "Region": "centralindia", 
       "MachineSize": "Standard_D4s_v4", 
       "PerfName": "WinDefend", 
       "PerfValue": 1, 
       "PerfType": "win_service" 
 }
```

### 4.2 SPOT REPORT COSMOS : Report Table || Structured Average per minute across all VM's Data.
```python
 {              
      "timestamp": 1656209500,  
       "PerfName": "WinDefend", 
       "PerfValue": 1, 
       "PerfType": "win_service",
       "average":5.003243
 }
```

 ## Contributors ✨

