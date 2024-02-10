---
title: "Azure Function Encountered an Error ServiceUnavailable From Host Runtime"
date: 2024-02-10T10:03:16Z
draft: false
tags: "azure, azure-function, python, asyncio"
description: "Syncing triggers (Attempt 6/6)... Error: Encountered an error (ServiceUnavailable) from host runtime."
summary: "
![](/images/azure-function-encountered-an-error-serviceunavailable-from-host-runtime/logo.webp)

I recently had an issue deploying Python 3.10 Azure Function on App Service Plan. The error message was:

```

9:38:43 AM myfunc: Deployment successful. deployer = ms-azuretools-vscode deploymentPath = Functions App ZipDeploy. Extract zip. Remote build.

9:38:56 AM myfunc: Syncing triggers...

9:39:50 AM myfunc: Syncing triggers (Attempt 2/6)...

9:40:51 AM myfunc: Syncing triggers (Attempt 3/6)...

9:42:01 AM myfunc: Syncing triggers (Attempt 4/6)...

9:43:01 AM myfunc: Syncing triggers (Attempt 5/6)...

9:45:12 AM myfunc: Syncing triggers (Attempt 6/6)...

9:45:33 AM: Error: Encountered an error (ServiceUnavailable) from host runtime.

```

Here is my investigation and solution.
"
---

![](/images/azure-function-encountered-an-error-serviceunavailable-from-host-runtime/logo.webp)


I recently had an issue deploying Python 3.10 Azure Function on App Service Plan. The error message was:

```
9:38:43 AM myfunc: Deployment successful. deployer = ms-azuretools-vscode deploymentPath = Functions App ZipDeploy. Extract zip. Remote build.
9:38:56 AM myfunc: Syncing triggers...
9:39:50 AM myfunc: Syncing triggers (Attempt 2/6)...
9:40:51 AM myfunc: Syncing triggers (Attempt 3/6)...
9:42:01 AM myfunc: Syncing triggers (Attempt 4/6)...
9:43:01 AM myfunc: Syncing triggers (Attempt 5/6)...
9:45:12 AM myfunc: Syncing triggers (Attempt 6/6)...
9:45:33 AM: Error: Encountered an error (ServiceUnavailable) from host runtime.
```

## Introduction

On one of by Azure Function App, I had to change the synchronious trigger to asynchronious trigger. This is the [best practice](https://learn.microsoft.com/en-us/azure/azure-functions/python-scale-performance-reference) for case when your function has some io-bound operations.

Here is what I did:
1. I added `async` keyword to the function definition.
2. Add peckages `aiohttp` and `asyncio` to the `requirements.txt` file.
3. Changed all the requests to asynchronious requests using `aiohttp` package.

Here is just an example of the function that can replicate the issue:

#### function_app.py
```python
import azure.functions as func
import asyncio

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.function_name(name="health")
@app.route(route="health")
async def health_function(req: func.HttpRequest) -> func.HttpResponse:
    
    await asyncio.sleep(1)
    return func.HttpResponse("OK")
```

#### requirements.txt
```txt
# DO NOT include azure-functions-worker in this file
# The Python Worker is managed by Azure Functions platform
# Manually managing azure-functions-worker may cause unexpected issues

azure-functions
aiohttp
asyncio
```

After I tried to deploy the function, I got the error message. I tried to deploy the function multiple times but the error was still there.
After 6 attempts to synchronize triggers, the deployment failed with the error message `Error: Encountered an error (ServiceUnavailable) from host runtime.`

## Googling

Similar problem was described [here](https://hungchienhsiang.medium.com/error-encountered-an-error-serviceunavailable-from-host-runtime-9088cb63835c) (no working solution), [here](https://learn.microsoft.com/en-us/answers/questions/1398281/azure-function-app-error-encountered-an-error-(ser)) (solution did not work for me), [here](https://github.com/microsoft/azure-pipelines-tasks/issues/16942) (github issue, no solution) and [here](https://learn.microsoft.com/en-us/answers/questions/1275484/function-app-deployment-error) (no solution).

## Investigation

I have tried many different things to get the internals of the error. Here is the thing that finally gave me the clue.

1. Go to [Kudu service UI](https://learn.microsoft.com/en-us/azure/app-service/resources-kudu) for your app. Please note: This will not work for _Consumption Plan_ but since we are using _App Service Plan_ it will work. Just take your function url and replace `https://<function_name>.azurewebsites.net` with `https://<function_name>.scm.azurewebsites.net`.
2. Switch to the _New Kudu UI_ by adding the 'newui' to the URL. So the URL will look like `https://<function_name>.scm.azurewebsites.net/newui`.
   It will look like this:
   ![](/images/azure-function-encountered-an-error-serviceunavailable-from-host-runtime/kudu_new_ui.png)
3. Go to the File Manager -> /LogFiles/Application/Functions/Host/ and open log file started with date. Scroll down and you will see the following:

```text
[Information] Starting JobHost
[Information] Starting Host (HostId=dedicated-python-test, InstanceId=2462a8c5-981d-45ea-b9b3-f8bd31fb18e1, Version=4.28.4.4, ProcessId=27, AppDomainId=1, InDebugMode=True, InDiagnosticMode=False, FunctionsExtensionVersion=~4)
[Information] Loading functions metadata
[Information] Traceback (most recent call last):
[Information] File "/usr/local/lib/python3.10/runpy.py", line 196, in _run_module_as_main
[Information] return _run_code(code, main_globals, None,
[Information] File "/usr/local/lib/python3.10/runpy.py", line 86, in _run_code
[Information] exec(code, run_globals)
[Information] File "/azure-functions-host/workers/python/3.10/LINUX/X64/azure_functions_worker/__main__.py", line 6, in <module>
[Information] main.main()
[Information] File "/azure-functions-host/workers/python/3.10/LINUX/X64/azure_functions_worker/main.py", line 49, in main
[Information] from ._thirdparty import aio_compat
[Information] File "/azure-functions-host/workers/python/3.10/LINUX/X64/azure_functions_worker/_thirdparty/aio_compat.py", line 8, in <module>
[Information] import asyncio
[Information] File "/home/site/wwwroot/.python_packages/lib/site-packages/asyncio/__init__.py", line 21, in <module>
[Information] from .base_events import *
[Information] File "/home/site/wwwroot/.python_packages/lib/site-packages/asyncio/base_events.py", line 296
[Information] future = tasks.async(future, loop=self)
[Information] ^^^^^
[Error] SyntaxError: invalid syntax
[Error] Exceeded language worker restart retry count for runtime:python. Shutting down and proactively recycling the Functions Host to recover
```
4. Finaly some useful information. The error is `SyntaxError: invalid syntax` and the file that caused the error is `base_events.py`. The error is caused by the line `future = tasks.async(future, loop=self)`. This is the line from the `asyncio` package. The error is caused by the fact that `async` is a keyword in Python 3.10 and it cannot be used as a function name.
5. Here is what helped me to fix the issue: [https://github.com/pyinstaller/pyinstaller/issues/6156](https://github.com/pyinstaller/pyinstaller/issues/6156)

## Solution
Just remove the `asyncio` package from the `requirements.txt` file. The `asyncio` package is included in the Python runtime and you do not need to include it in the `requirements.txt` file.