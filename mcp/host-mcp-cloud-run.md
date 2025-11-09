# Host MCP servers on Cloud Run

This guide shows how to host a Model Context Protocol (MCP) server with streamable HTTP transport on Cloud Run, and provides guidance for authenticating MCP clients. If you're new to MCP, read the following resources:

* [What is the Model Context Protocol (MCP)?](link_to_what_is_mcp)
* [What is the MCP and how does it work?](link_to_how_mcp_works)

MCP is an open protocol that standardizes how AI agents interact with their environment. The AI agent hosts an MCP client, and the tools and resources it interacts with are MCP servers. The MCP client can communicate with the MCP server over two distinct transport types:

* Server Sent Events (SSE) or Streamable HTTP
* Standard Input/Output (stdio)

You can host MCP clients and servers on the same local machine, host an MCP client locally and have it communicate with remote MCP servers hosted on a cloud platform like Cloud Run, or host both the MCP client and server on a cloud platform.

Cloud Run supports hosting MCP servers with streamable HTTP transport, but not MCP servers with stdio transport.

The following diagram shows how the MCP client takes the AI agent's intent and sends a standardized request to MCP servers, specifying the tool to be executed. After the MCP server executes the action and retrieves the results, the MCP server returns the result back to the MCP client in a consistent format.

*Figure 1. The MCP server hosted on Cloud Run interacts with the MCP client, which interacts with the AI agent.*

The guidance on this page applies if you are developing your own MCP server or if you are using an existing MCP server.

