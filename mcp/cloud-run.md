# Build and deploy a remote MCP server on Cloud Run

This tutorial shows you how to build and deploy a remote Model Context Protocol (MCP) server on Cloud Run using the streamable HTTP transport. With streamable HTTP transport, the MCP server operates as an independent process that can handle multiple client connections.

## Objectives

In this tutorial, you will:

* Prepare your Python project with the `uv` package manager.
* Create an MCP server for math operations.
* Deploy to Cloud Run.
* Authenticate MCP client.
* Test the remote MCP server.

## Costs

In this document, you use the following billable components of Google Cloud:

* Artifact Registry
* Cloud Build
* Cloud Run

To generate a cost estimate based on your projected usage, use the pricing calculator.

New Google Cloud users might be eligible for a free trial.

## Before you begin

In the Google Cloud console, on the project selector page, select or create a Google Cloud project.

**Roles required to select or create a project**

Note: If you don't plan to keep the resources that you create in this procedure, create a project instead of selecting an existing project. After you finish these steps, you can delete the project, removing all resources associated with the project.

[Go to project selector](link_to_project_selector)

Verify that billing is enabled for your Google Cloud project.

Enable the Artifact Registry, Cloud Run Admin API, and Cloud Build APIs.

**Roles required to enable APIs**

[Enable the APIs](link_to_enable_apis)

Set up your Cloud Run development environment in your Google Cloud project. Make sure you have the appropriate permissions to deploy services, and the **Cloud Run Admin** (`roles/run.admin`) and **Service Account User** (`roles/iam.serviceAccountUser`) roles granted to your account.

Grant the **Cloud Run Invoker** (`roles/run.invoker`) role to your account. This role allows the remote MCP server to access the Cloud Run service.

[Learn how to grant the roles](link_to_grant_roles)

