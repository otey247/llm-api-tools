# Build and Deploy a Remote MCP Server to Google Cloud Run in Under 10 Minutes

**June 17, 2025**
![Hero image for MCP on Cloud Run](https://storage.googleapis.com/gweb-cloudblog-publish/images/mcp_on_cloud_run_hero_image.max-2500x2500.jpg)
Jack Wotherspoon
Developer Advocate

Integrating context from tools and data sources into LLMs can be challenging, which impacts ease-of-use in the development of AI agents. To address this challenge, Anthropic introduced the Model Context Protocol (MCP), which standardizes how applications provide context to LLMs. Imagine you want to build an MCP server for your API to make it available to fellow developers so they can use it as context in their own AI applications. But where do you deploy it? Google Cloud Run could be a great option.

Drawing directly from the official Cloud Run documentation for hosting MCP servers, this guide shows you the straightforward process of setting up your very own remote MCP server. Get ready to transform how you leverage context in your AI endeavors!

## MCP Transports

MCP follows a client-server architecture, and for a while, only supported running the server locally using the `stdio` transport.

![Diagram illustrating MCP client-server architecture](https://storage.googleapis.com/gweb-cloudblog-publish/images/MCP-blog-image.max-2100x2100.png)
[https://modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)

MCP has evolved and now supports remote access transports: `streamable-http` and `sse`. Server-Sent Events (SSE) has been deprecated in favor of Streamable HTTP in the latest MCP specification but is still supported for backwards compatibility. Both of these two transports allow for running MCP servers remotely.

With Streamable HTTP, the server operates as an independent process that can handle multiple client connections. This transport uses HTTP POST and GET requests.

The server **MUST** provide a single HTTP endpoint path (hereafter referred to as the MCP endpoint) that supports both POST and GET methods. For example, this could be a URL like `https://example.com/mcp`.

You can read more about the different transports in the official MCP docs.

## Benefits of running an MCP server remotely

Running an MCP server remotely on Cloud Run can provide several benefits:

*   **Scalability:** Cloud Run is built to rapidly scale out to handle all incoming requests. Cloud Run will scale your MCP server automatically based on demand.
*   **Centralized server:** You can share access to a centralized MCP server with team members through IAM privileges, allowing them to connect to it from their local machines instead of all running their own servers locally. If a change is made to the MCP server, all team members will benefit from it.
*   **Security:** Cloud Run provides an easy way to force authenticated requests. This allows only secure connections to your MCP server, preventing unauthorized access.

**IMPORTANT:** The security benefit is critical. If you don't enforce authentication, anyone on the public internet can potentially access and call your MCP server.

## Prerequisites

*   Python 3.10+
*   Uv (for package and project management, see docs for installation)
*   Google Cloud SDK (`gcloud`)

## Installation

Create a folder, `mcp-on-cloudrun`, to store the code for our server and deployment:

```bash
mkdir mcp-on-cloudrun
cd mcp-on-cloudrun
```

Let’s get started by using `uv` to create a project. Uv is a powerful and fast package and project manager.

```bash
uv init --name "mcp-on-cloudrun" --description "Example of deploying a MCP server on Cloud Run" --bare --python 3.10
```

After running the above command, you should see the following `pyproject.toml`:

```toml
[project]
name = "mcp-on-cloudrun"
version = "0.1.0"
description = "Example of deploying a MCP server on Cloud Run"
requires-python = ">=3.10"
dependencies = []
```

Next, let’s create the additional files we will need: a `server.py` for our MCP server code, a `test_server.py` that we will use to test our remote server, and a `Dockerfile` for our Cloud Run deployment.

```bash
touch server.py test_server.py Dockerfile
```

Our file structure should now be complete:

```
├── mcp-on-cloudrun
│   ├── pyproject.toml
│   ├── server.py
│   ├── test_server.py
│   └── Dockerfile
```

Now that we have our file structure taken care of, let's configure our Google Cloud credentials and set our project:

```bash
gcloud auth login
export PROJECT_ID=<your-project-id>
gcloud config set project $PROJECT_ID
```

## Math MCP Server

LLMs are great at non-deterministic tasks: understanding intent, generating creative text, summarizing complex ideas, and reasoning about abstract concepts. However, they are notoriously unreliable for deterministic tasks – things that have one, and only one, correct answer.

Enabling LLMs with deterministic tools (such as math operations) is one example of how tools can provide valuable context to improve the use of LLMs using MCP.

We will use FastMCP to create a simple math MCP server that has two tools: `add` and `subtract`. FastMCP provides a fast, Pythonic way to build MCP servers and clients.

Add FastMCP as a dependency to our `pyproject.toml`:

```bash
uv add fastmcp==2.6.1 --no-sync
```

Copy and paste the following code into `server.py` for our math MCP server:

```python
import asyncio
import logging
import os
​
from fastmcp import FastMCP 
​
logger = logging.getLogger(__name__)
logging.basicConfig(format="[%(levelname)s]: %(message)s", level=logging.INFO)
​
mcp = FastMCP("MCP Server on Cloud Run")
​
@mcp.tool()
def add(a: int, b: int) -> int:
    """Use this to add two numbers together.
    
    Args:
        a: The first number.
        b: The second number.
    
    Returns:
        The sum of the two numbers.
    """
    logger.info(f">>> Tool: 'add' called with numbers '{a}' and '{b}'")
    return a + b
​
@mcp.tool()
def subtract(a: int, b: int) -> int:
    """Use this to subtract two numbers.
    
    Args:
        a: The first number.
        b: The second number.
    
    Returns:
        The difference of the two numbers.
    """
    logger.info(f">>> Tool: 'subtract' called with numbers '{a}' and '{b}'")
    return a - b
​
if __name__ == "__main__":
    logger.info(f" MCP server started on port {os.getenv('PORT', 8080)}")
    # Could also use 'sse' transport, host="0.0.0.0" required for Cloud Run.
    asyncio.run(
        mcp.run_async(
            transport="streamable-http", 
            host="0.0.0.0", 
            port=os.getenv("PORT", 8080),
        )
    )
```

**Transport**

We are using the `streamable-http` transport for this example as it is the recommended transport for remote servers, but you can also still use `sse` if you prefer as it is backwards compatible.

If you want to use `sse`, you will need to update the last line of `server.py` to use `transport="sse"`.

## Deploying to Cloud Run

Now let's deploy our simple MCP server to Cloud Run.

Copy and paste the below code into our empty `Dockerfile`; it uses `uv` to run our `server.py`:

```dockerfile
# Use the official Python lightweight image
FROM python:3.13-slim
​
# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
​
# Install the project into /app
COPY . /app
WORKDIR /app
​
# Allow statements and log messages to immediately appear in the logs
ENV PYTHONUNBUFFERED=1
​
# Install dependencies
RUN uv sync
​
EXPOSE $PORT
​
# Run the FastMCP server
CMD ["uv", "run", "server.py"]
```

You can deploy directly from source, or by using a container image.

For both options we will use the `--no-allow-unauthenticated` flag to require authentication.

This is important for security reasons. If you don't require authentication, anyone can call your MCP server and potentially cause damage to your system.

### Option 1 - Deploy from source

```bash
gcloud run deploy mcp-server --no-allow-unauthenticated --region=us-central1 --source .
```

### Option 2 - Deploy from a container image

Create an Artifact Registry repository to store the container image.

```bash
gcloud artifacts repositories create remote-mcp-servers \
  --repository-format=docker \
  --location=us-central1 \
  --description="Repository for remote MCP servers" \
  --project=$PROJECT_ID
```

Build the container image and push it to Artifact Registry with Cloud Build.

```bash
gcloud builds submit --region=us-central1 --tag us-central1-docker.pkg.dev/$PROJECT_ID/remote-mcp-servers/mcp-server:latest
```

Deploy our MCP server container image to Cloud Run.

```bash
gcloud run deploy mcp-server \
  --image us-central1-docker.pkg.dev/$PROJECT_ID/remote-mcp-servers/mcp-server:latest \
  --region=us-central1 \
  --no-allow-unauthenticated
```

Once you have completed either option, if your service has successfully deployed you will see a message like the following:

`Service [mcp-server] revision [mcp-server-12345-abc] has been deployed and is serving 100 percent of traffic.`

## Authenticating MCP Clients

Since we specified `--no-allow-unauthenticated` to require authentication, any MCP client connecting to our remote MCP server will need to authenticate.

The official docs for [Host MCP servers on Cloud Run](link_to_host_mcp_servers_on_cloud_run) provides more information on this topic depending on where you are running your MCP client.

For this example, we will run the Cloud Run proxy to create an authenticated tunnel to our remote MCP server on our local machines.

By default, the URL of Cloud Run services requires all requests to be authorized with the **Cloud Run Invoker** (`roles/run.invoker`) IAM role. This IAM policy binding ensures that a strong security mechanism is used to authenticate your local MCP client.

Make sure that you or any team members trying to access the remote MCP server have the `roles/run.invoker` IAM role bound to their IAM principal (Google Cloud account).

NOTE: The following command may prompt you to download the Cloud Run proxy if it is not already installed. Follow the prompts to download and install it.

```bash
gcloud run services proxy mcp-server --region=us-central1
```

You should see the following output:

```
Proxying to Cloud Run service [mcp-server] in project [<YOUR_PROJECT_ID>] region [us-central1]
http://127.0.0.1:8080 proxies to https://mcp-server-abcdefgh-uc.a.run.app
```

All traffic to `http://127.0.0.1:8080` will now be authenticated and forwarded to our remote MCP server.

## Testing the remote MCP server

Let's test and connect to the remote MCP server using the FastMCP client to connect to `http://127.0.0.1:8080/mcp` (note the `/mcp` at the end as we are using the Streamable HTTP transport) and call the `add` and `subtract` tools.

Add the following code to the empty `test_server.py` file:

```python
import asyncio
​
from fastmcp import Client
​
async def test_server():
    # Test the MCP server using streamable-http transport.
    # Use "/sse" endpoint if using sse transport.
    async with Client("http://localhost:8080/mcp") as client:
        # List available tools
        tools = await client.list_tools()
        for tool in tools:
            print(f">>> Tool found: {tool.name}")
        # Call add tool
        print(">>>  Calling add tool for 1 + 2")
        result = await client.call_tool("add", {"a": 1, "b": 2})
        print(f"<<<  Result: {result[0].text}")
        # Call subtract tool
        print(">>>  Calling subtract tool for 10 - 3")
        result = await client.call_tool("subtract", {"a": 10, "b": 3})
        print(f"<<< Result: {result[0].text}")
​
if __name__ == "__main__":
    asyncio.run(test_server())
```

NOTE: Make sure you have the Cloud Run proxy running before running the test server.

In a new terminal run:

```bash
uv run test_server.py
```

You should see the following output:

```
>>> Tool found: add
>>> Tool found: subtract
>>> Calling add tool for 1 + 2
<<< Result: 3
>>> Calling subtract tool for 10 - 3
<<< Result: 7
```

You've done it! You have successfully deployed a remote MCP server to Cloud Run and tested it using the FastMCP client.

Want to learn more about deploying AI applications on Cloud Run? Check out this blog from Google I/O to learn the latest on [Easily Deploying AI Apps to Cloud Run!](link_to_ai_apps_cloud_run_blog)

Continue Reading
*   [Host MCP servers on Cloud Run](link_to_host_mcp_servers_on_cloud_run)
*   [MCP server for helping deploy applications to Cloud Run](link_to_mcp_server_for_cloud_run)

Posted in
Developers & Practitioners