* If you are developing your own MCP server, we recommended that you use an MCP server SDK, such as the official language SDKs (TypeScript, Python, Go, Kotlin, Java, C#, Ruby, or Rust) or [FastMCP](link_to_fastmcp).
* If you are using an existing MCP server, find a list of official and community MCP servers on the [MCP servers GitHub repository](link_to_mcp_servers_repo). Docker Hub also provides a curated list of MCP servers.

## Before you begin

In the Google Cloud console, on the project selector page, select or create a Google Cloud project.

**Roles required to select or create a project**

Note: If you don't plan to keep the resources that you create in this procedure, create a project instead of selecting an existing project. After you finish these steps, you can delete the project, removing all resources associated with the project.

[Go to project selector](link_to_project_selector)

Verify that billing is enabled for your Google Cloud project.

Set up your Cloud Run development environment in your Google Cloud project. Ensure you have the appropriate permissions to deploy services, and the **Cloud Run Admin** (`roles/run.admin`) and **Service Account User** (`roles/iam.serviceAccountUser`) roles granted to your account.

[Learn how to grant the roles](link_to_grant_roles)

## Host remote SSE or streamable HTTP MCP servers

MCP servers that use the Server-sent events (SSE) or streamable HTTP transport can be hosted remotely from their MCP clients.

To deploy this type of MCP server to Cloud Run, you can deploy the MCP server as a container image or as source code (commonly Node.js or Python), depending on how the MCP server is packaged.

### Container images

Remote MCP servers distributed as container images are web servers that listen for HTTP requests on a specific port, which means they adhere to Cloud Run's container runtime contract and can be deployed to a Cloud Run service.

To deploy an MCP server packaged as a container image, you need to have the URL of the container image and the port on which it expects to receive requests. These can be deployed using the following `gcloud` CLI command:

```bash
gcloud run deploy --image IMAGE_URL --port PORT
```

Replace:

* `IMAGE_URL` with the container image URL, for example `us-docker.pkg.dev/cloudrun/container/mcp`.
* `PORT` with the port it listens on, for example `3000`.

### Sources

Remote MCP servers that are not provided as container images can be deployed to Cloud Run from their sources, notably if they are written in Node.js or Python.

Clone the Git repository of the MCP server:

```bash
git clone https://github.com/ORGANIZATION/REPOSITORY.git
```

Navigate to the root of the MCP server:

```bash
cd REPOSITORY
```

Deploy to Cloud Run with the following `gcloud` CLI command:

```bash
gcloud run deploy --source .
```

After you deploy your HTTP MCP server to Cloud Run, the MCP server gets a HTTPS URL and communication can use Cloud Run's built in support for HTTP response streaming.

## Authenticate MCP clients

Depending on where you hosted the MCP client, see the section that is relevant for you:

* [Authenticate local MCP clients](#authenticate-local-mcp-clients)
* [Authenticate MCP clients running on Cloud Run](#authenticate-mcp-clients-running-on-cloud-run)

### Authenticate local MCP clients

If the AI agent hosting the MCP client runs on a local machine, use one of the following methods to authenticate the MCP client:

* IAM invoker permission
* OIDC ID token

For more information, refer to the MCP specification on [Authentication](link_to_mcp_auth_spec).

#### IAM invoker permission

By default, the URL of Cloud Run services requires all requests to be authorized with the **Cloud Run Invoker** (`roles/run.invoker`) IAM role. This IAM policy binding ensures that a strong security mechanism is used to authenticate your local MCP client.

After deploying your MCP server to a Cloud Run service in a region, run the Cloud Run proxy on your local machine to securely expose the remote MCP server to your client using your own credentials:

```bash
gcloud run services proxy MCP_SERVER_NAME --region REGION --port=3000
```

Replace:

* `MCP_SERVER_NAME` with the name of your Cloud Run service.
* `REGION` with the Google Cloud region where you deployed your service. For example, `europe-west1`.

The Cloud Run proxy command creates a local proxy on port 3000 that forwards requests to the remote MCP server and injects your identity.

Update the MCP configuration file of your MCP client with the following:

```json
{
  "mcpServers": {
    "cloud-run": {
      "url": "http://localhost:3000/sse"
    }
  }
}
```

If your MCP client does not support the `url` attribute, use the `mcp-remote` npm package:

```json
{
  "mcpServers": {
    "cloud-run": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://localhost:3000/sse"
      ]
    }
  }
}
```

#### OIDC ID token

Depending on whether the MCP client exposes headers or uses a way of providing a custom authenticated transport, you might consider authenticating the MCP client with an OIDC ID token.

You can use various Google authentication libraries to get an ID token from the runtime environment, for example the Google Auth Library for Python. This token must have the correct audience claim that matches the receiving service's `*.run.app` URL, unless you use custom audiences. You must also include the ID token in client requests, such as `Authorization: Bearer <token value>`.

If the MCP client does not expose either headers or transport, use a different authentication method.

### Authenticate MCP clients running on Cloud Run

If the AI agent hosting the MCP client runs on Cloud Run, use one of the following methods to authenticate the MCP client:

* Deploy as a sidecar
* Authenticate service to service
* Use Cloud Service Mesh

#### Deploy the MCP server as a sidecar

The MCP server can be deployed as a sidecar where the MCP client runs.

No specific authentication is required for this use case, since the MCP client and MCP server are on the same instance. The client can connect to the MCP server using a port on `http://localhost:PORT`. Replace `PORT` with a different port than the one used to send requests to the Cloud Run service.

#### Authenticate service to service

If the MCP server and MCP client run as distinct Cloud Run services, see [Authenticating service-to-service](link_to_service_to_service_auth).

#### Use Cloud Service Mesh

An agent hosting an MCP client can connect to a remote MCP server using Cloud Service Mesh.

You can configure the MCP server service to have a short name on the mesh, and the MCP client can communicate to the MCP server using the short name `http://mcp-server`. Authentication is managed by the mesh.

## What's next

* [Host AI agents on Cloud Run](link_to_host_ai_agents).
* Follow a tutorial to build and deploy a remote MCP server to Cloud Run.
* Follow these MCP codelabs:
    * [How to deploy a secure MCP server on Cloud Run.](link_to_secure_mcp_codelab)
    * [Use an MCP server on Cloud Run with an ADK agent.](link_to_mcp_adk_codelab)