If you are under a domain restriction organization policy restricting unauthenticated invocations for your project, you will need to access your deployed service as described under [Testing private services](#testing-private-services).

Install Uv, a Python package and project manager.

## Prepare your Python project

The following steps describe how to set up your Python project with the `uv` package manager.

Create a folder named `mcp-on-cloudrun` to store the source code for deployment:

```bash
mkdir mcp-on-cloudrun
cd mcp-on-cloudrun
```

Create a Python project with the `uv` tool to generate a `pyproject.toml` file:

```bash
uv init --name "mcp-on-cloudrun" --description "Example of deploying an MCP server on Cloud Run" --bare --python 3.10
```

The `uv init` command creates the following `pyproject.toml` file:

```toml
[project]
name = "mcp-server"
version = "0.1.0"
description = "Example of deploying an MCP server on Cloud Run"
readme = "README.md"
requires-python = ">=3.10"
dependencies = []
```

Create the following additional new files:

* `server.py` for the MCP server source code
* `test_server.py` to test the remote server
* A `Dockerfile` for deploying to Cloud Run

```bash
touch server.py test_server.py Dockerfile
```

Your project directory should contain the following structure:

```
├── mcp-on-cloudrun
│   ├── pyproject.toml
│   ├── server.py
│   ├── test_server.py
│   └── Dockerfile
```

## Create an MCP server for math operations

To provide valuable context for improving the use of LLMs with MCP, set up a math MCP server with FastMCP. FastMCP provides a quick way to build MCP servers and clients with Python.

Follow these steps to create an MCP server for math operations such as addition and subtraction.

Run the following command to add `fastmcp` as a dependency in the `pyproject.toml` file:

```bash
uv add fastmcp==2.8.0 --no-sync
```

Add the following math MCP server source code in the `server.py` file:

```python
import asyncio
import logging
import os

from fastmcp import FastMCP 

logger = logging.getLogger(__name__)
logging.basicConfig(format="[%(levelname)s]: %(message)s", level=logging.INFO)

mcp = FastMCP("MCP Server on Cloud Run")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Use this to add two numbers together.

    Args:
        a: The first number.
        b: The second number.

    Returns:
        The sum of the two numbers.
    """
    logger.info(f">>> 🛠️ Tool: 'add' called with numbers '{a}' and '{b}'")
    return a + b

@mcp.tool()
def subtract(a: int, b: int) -> int:
    """Use this to subtract two numbers.

    Args:
        a: The first number.
        b: The second number.

    Returns:
        The difference of the two numbers.
    """
    logger.info(f">>> 🛠️ Tool: 'subtract' called with numbers '{a}' and '{b}'")
    return a - b

if __name__ == "__main__":
    logger.info(f"🚀 MCP server started on port {os.getenv('PORT', 8080)}")
    # Could also use 'sse' transport, host="0.0.0.0" required for Cloud Run.
    asyncio.run(
        mcp.run_async(
            transport="streamable-http",
            host="0.0.0.0",
            port=os.getenv("PORT", 8080),
        )
    )
```

Include the following code in the `Dockerfile` to use the `uv` tool for running the `server.py` file:

```dockerfile
# Use the official Python image
FROM python:3.13-slim

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Install the project into /app
COPY . /app
WORKDIR /app

# Allow statements and log messages to immediately appear in the logs
ENV PYTHONUNBUFFERED=1

# Install dependencies
RUN uv sync

EXPOSE $PORT

# Run the FastMCP server
CMD ["uv", "run", "server.py"]
```

## Deploy to Cloud Run

You can deploy the MCP server as a container image or as source code:

### Container image

To deploy an MCP server packaged as a container image, follow these instructions.

Create an Artifact Registry repository to store the container image:

```bash
gcloud artifacts repositories create remote-mcp-servers \
--repository-format=docker \
--location=us-central1 \
--description="Repository for remote MCP servers" \
--project=PROJECT_ID
```

Build the container image and push it to Artifact Registry with Cloud Build:

```bash
gcloud builds submit --region=us-central1 --tag us-central1-docker.pkg.dev/PROJECT_ID/remote-mcp-servers/mcp-server:latest
```

Deploy the MCP server container image to Cloud Run:

```bash
gcloud run deploy mcp-server \
--image us-central1-docker.pkg.dev/PROJECT_ID/remote-mcp-servers/mcp-server:latest \
--region=us-central1 \
--no-allow-unauthenticated
```

### Source

You can deploy remote MCP servers to Cloud Run from their sources.

Deploy from source by running the following command:

```bash
gcloud run deploy mcp-server --no-allow-unauthenticated --region=us-central1 --source .
```

## Authenticate MCP client

If you deployed your service with the `--no-allow-unauthenticated` flag, any MCP client that connects to your remote MCP server must authenticate.

Grant the **Cloud Run Invoker** (`roles/run.invoker`) role to the service account. This Identity and Access Management policy binding makes sure that a strong security mechanism is used to authenticate your local MCP client.

Run the Cloud Run proxy to create an authenticated tunnel to the remote MCP server on your local machine:

```bash
gcloud run services proxy mcp-server --region=us-central1
```

If the Cloud Run proxy is not yet installed, this command prompts you to download the proxy. Follow the prompts to download and install the proxy.

Cloud Run authenticates all traffic to `http://127.0.0.1:8080` and forwards requests to the remote MCP server.

## Test the remote MCP server

You test and connect to the remote MCP server by using the FastMCP client and accessing the URL `http://127.0.0.1:8080/mcp`.

To test and invoke the `add` and `subtract` mechanism, follow these steps:

Before running the test server, run the Cloud Run proxy.

Create a test file called `test_server.py` and add the following code:

```python
import asyncio

from fastmcp import Client

async def test_server():
    # Test the MCP server using streamable-http transport.
    # Use "/sse" endpoint if using sse transport.
    async with Client("http://localhost:8080/mcp") as client:
        # List available tools
        tools = await client.list_tools()
        for tool in tools:
            print(f">>> 🛠️  Tool found: {tool.name}")
        # Call add tool
        print(">>> 🪛  Calling add tool for 1 + 2")
        result = await client.call_tool("add", {"a": 1, "b": 2})
        print(f"<<< ✅ Result: {result[0].text}")
        # Call subtract tool
        print(">>> 🪛  Calling subtract tool for 10 - 3")
        result = await client.call_tool("subtract", {"a": 10, "b": 3})
        print(f"<<< ✅ Result: {result[0].text}")

if __name__ == "__main__":
    asyncio.run(test_server())
```

In a new terminal, run the test server:

```bash
uv run test_server.py
```

You should see the following output:

```
 🛠️ Tool found: add
 🛠️ Tool found: subtract
 🪛 Calling add tool for 1 + 2
 ✅ Result: 3
 🪛 Calling subtract tool for 10 - 3
 ✅ Result: 7
```

Success: You successfully deployed a remote MCP server to Cloud Run and tested it using the FastMCP client.

## Clean up

To avoid additional charges to your Google Cloud account, delete all the resources you deployed with this tutorial.

### Delete the project

If you created a new project for this tutorial, delete the project. If you used an existing project and need to keep it without the changes you added in this tutorial, delete resources that you created for the tutorial.

The easiest way to eliminate billing is to delete the project that you created for the tutorial.

To delete the project:

**Caution:** Deleting a project has the following effects:
* Everything in the project is deleted. If you used an existing project for the tasks in this document, when you delete it, you also delete any other work you've done in the project.
* Custom project IDs are lost. When you created this project, you might have created a custom project ID that you want to use in the future. To preserve the URLs that use the project ID, such as an appspot.com URL, delete selected resources inside the project instead of deleting the whole project.
If you plan to explore multiple architectures, tutorials, or quickstarts, reusing projects can help you avoid exceeding project quota limits.

In the Google Cloud console, go to the [Manage resources page](link_to_manage_resources).

In the project list, select the project that you want to delete, and then click **Delete**.

In the dialog, type the project ID, and then click **Shut down** to delete the project.

### Delete tutorial resources

Delete the Cloud Run service you deployed in this tutorial. Cloud Run services don't incur costs until they receive requests.

To delete your Cloud Run service, run the following command:

```bash
gcloud run services delete SERVICE-NAME
```

Replace `SERVICE-NAME` with the name of your service.

You can also delete Cloud Run services from the Google Cloud console.

Remove the `gcloud` default region configuration you added during tutorial setup:

```bash
gcloud config unset run/region
```

Remove the project configuration:

```bash
gcloud config unset project
```
