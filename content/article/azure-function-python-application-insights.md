---
title: "Azure Function (Python) Application Insights"
date: 2024-10-05T11:32:19Z
draft: false
keywords: "azure, azure function, python, logging, opentelimetry, application insights"
description: "How to configure Application Insights logging for Python Azure Function"
Summary: "
I have some Python Azure Functions and recently I decided to review and improve the logging mechanism for them (Azure Application Insights integration). What I wanted to acheave is to have visual spans for individual
function calls that can be nested and give the better idea of the function call behavior.

![](/images/azure-function-python-application-insights/application-insights-spans.png)"
---

I have some Python Azure Functions and recently I decided to review and improve the logging mechanism for them (Azure Application Insights integration). What I wanted to acheave is to have visual spans for individual
function calls that can be nested and give the better idea of the function call behavior.

![](/images/azure-function-python-application-insights/application-insights-spans.png)

### Long story short (Solution):

Use [azure-monitor-opentelemetry](https://pypi.org/project/azure-monitor-opentelemetry/) python package:

``` python

import logging
import requests
import azure.functions as func
import opentelemetry.trace
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor()
tracer: opentelemetry.trace.Tracer = opentelemetry.trace.get_tracer(__name__)
app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.timer_trigger(schedule="0 * * * * *", arg_name="timer")
async def timer_trigger(timer: func.TimerRequest) -> None:
    logging.info("Call: TimerTrigger Started")
    await RunBusinessLogic()
    logging.info("Call: TimerTrigger Ended")

@app.function_name(name="GetRecord")
@app.route(route="GetRecord", methods=["GET"])
async def GetRecord(req: func.HttpRequest) -> func.HttpResponse:
    logging.info("Call: GetRecord Started")
    await RunBusinessLogic()
    logging.info("Call: GetRecord Ended")
    return func.HttpResponse("Done", status_code=200)

async def RunBusinessLogic() -> None:
    with tracer.start_as_current_span("My Business Logic"):
        with tracer.start_as_current_span("HTTP Request to Google"):
            google_resp = requests.get(url='https://google.com')
            logging.info(f"google response status code {google_resp.status_code}")
        with tracer.start_as_current_span("HTTP Request to Microsoft"):
            msft_resp = requests.get(url='https://www.microsoft.com')
            logging.info(f"msft response status code {msft_resp.status_code}")

```

And make sure you set `OTEL_LOGS_EXPORTER` Environment Variable (App Setting) to `None` as it is described here: [Duplicate trace logs in Azure Functions](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/opentelemetry-troubleshooting-python#duplicate-trace-logs-in-azure-functions). If you do not do this your trace logs will be duplicated.

Here is how it should be:
![](/images/azure-function-python-application-insights/application-insights-plain.png)

That's it.

### Longer story (also short) with links:

There is no [automatic dependency and transaction diagnostic auto collection](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-dependencies#dependency-auto-collection). There are also no samples of [how to track dependencies manually](https://learn.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics#trackdependency) for python Azure Functions. [The Azure Functions Python Developer Reference Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python?tabs=get-started%2Casgi%2Capplication-level&pivots=python-mode-decorators#log-custom-telemetry) describes how to Log Custom Telimetry using OpenCensus library. You can
also find samples of how to use this library [Monitor a distributed system by using Application Insights and OpenCensus](https://learn.microsoft.com/en-us/azure/architecture/guide/devops/monitor-with-opencensus-application-insights). However... you should not use it because [Azure Monitor support for OpenCensus will end on 30 September 2024 - transition to using Azure Monitor OpenTelemetry Python Distro](https://azure.microsoft.com/en-us/updates/python-opencensus-retirement/#:~:text=On%2030%20September%202024%2C%20we,a%20single%20line%2Dof%2Dcode). Here is the guide for [Migrating from OpenCensus Python SDK and Azure Monitor OpenCensus exporter for Python to Azure Monitor OpenTelemetry Python Distro](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-python-opencensus-migrate).