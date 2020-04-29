---
title: What 
author: Joe Fecht
date: 2020-04-27 00:00:00 -0600
categories: [Monitoring]
tags: [Azure, monitoring, KQL]
toc: false
---

Some samples

//Tables
AzureDiagnostics // Azure Diagnostics Logs that are configured to go to this workspace

AzureActivity // Azure Activity Logs that are configured to go to this work space via Diagnostic Settings on the Activity Log Itself

AzureMetrics // Azure Platform Metrics - https://docs.microsoft.com/en-us/azure/azure-monitor/platform/data-platform-metrics#sources-of-azure-monitor-metrics

// Get a list of resource types we have reporting to this workspaces
AzureDiagnostics
| distinct ResourceType

// Get a list of resource types taht are not related SQL Databases or KeyVaults
AzureDiagnostics
| where ResourceType !contains "DATABASE"
    and ResourceType !contains "VAULTS"


// Searching the AzureDiagnostics table to find any records that contain the text 'database'. (Search is Case Insensitive by default)
AzureDiagnostics
| search "*database*"

// Getting a list of all records where the resource type is a SQL Database
AzureDiagnostics
| where ResourceType == "SERVERS/DATABASES" 

// Getting a list of unique SQL Databases
AzureDiagnostics
| where ResourceType == "SERVERS/DATABASES" 
| distinct DatabaseName_s


// Getting a list of all records for a database named "jfsqldb01"
AzureDiagnostics
| where DatabaseName_s == "jfsqldb01"

// Getting a list of all records for a Server Name "jfsqlsvr"
AzureDiagnostics
| where LogicalServerName_s == "jfsqlsvr" 

// Get a count of the types of logs associated with the database "jfsqldb01"
AzureDiagnostics
| where DatabaseName_s == "jfsqldb01"
| summarize count() by Category

// Get a list of all of the SQL Databases and number of events
AzureDiagnostics
| where ResourceType == "SERVERS/DATABASES" 
| summarize count() by DatabaseName_s

// Get a list of all activities associated with all the databases
AzureDiagnostics
| where ResourceType == "SERVERS/DATABASE" 
| summarize count() by Category, DatabaseName_s, LogicalServerName_s

// Get a list of events from NSG
AzureDiagnostics
| where ResourceType == "NETWORKSECURITYGROUPS" 

// Get a list of unique NSGs 
AzureDiagnostics
| where ResourceType == "NETWORKSECURITYGROUPS" 
| distinct Resource

// Get a list of blocked events from a specific NSG
AzureDiagnostics
| where Resource == "JF-WIN01-NSG" 
    and type_s == "block"

// Get details on the inbound blocked events from a specific NSG
AzureDiagnostics
| where Resource == "JF-WIN01-NSG" 
    and type_s == "block"
    and direction_s == "In" 
    and priority_d < 65000


// Now we can take the data from the above query and clean it up
AzureDiagnostics
| where Resource == "JF-WIN01-NSG" 
    and type_s == "block"
    and direction_s == "In" 
    and priority_d < 65000
| summarize Occurences=count() by ruleName_s, 
    primaryIPv4Address_s,
    Resource


// Now lets take this and paramterize it
let nsgName = "JF-WIN01-NSG";
let action = "block";
let direction = "In";
AzureDiagnostics
| where Resource == nsgName 
    and type_s == action
    and direction_s == direction
    and priority_d < 65000
| summarize Occurences=count() by ruleName_s, 
    primaryIPv4Address_s,
    Resource