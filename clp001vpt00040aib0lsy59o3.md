---
title: "Collecting Diagnostic Data Automatically with dotnet-monitor"
datePublished: Wed Nov 15 2023 16:51:32 GMT+0000 (Coordinated Universal Time)
cuid: clp001vpt00040aib0lsy59o3
slug: collecting-diagnostic-data-automatically-with-dotnet-monitor
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1699969887791/6bec3d40-2747-4a9c-9a43-94ce5afb05f7.png
tags: net, diagnostics, dotnet-monitor

---

In the article [Diagnostic .NET Apps using dotnet-monitor](https://blog.raulnq.com/diagnostic-net-apps-using-dotnet-monitor), we explored the process of collecting diagnostic data on demand through a REST API. This approach is useful when we are aware that our application is facing an issue and want to gather information. In addition to this approach, `dotnet-monitor` can be easily configured to automatically gather diagnostic data when a specific condition is met by using [Collection Rules](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionrules.md).

## Diagnostics Port

`dotnet-monitor` communicates with .NET applications through their [diagnostic port](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/diagnostic-port).

> The .NET runtime exposes a service endpoint that allows other processes to send diagnostic commands and receive responses over an [IPC channel](https://en.wikipedia.org/wiki/Inter-process_communication). This endpoint is called a *diagnostic port*.

The runtime has one diagnostic port open by default as a well-known endpoint:

> * Windows - Named Pipe `\\.\pipe\dotnet-diagnostic-{pid}`
>     
> * Linux and macOS - Unix Domain Socket `{temp}/dotnet-diagnostic-{pid}-{disambiguation_key}-socket`
>     

In the default [diagnostic port configuration](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/diagnostic-port-configuration.md), `dotnet-monitor` uses the `connect` mode. In `connect` mode, the [default diagnostic port](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/diagnostic-port#default-diagnostic-port) of each .NET application is used to send commands for capturing dumps, traces, metrics, etc. **Alternatively, in the** `listen` **mode, the .NET applications themselves connect to** `dotnet-monitor`**, and the collection rules are enabled.** On Windows, update the `dotnet-monitor` setting file as follows:

```json
{
  "DiagnosticPort": {
    "ConnectionMode": "Listen",
    "EndpointName": "dotnet-monitor-pipe"
  },
}
```

With `dotnet-monitor` in `listen` mode, we need to configure our .NET application to connect to `dotnet-monitor` on startup. We can achieve this by specifying an environment variable for our .NET application:

```powershell
$env:DOTNET_DiagnosticPorts="dotnet-monitor-pipe,suspend"
```

## Collection Rules

Collection rules are how `dotnet-monitor` can be configured to automatically gather diagnostic data based on conditions within the discovered processes. Each collection rule consists of up to four properties: Filters, Triggers, Actions, and Limits.

### Filters

`dotnet-monitor` is capable of observing multiple processes simultaneously. Each collection rule can optionally specify a set of filters to determine which processes the rule should be applied to (a process must match all the specified filters). So, let's begin by adding our first rule:

```json
{
  "DiagnosticPort": {
    "ConnectionMode": "Listen",
    "EndpointName": "dotnet-monitor-pipe"
  },
  "CollectionRules": {
    "LargeGCHeapSize": {
      "Filters": [{
        "Key": "ProcessName",
        "Value": "DotNetMonitorSandBox",
        "MatchType": "Exact"
      }],
    }
  }
}
```

The name of our first rule is `LargeGCHeapSize` and [each filter consists of](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/default-process-configuration.md):

* `Key`: Specifies which criteria to match in the process. Can be `ProcessId`, `ProcessName`, or `CommandLine`.
    
* `Value`: The text to match against the process.
    
* `MatchKey`: The type of match to perform. Can be `Exact` or `Contains`.
    

### Triggers

A trigger monitors a specific condition in the target process and sends a notification when that condition is met. It represents the metric and the threshold that will fire an action. The currently supported triggers are:

* Startup
    
* [AspNetRequestCount](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#aspnetrequestcount-trigger)
    
* [AspNetRequestDuration](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#aspnetrequestduration-trigger)
    
* [AspNetResponseStatus](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#aspnetresponsestatus-trigger)
    
* [EventCounter](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#eventcounter-trigger)
    
* [EventMeter](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#eventmeter-trigger-80)
    

Let's continue with our example by adding a trigger to our configuration:

```json
{
  "DiagnosticPort": {
    "ConnectionMode": "Listen",
    "EndpointName": "dotnet-monitor-pipe"
  },
  "CollectionRules": {
    "LargeGCHeapSize": {
      "Filters": [{
        "Key": "ProcessName",
        "Value": "DotNetMonitorSandBox",
        "MatchType": "Exact"
      }],
      "Trigger": {
        "Type": "EventCounter",
        "Settings": {
          "ProviderName": "System.Runtime",
          "CounterName": "gc-heap-size",
          "GreaterThan": 10
        }
      }
    }
  }
}
```

### Actions

Actions enable the execution of an operation or an external program when a specified trigger condition is met. The following actions are currently available:

* [CollectDump](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collectdump-action)
    
* [CollectExceptions](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collectexceptions-action)
    
* [CollectGCDump](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collectgcdump-action)
    
* [CollectLiveMetrics](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collectlivemetrics-action)
    
* [CollectLogs](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collectlogs-action)
    
* [CollectStacks](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collectstacks-action)
    
* [CollectTrace](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#collecttrace-action)
    
* [Execute](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#execute-action)
    
* [LoadProfiler](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#loadprofiler-action)
    
* [SetEnvironmentVariable](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#setenvironmentvariable-action)
    
* [GetEnvironmentVariable](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#getenvironmentvariable-action)
    

In our example, we will use the `CollectGCDump` action. However, this needs an egress provider to determine where to store the action's output (learn more about egress providers [here](https://blog.raulnq.com/dotnet-monitor-authentication-and-egress-providers)):

```json
{
  "Egress": {
    "FileSystem": {
      "monitorFile": {
          "directoryPath": "/artifacts",
          "intermediateDirectoryPath": "/tempArtifacts"
      }
    }
  },
  "DiagnosticPort": {
    "ConnectionMode": "Listen",
    "EndpointName": "dotnet-monitor-pipe"
  },
  "CollectionRules": {
    "LargeGCHeapSize": {
      "Filters": [{
        "Key": "ProcessName",
        "Value": "DotNetMonitorSandBox",
        "MatchType": "Exact"
      }],
      "Trigger": {
        "Type": "EventCounter",
        "Settings": {
          "ProviderName": "System.Runtime",
          "CounterName": "gc-heap-size",
          "GreaterThan": 10
        }
      },
      "Actions": [
        {
          "Type": "CollectGCDump",
          "Settings": {
            "Egress": "monitorFile"
          }
        }
      ]
    }
  }
}
```

### Limits

Limits can optionally be applied to a collection rule to constrain the lifetime of the rule and how often its actions can be run before being throttled. A limit has the following properties:

* `ActionCount`: The number of times the action may be executed before being throttled.
    
* `ActionCountSlidingWindowDuration`: The time window considered for determining whether the action should be throttled based on the number of times it has been executed. If not specified, all action executions will be counted for the entire duration of the rule.
    
* `RuleDuration`: The amount of time before the rule will stop monitoring a process after it has been applied to a process. If not specified, the rule will monitor the process indefinitely.
    

Let's include a limit section to complete our example:

```json
{
  "Egress": {
    "FileSystem": {
      "monitorFile": {
          "directoryPath": "/artifacts",
          "intermediateDirectoryPath": "/tempArtifacts"
      }
    }
  },
  "DiagnosticPort": {
    "ConnectionMode": "Listen",
    "EndpointName": "dotnet-monitor-pipe"
  },
  "CollectionRules": {
    "LargeGCHeapSize": {
      "Filters": [{
        "Key": "ProcessName",
        "Value": "DotNetMonitorSandBox",
        "MatchType": "Exact"
      }],
      "Trigger": {
        "Type": "EventCounter",
        "Settings": {
          "ProviderName": "System.Runtime",
          "CounterName": "gc-heap-size",
          "GreaterThan": 10
        }
      },
      "Actions": [
        {
          "Type": "CollectGCDump",
          "Settings": {
            "Egress": "monitorFile"
          }
        }
      ],
      "Limits": {
        "ActionCount": 2,
        "ActionCountSlidingWindowDuration": "1:00:00"
      }
    }
  }
}
```

## Triggering the Collection Rule

We will use the .NET application here. Since we are using Visual Studio to run the application, navigate to the `launchSettings.json` file and update it as follows to specify the `DOTNET_DiagnosticPorts` environment variable:

```json
{
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:57634",
      "sslPort": 44310
    }
  },
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:5252",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "DOTNET_DiagnosticPorts": "dotnet-monitor-pipe,suspend"
      }
    }
  }
}
```

Run `dotnet-monitor` with the following command:

```powershell
 dotnet monitor collect --no-auth
```

Then, start your .NET application. `dotnet-monitor` will display an output similar to:

```powershell
10:40:29 info: Microsoft.Diagnostics.Tools.Monitor.CollectionRules.CollectionRuleService[40]
      => TargetProcessId:45292 TargetRuntimeInstanceCookie:a3b37540867847e2922764c157068587
      Starting collection rules.
10:40:29 info: Microsoft.Diagnostics.Tools.Monitor.CollectionRules.CollectionRuleService[29]
      => TargetProcessId:45292 TargetRuntimeInstanceCookie:a3b37540867847e2922764c157068587 CollectionRuleName:LargeGCHeapSize
      Collection rule 'LargeGCHeapSize' started.
10:40:29 info: Microsoft.Diagnostics.Tools.Monitor.CollectionRules.CollectionRuleService[35]
      => TargetProcessId:45292 TargetRuntimeInstanceCookie:a3b37540867847e2922764c157068587 CollectionRuleName:LargeGCHeapSize => CollectionRuleTriggerType:EventCounter
      Collection rule 'LargeGCHeapSize' trigger 'EventCounter' started.
10:40:29 info: Microsoft.Diagnostics.Tools.Monitor.CollectionRules.CollectionRuleService[32]
      => TargetProcessId:45292 TargetRuntimeInstanceCookie:a3b37540867847e2922764c157068587
      All collection rules started.
```

There is a [Collection Rules API](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/api/collectionrules.md) to see the state of configured collection rules. Navigate to `https://localhost:52323/processes` to get the PID of the project and then to `https://localhost:52323/collectionrules?pid={pid}` or `https://localhost:52323/collectionrules/LargeGCHeapSize?pid={pid}` to see details about the state of our collection rule. To trigger the rule, simply visit `http://localhost:5252/memory-leak` multiple times, and eventually, the GCDump will be generated at the specified location.

In conclusion, `dotnet-monitor` provides a powerful and flexible way to automatically collect diagnostic data based on specific conditions. By configuring collection rules with filters, triggers, actions, and limits, you can efficiently monitor and troubleshoot your .NET applications, ensuring optimal performance and swift resolution of issues. [Here](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionruleexamples.md) you can find multiple examples. Thank you, and happy coding.