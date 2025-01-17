---
title: Samples of Azure Monitor log alert rule queries
description: See examples of Azure monitor log alert rule queries.
ms.topic: reference
ms.date: 01/04/2024
author: AbbyMSFT
ms.author: abbyweisberg
ms.reviewer: nolavime
---

# Sample log alert queries that include ADX and ARG

A log alert rule monitors a resource by using a Log Analytics query to evaluate resource logs at a set frequency. You can include data from Azure Data Explorer and Azure Resource Graph in your log alert rule queries.

This article provides examples of log alert rule queries that use Azure Data Explorer and Azure Resource Graph. For more information about creating a log alert rule, see [Create a log alert rule](./alerts-create-log-alert-rule.md).

## Query that checks virtual machine health

This query finds virtual machines that are marked as critical and that had a heartbeat more than 24 hours ago, but that haven't had a heartbeat in the last 2 minutes.

```kusto
{
    arg("").Resources
    | where type == "microsoft.compute/virtualmachines"
    | where tags.BusinessCriticality =~ 'critical' and subscriptionId == '123-456-123-456'
    | join kind=leftouter (
    Heartbeat
    | where TimeGenerated > ago(24h)
    )
    on $left.name == $right.Resource
    | summarize LastCall = max(case(isnull(TimeGenerated), make_datetime(1970, 1, 1), TimeGenerated)) by name, id
    | extend SystemDown = case(LastCall < ago(2m), 1, 0)
    | where SystemDown == 1
}

```

## Query that filters virtual machines that need to be monitored

```kusto
{
    let RuleGroupTags = dynamic(['Linux']);
    Perf | where ObjectName == 'Processor' and CounterName == '% Idle Time' and (InstanceName == '_Total' or InstanceName == 'total')
    | extend CpuUtilisation = (100 - CounterValue)   
    | join kind=inner hint.remote=left (arg("").Resources
    | where type =~ 'Microsoft.Compute/virtualMachines'
    | project _ResourceId=tolower(id), tags) on _ResourceId
    | project-away _ResourceId1
    | where (isnull(tags.monitored) or tolower(tostring(tags.monitored)) != 'false') and (tostring(tags.monitorRuleGroup) in (RuleGroupTags) or isnull(tags.monitorRuleGroup) or tostring(tags.monitorRuleGroup) == '')
}
```

## Query that finds resources with certificates that are going to expire within 30 days

```kusto
{
    arg("").Resources
    | where type == "microsoft.web/certificates"
    | extend ExpirationDate = todatetime(properties.expirationDate)
    | project ExpirationDate, name, resourceGroup, properties.expirationDate
    | where ExpirationDate < now() + 30d
    | order by ExpirationDate asc
}
```

## Query that alerts when a new resource is created in the subscription

```kusto
{
    arg("").resourcechanges
    | extend changeTime = todatetime(properties.changeAttributes.timestamp), targetResourceId = tostring(properties.targetResourceId),
    changeType = tostring(properties.changeType),targetResourceType = tostring(properties.targetResourceType),
    changedBy = tostring(properties.changeAttributes.changedBy)
    | where changeType == "Create"
    | project changeTime,targetResourceId,changedBy
}
```

## Next steps
- [Learn more about creating a log alert rule](./alerts-create-log-alert-rule.md)
- [Learn how to optimize log alert queries](./alerts-log-query.md)
