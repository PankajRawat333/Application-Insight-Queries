# Application-Insight-Queries
## Azure Application Insight query example.

### 1. Get all trace where at least one Error trace present
```
requests
| where success == "False" and timestamp >= ago(7d) 
| join kind= inner traces on operation_Id  
| project operation_Id , timestamp, message, severityLevel, appName 
| order  by timestamp, operation_Id
```
Note: severityLevel :- 1 = Info and 3 = Error

### 2. Get Error trace only
```
requests
| where success == "False" and timestamp >= ago(7d) 
| join kind= inner (
traces
| where severityLevel == 3
) on operation_Id  
| project operation_Id , timestamp, message, severityLevel  
| order  by timestamp, operation_Id
```
### 3. Get Error trace only (using timechart)
```
requests
| where success == "False" and timestamp >= ago(7d)
| join kind= inner (
traces
| where severityLevel == 3
) on operation_Id  
| summarize event_count=count() by bin(timestamp, 1h) 
| render timechart
```

### 4. Count all record in all tables
```
union *
| count 
```

### 5. Count all record in particular tables
```
requests
| count
```

### 6. Filter by regex
```
requests
| where timestamp >= ago(1d) 
| where operation_Name matches regex ".*Func" 
| limit 10 
| order by timestamp desc 
```

```
requests
| where timestamp >= ago(1d) 
| where * has "function" 
```

### 7. Show me dependencies related to slow requests
```
requests
| where timestamp > ago(1d)
| where duration > 1000
| limit 100
| order by duration desc
```

### 8. Extract data from Json
```
traces
| where timestamp <= ago(1d) 
| where  message has "header"
| extend jsonObj = parse_json(message) 
| project timestamp, messageTimestamp=jsonObj.messageTimestamp, message
| limit 1000
| order by timestamp desc 
```

### 9. Extending with new calculated fields
```
requests
| where timestamp > ago(1d)
| extend responseBucket = iff(duration > 1000, "Too long", "Ok") 
| project name, duration , responseBucket 
```

### 10. Aggregation
```
requests
| where timestamp > ago(7d)
| summarize slaMet=count(duration<2000),slaBreached=count(duration>=2000) by bin(timestamp, 1h)  
| render timechart 
```

### 11. Check service meeting SLA
```
requests
| where timestamp > ago(7d)
| summarize slaMet=count(duration<2000),slaBreached=count(duration>=2000), totalCount=count()  by bin(timestamp, 1h) 
| extend pctIndex = slaMet * 100.0/totalCount 
| project pctIndex ,timestamp
| render timechart  
```

### 12. Percentiles
```
requests
| where timestamp > ago(1d)
| summarize percentiles(duration, 50, 95), reqCount100s=count()/100 by bin(timestamp, 1h)  
| render timechart  
```

### 13. Analyzing latest failure
```
requests
| where timestamp >= ago(7d)
| where success == "False" 
| summarize arg_max(timestamp, name, resultCode) by cloud_RoleInstance
```


### 14. Reduce by
```
traces
| where timestamp >= ago(7d)
| summarize Count=count() by message
| reduce by message 
```

### 15. Distinct page view by session
```
pageViews
| where timestamp >= ago(7d)
| summarize dcount(name) by session_Id  
| order by session_Id
```

```
pageViews
| where timestamp >= ago(7d)
| summarize dcount(name) by session_Id  
| order by dcount_name  
```

### 16. Top set of page people visited
```
pageViews
| where timestamp >= ago(7d)
| order by timestamp desc
| summarize pageVisited = makelist(name) by session_Id 
| top 10
```

### 17. what are the top 10 common page flows for my users
```
pageViews
| where timestamp >= ago(7d)
| order by timestamp desc
| summarize pageVisited = makelist(name) by session_Id 
| summarize count() by tostring(pageVisited)  
| top 10 by count_ desc
```
### 18. SelfJoin, RegEx Filter and extract Json Data
```
traces
| where timestamp > ago(3d)
| join kind= inner (
   traces
    | extend jsonObj = parse_json(message) 
    | where timestamp > ago(3d)
    | where message matches regex ".*RAM01355"
    | limit 1
) on operation_Id 
| order by timestamp asc 
```

### 19. Search any data in any column by search keyword
```
traces
    | extend jsonObj = parse_json(message) 
    | where operation_Name == "ApplicationName" 
    | where timestamp > ago(30d)
    | search "2c2373f5-5adb-4552-9b44-f303770c4abc"  
    | limit 50
    | order by timestamp asc 
```

### 20. Counting the number of unique VM in 1 minute.
```
let start = datetime(2019-01-25T00:00);
customMetrics
| where timestamp > start
| where timestamp < start +1m
| where name == "Function1 Successes" 
| summarize uniqueVMCount=dcount(cloud_RoleInstance)
```

### 21. Check work distrubution acroll VM
```
let start = datetime(2019-01-25T00:00);
customMetrics
| where timestamp > start
| where timestamp < start +1m
| where name == "Function1 Successes" 
| summarize sum(value) by cloud_RoleInstance
| render piechart 
```

### 22. Check application metrics (Successes, Failures and Duration).
```
let start = datetime(2019-01-30T00:00);
customMetrics
| where timestamp > start
| where timestamp < start +1d
| where name in ("Func Successes", "Func Failures" ,"Func Duration")
| summarize sum(value) by name, timestamp
| render timechart     
```

### 23. Query with IN clause or subquery
```
let operationids = toscalar ( // convert single column to value
traces
| where operation_Name == "Function" and message contains "1560" 
| summarize makeset(operation_Id));
traces
| where operation_Id in (operationids)   
```

### 24. Get distinct records
```
traces
| where operation_Name == "Function" and message contains "1560" 
| distinct operation_Id  
```

### 25. Get Azure durable function trace logs from Application Insights
```
traces
| extend functionName = tostring(customDimensions["prop__functionName"]) 
| extend instanceId = tostring(customDimensions["prop__instanceId"]) 
| extend state = tostring(customDimensions["prop__state"]) 
| extend isReplay = tobool(customDimensions["prop__isReplay"]) 
| extend output = tostring(customDimensions["prop__output"]) 
| extend functionType = tostring(customDimensions["prop__functionType"]) 
| where isReplay != true
| project timestamp, instanceId , functionName ,state, output , appName = cloud_RoleName , functionType 
| summarize arg_max(timestamp, *) by instanceId 
| order by timestamp desc nulls last 
```
