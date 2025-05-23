---
title: Analyze Azure Functions telemetry in Application Insights
description: Learn how to view and query for Azure Functions telemetry data collected by and stored in Azure Application Insights.
ms.topic: how-to
ms.custom:
  - build-2024
  - ignite-2024
ms.date: 10/14/2020
# Customer intent: As a developer, I want to view and query the data being collected from my function app so I can know if it's running correctly and to make improvements.
---
# Analyze Azure Functions telemetry in Application Insights 

Azure Functions integrates with Application Insights to better enable you to monitor your function apps. Application Insights collects telemetry data generated by your function app, including information your app writes to logs. Application Insights integration is typically enabled when your function app is created. If your function app doesn't have the instrumentation key set, you must first [enable Application Insights integration](configure-monitoring.md#enable-application-insights-integration). 

By default, the data collected from your function app is stored in Application Insights. In the [Azure portal](https://portal.azure.com), Application Insights provides an extensive set of visualizations of your telemetry data. You can drill into error logs and query events and metrics. This article provides basic examples of how to view and query your collected data. To learn more about exploring your function app data in Application Insights, see [What is Application Insights?](/azure/azure-monitor/app/app-insights-overview). 

To be able to view Application Insights data from a function app, you must have at least Contributor role permissions on the function app. You also need to have the [Monitoring Reader permission](/azure/azure-monitor/roles-permissions-security#monitoring-reader) on the Application Insights instance. You have these permissions by default for any function app and Application Insights instance that you create.   

To learn more about data retention and potential storage costs, see [Data collection, retention, and storage in Application Insights](/previous-versions/azure/azure-monitor/app/data-retention-privacy).   

## Viewing telemetry in Monitor tab

With [Application Insights integration enabled](configure-monitoring.md#enable-application-insights-integration), you can view telemetry data in the **Monitor** tab.

1. In the function app page, select a function that has run at least once after Application Insights was configured. Then, select **Monitor** from the left pane. Select **Refresh** periodically, until the list of function invocations appears.

   ![Invocations list](media/functions-monitoring/monitor-tab-ai-invocations.png)

    > [!NOTE]
    > It can take up to five minutes for the list to appear while the telemetry client batches data for transmission to the server. The delay doesn't apply to the [Live Metrics Stream](/azure/azure-monitor/app/live-stream). That service connects to the Functions host when you load the page, so logs are streamed directly to the page.

1. To see the logs for a particular function invocation, select the **Date (UTC)** column link for that invocation. The logging output for that invocation appears in a new page.

   ![Invocation details](media/functions-monitoring/invocation-details-ai.png)

1. Choose **Run in Application Insights** to view the source of the query that retrieves the Azure Monitor log data in Azure Log. If this is your first time using Azure Log Analytics in your subscription, you're asked to enable it.

1. After you enable Log Analytics, the following query is displayed. You can see that the query results are limited to the last 30 days (`where timestamp > ago(30d)`), and the results show no more than 20 rows (`take 20`). In contrast, the invocation details list for your function is for the last 30 days with no limit.

   ![Application Insights Analytics invocation list](media/functions-monitoring/ai-analytics-invocation-list.png)

For more information, see [Query telemetry data](#query-telemetry-data) later in this article.

## View telemetry in Application Insights

To open Application Insights from a function app in the [Azure portal](https://portal.azure.com):

1. Browse to your function app in the portal.

1. Select **Application Insights** under **Settings** in the left page. 

1. If this is your first time using Application Insights with your subscription, you'll be prompted to enable it. To do this, select **Turn on Application Insights**, and then select **Apply** on the next page.

![Open Application Insights from the function app Overview page](media/functions-monitoring/ai-link.png)

For information about how to use Application Insights, see the [Application Insights documentation](/azure/azure-monitor/app/app-insights-overview). This section shows some examples of how to view data in Application Insights. If you're already familiar with Application Insights, you can go directly to [the sections about how to configure and customize the telemetry data](configure-monitoring.md#configure-log-levels).

![Application Insights Overview tab](media/functions-monitoring/metrics-explorer.png)

The following areas of Application Insights can be helpful when evaluating the behavior, performance, and errors in your functions:

| Investigate | Description |
| ---- | ----------- |
| **[Failures](/azure/azure-monitor/app/asp-net-exceptions)** |  Create charts and alerts based on function failures and server exceptions. The **Operation Name** is the function name. Failures in dependencies aren't shown unless you implement custom telemetry for dependencies. |
| **[Performance](/azure/azure-monitor/app/performance-counters)** | Analyze performance issues by viewing resource utilization and throughput per **Cloud role instances**. This performance data can be useful for debugging scenarios where functions are bogging down your underlying resources. |
| **[Metrics](/azure/azure-monitor/essentials/metrics-charts)** | Create charts and alerts that are based on metrics. Metrics include the number of function invocations, execution time, and success rates. |
| **[Live Metrics](/azure/azure-monitor/app/live-stream)** | View metrics data as it's created in near real time. |

## Query telemetry data

[Application Insights Analytics](/azure/azure-monitor/logs/log-query-overview) gives you access to all telemetry data in the form of tables in a database. Analytics provides a query language for extracting, manipulating, and visualizing the data. 

Choose **Logs** to explore or query for logged events.

![Analytics example](media/functions-monitoring/analytics-traces.png)

Here's a query example that shows the distribution of requests per worker over the last 30 minutes.

```kusto
requests
| where timestamp > ago(30m) 
| summarize count() by cloud_RoleInstance, bin(timestamp, 1m)
| render timechart
```

The tables that are available are shown in the **Schema** tab on the left. You can find data generated by function invocations in the following tables:

| Table | Description |
| ----- | ----------- |
| **traces** | Logs created by the runtime, scale controller, and traces from your function code. For Flex Consumption plan hosting, `traces` also includes logs created during code deployment. |
| **requests** | One request for each function invocation. |
| **exceptions** | Any exceptions thrown by the runtime. |
| **customMetrics** | The count of successful and failing invocations, success rate, and duration. |
| **customEvents** | Events tracked by the runtime, for example: HTTP requests that trigger a function. |
| **performanceCounters** | Information about the performance of the servers that the functions are running on. |

The other tables are for availability tests, and client and browser telemetry. You can implement custom telemetry to add data to them.

Within each table, some of the Functions-specific data is in a `customDimensions` field.  For example, the following query retrieves all traces that have log level `Error`.

```kusto
traces 
| where customDimensions.LogLevel == "Error"
```

The runtime provides the `customDimensions.LogLevel` and `customDimensions.Category` fields. You can provide additional fields in logs that you write in your function code. For an example in C#, see [Structured logging](functions-dotnet-class-library.md#structured-logging) in the .NET class library developer guide.

## Query function invocations

Every function invocation is assigned a unique ID. `InvocationId` is included in the custom dimension and can be used to correlate all the logs from a particular function execution.

```kusto
traces
| project customDimensions["InvocationId"], message
```

## Telemetry correlation

Logs from different functions can be correlated using `operation_Id`. Use the following query to return all the logs for a specific logical operation.

```kusto
traces
| where operation_Id == '45fa5c4f8097239efe14a2388f8b4e29'
| project timestamp, customDimensions["InvocationId"], message
| order by timestamp
```

## Sampling percentage

Sampling configuration can be used to reduce the volume of telemetry. Use the following query to determine if sampling is operational or not. If you see that `RetainedPercentage` for any type is less than 100, then that type of telemetry is being sampled.

```kusto
union requests,dependencies,pageViews,browserTimings,exceptions,traces
| where timestamp > ago(1d)
| summarize RetainedPercentage = 100/avg(itemCount) by bin(timestamp, 1h), itemType
```
## Query scale controller logs

_This feature is in preview._

After enabling both [scale controller logging](configure-monitoring.md#configure-scale-controller-logs) and [Application Insights integration](configure-monitoring.md#enable-application-insights-integration), you can use the Application Insights log search to query for the emitted scale controller logs. Scale controller logs are saved in the `traces` collection under the **ScaleControllerLogs** category.

The following query can be used to search for all scale controller logs for the current function app within the specified time period:

```kusto
traces 
| extend CustomDimensions = todynamic(tostring(customDimensions))
| where CustomDimensions.Category == "ScaleControllerLogs"
```

The following query expands on the previous query to show how to get only logs indicating a change in scale:

```kusto
traces 
| extend CustomDimensions = todynamic(tostring(customDimensions))
| where CustomDimensions.Category == "ScaleControllerLogs"
| where message == "Instance count changed"
| extend Reason = CustomDimensions.Reason
| extend PreviousInstanceCount = CustomDimensions.PreviousInstanceCount
| extend NewInstanceCount = CustomDimensions.CurrentInstanceCount
```

## Query Flex Consumption code deployment logs

The following query can be used to search for all code deployment logs for the current function app within the specified time period:

```kusto
traces
| extend deploymentId = customDimensions.deploymentId
| where deploymentId != ''
| project timestamp, deploymentId, message, severityLevel, customDimensions, appName
```

## Consumption plan-specific metrics

When running in a [Consumption plan](consumption-plan.md), the execution *cost* of a single function execution is measured in *GB-seconds*. Execution cost is calculated by combining its memory usage with its execution time. To learn more, see [Estimating Consumption plan costs](functions-consumption-costs.md).

The following telemetry queries are specific to metrics that impact the cost of running functions in the Consumption plan.

[!INCLUDE [functions-consumption-metrics-queries](../../includes/functions-consumption-metrics-queries.md)]

## Next steps

Learn more about monitoring Azure Functions:

+ [Monitor Azure Functions](functions-monitoring.md)
+ [How to configure monitoring for Azure Functions](configure-monitoring.md)
