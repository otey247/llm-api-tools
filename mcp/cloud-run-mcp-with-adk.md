# Use MCP Server on Cloud Run with an ADK Agent

**Last updated:** Oct 28, 2025
**Written by:** Luke Schlangen, Jack Wotherspoon

-----

## 1\. Introduction

### Overview

In this lab, you will build and deploy a simple Agent Development Kit (ADK) agent. This agent will connect to the secure Model Context Protocol (MCP) server you deployed in the previous lab. You will configure the agent to securely access the MCP server on Cloud Run using a service account.

### What you'll do

  * Create a new ADK agent project.
  * Add the remote MCP server as a tool provider for the agent.
  * Create a service account with permissions to invoke the secure Cloud Run service.
  * Deploy the ADK agent to Cloud Run.
  * Interact with the deployed agent.

### What you'll learn

  * How to configure an ADK agent to securely connect to a remote MCP server.
  * How to use service accounts to authenticate between Cloud Run services.
  * How to deploy an ADK agent to Cloud Run.

-----

## 2\. Before you begin

This lab builds on the previous lab, "How to deploy a secure MCP server on Cloud Run".

  * **You must have completed the previous lab** and have the secure MCP server running on Cloud Run.
  * You will need the **Service URL** of your deployed `mcp-on-cloudrun` service from the previous lab.

-----

## 3\. Project Setup

1.  If you don't already have a Google Account, you must create one.
2.  Sign in to the Google Cloud Console.
3.  Ensure billing is enabled for your project.
4.  You will continue working in the same project you used for the previous lab.

-----

## 4\. Open Cloud Shell Editor

1.  In the Google Cloud Console, activate Cloud Shell by clicking the "Activate Cloud Shell" icon in the top-right toolbar.
2.  If you're not already in the editor, click **Open Editor**.
3.  If a terminal doesn't appear at the bottom, click **View \> Terminal \> Open new terminal in Cloud Shell Editor**.
4.  Set your project ID in the terminal:
    ```bash
    gcloud config set project [PROJECT_ID]
    ```
    (Replace `[PROJECT_ID]` with your actual project ID).
5.  You should see a message: `Updated property [core/project].`

-----

## 5\. Prepare your ADK Agent project

1.  Navigate out of the previous lab's directory.

    ```bash
    cd ~
    ```

2.  Create a new folder for your ADK agent:

    ```bash
    mkdir adk-agent-on-cloudrun && cd adk-agent-on-cloudrun
    ```

3.  Create a Python project using `uv`:

    ```bash
    uv init --description "ADK agent on Cloud Run" --bare --python 3.13
    ```

4.  Add the `google-cloud-adk` library as a dependency:

    ```bash
    uv add google-cloud-adk==0.1.1 --no-sync
    ```

5.  Create and open a new `main.py` file for the agent's source code:

    ```bash
    cloudshell edit ~/adk-agent-on-cloudrun/main.py
    ```

6.  Add the following basic ADK agent code to `main.py`:

    ```python
    import uvicorn
    import os
    import logging
    from typing import Awaitable, Callable

    from adk.api.fastapi import build_app, ZsTask
    from adk.services import user_service
    from adk.domain.agent import Agent
    from adk.domain.conversation import Conversation
    from adk.domain.message import Message
    from adk.domain.tool import RemoteToolProvider
    from adk.domain.user import User

    logger = logging.getLogger(__name__)
    logging.basicConfig(level=logging.INFO, format="[%(levelname)s]: %(message)s")

    # Get the MCP Server URL from an environment variable
    MCP_SERVER_URL = os.environ.get("MCP_SERVER_URL")
    if not MCP_SERVER_URL:
      raise ValueError("MCP_SERVER_URL environment variable not set.")

    # Create a RemoteToolProvider to connect to the MCP server
    zoo_mcp_server = RemoteToolProvider(
        name="zoo_mcp_server",
        url=MCP_SERVER_URL,
        # Authentication is handled automatically when deployed on Cloud Run
        # with an appropriate service account.
    )

    class ZooAgent(Agent):
      """A simple agent that uses the remote zoo MCP server."""

      async def __call__(
          self,
          conversation: Conversation,
          sender: User,
          context_callback: Callable[[], Awaitable[None]] | None = None,
      ) -> Message | None:
        """Invokes the agent."""
        logger.info("ZooAgent invoked.")
        
        # Get the latest user message
        user_message = conversation.get_last_user_message()
        if not user_message:
          return user_service.create_message(
              "Hello! Ask me about the zoo animals."
          )

        # Use Gemini 1.5 Flash Lite for a fast response
        model = "gemini-1.5-flash-lite"
        
        # Generate a response using the model and the remote tools
        response = await self.generate_async(
            conversation,
            model=model,
            tool_providers=[zoo_mcp_server],
        )
        
        logger.info(f"Generated response: {response.text}")
        return response

    # Instantiate the agent
    agent = ZooAgent(
        id="zoo-agent",
        name="Zoo Agent",
        description="An agent that can answer questions about the zoo.",
    )

    # Build the FastAPI app
    app = build_app(agent=agent, allow_unauth_users=True)

    if __name__ == "__main__":
      port = int(os.environ.get("PORT", 8080))
      uvicorn.run(app, host="0.0.0.0", port=port, log_level="info")
    ```

-----

## 6\. Create a Service Account

To allow your new ADK agent service to securely communicate with your existing MCP server service, you'll create a dedicated service account. This account will have the "Cloud Run Invoker" role, granting it permission to call the MCP server.

1.  Create a new service account named `adk-agent-invoker`:

    ```bash
    gcloud iam service-accounts create adk-agent-invoker \
      --description="Service account for ADK agent to invoke MCP server" \
      --display-name="ADK Agent Invoker"
    ```

2.  Set environment variables for your `PROJECT_ID` and the new service account email:

    ```bash
    export PROJECT_ID=$(gcloud config get-value project)
    export AGENT_SA_EMAIL="adk-agent-invoker@${PROJECT_ID}.iam.gserviceaccount.com"
    ```

3.  Grant the new service account the **Cloud Run Invoker** role *specifically for your MCP server service*. This follows the principle of least privilege.

    ```bash
    gcloud run services add-iam-policy-binding mcp-on-cloudrun \
      --member="serviceAccount:${AGENT_SA_EMAIL}" \
      --role="roles/run.invoker" \
      --region=us-central1 \
      --project=$PROJECT_ID
    ```

    *(Note: If you deployed your `mcp-on-cloudrun` service to a different region, change `us-central1` accordingly.)*

4.  You should see a success message showing the updated policy.

-----

## 7\. Deploy the ADK Agent

Now you're ready to deploy the agent. During deployment, you'll specify the service account you just created and pass the MCP server's URL as an environment variable.

1.  Set the `REGION` environment variable (use the same region as before):

    ```bash
    export REGION=us-central1
    ```

2.  Retrieve your `mcp-on-cloudrun` service URL and store it in an environment variable.

    ```bash
    export MCP_SERVER_URL=$(gcloud run services describe mcp-on-cloudrun --region $REGION --format 'value(status.url)')
    ```

3.  Verify the variable is set:

    ```bash
    echo $MCP_SERVER_URL
    ```

    You should see your MCP server's URL, e.g., `https://mcp-on-cloudrun-xxxxxxxxxx-uc.a.run.app`.

4.  Deploy the ADK agent to Cloud Run:

    ```bash
    gcloud run deploy adk-agent-on-cloudrun \
      --source . \
      --region $REGION \
      --allow-unauthenticated \
      --project $PROJECT_ID \
      --service-account $AGENT_SA_EMAIL \
      --set-env-vars="MCP_SERVER_URL=${MCP_SERVER_URL}"
    ```

      * `--service-account`: Attaches the `adk-agent-invoker` service account to this new service.
      * `--set-env-vars`: Passes the MCP server's URL to the agent's code.

5.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.

6.  This will take a few minutes. Once complete, you'll get a new `Service URL` for your ADK agent.

-----

## 8\. Interact with your Agent

Your agent is now deployed and ready to talk to. You can interact with it directly using `curl` or by visiting its URL in a browser.

1.  Copy the `Service URL` from the deployment output of your `adk-agent-on-cloudrun` service.
2.  Store it in an environment variable:
    ```bash
    export ADK_AGENT_URL=[YOUR_ADK_AGENT_URL]
    ```
    Example:
    ```bash
    export ADK_AGENT_URL=https://adk-agent-on-cloudrun-xxxxxxxxxx-uc.a.run.app
    ```
3.  Send a test message to your agent using `curl`:
    ```bash
    curl -X POST -H "Content-Type: application/json" \
      -d '{"conversation": {}, "sender": {"id": "test-user"}}' \
      "${ADK_AGENT_URL}/v1/agent:run"
    ```
4.  You should receive the agent's greeting response:
    ```json
    {"text":"Hello! Ask me about the zoo animals.","scopes":[]}
    ```
5.  Now, ask it a question that requires using the remote tools from the MCP server:
    ```bash
    curl -X POST -H "Content-Type: application/json" \
      -d '{
            "conversation": {
              "messages": [
                {"user": {"id": "test-user"}, "text": "How many penguins are at the zoo?"}
              ]
            },
            "sender": {"id": "test-user"}
          }' \
      "${ADK_AGENT_URL}/v1/agent:run"
    ```
6.  This time, the agent will contact the MCP server to get the information. You should get a response like this:
    ```json
    {"text":"There are 6 penguins at the zoo: Waddles, Pip, Skipper, Chilly, Pingu, and Noot.","scopes":[]}
    ```

-----

## 9\. (Optional) Verify Logs

You can check the logs for both services to see the communication.

1.  **Check the ADK Agent logs:**

      * Go to the Cloud Run page in the Google Cloud Console.
      * Click on your `adk-agent-on-cloudrun` service.
      * Go to the **LOGS** tab.
      * You should see logs for `ZooAgent invoked.` and the `Generated response:`.

2.  **Check the MCP Server logs:**

      * Go back to the Cloud Run services list.
      * Click on your `mcp-on-cloudrun` service.
      * Go to the **LOGS** tab.
      * You should see the log entry from your `server.py` file:
        `[INFO]: Tool call: get_animals_by_species(species="penguin")`

    This confirms your ADK agent successfully and securely called your MCP server.

-----

## 10\. (Optional) Use the ADK Chat UI

The `google-cloud-adk` library also provides a simple, built-in chat interface for testing.

1.  Open the `Service URL` for your `adk-agent-on-cloudrun` service in your web browser.
2.  You will see a simple chat interface.
3.  Try asking questions:
      * "Hello"
      * "Where can I find Leo the lion?"
      * "Which trail are the bears on?" (This will use the prompt context from the MCP server if you added it in the previous lab).

You now have a fully functional, secure, and interacting two-service agent system running on Cloud Run.

-----

## 11\. Conclusion

Congratulations\! You have successfully built and deployed an ADK agent that securely connects to a remote MCP server using a service account for authentication between Cloud Run services.

You have learned a powerful, secure, and scalable pattern for building complex agents that can leverage tools from multiple microservices.

### Clean up

To avoid incurring charges, delete the resources you created.

1.  Delete the ADK agent Cloud Run service:

    ```bash
    gcloud run services delete adk-agent-on-cloudrun \
      --region $REGION \
      --project $PROJECT_ID
    ```

    (Type `Y` when prompted).

2.  Delete the MCP server Cloud Run service:

    ```bash
    gcloud run services delete mcp-on-cloudrun \
      --region $REGION \
      --project $PROJECT_ID
    ```

    (Type `Y` when prompted).

3.  Delete the service account:

    ```bash
    gcloud iam service-accounts delete $AGENT_SA_EMAIL
    ```

    (Type `Y` when prompted).

4.  Delete the container images from Artifact Registry:

    ```bash
    gcloud artifacts docker images delete \
      $REGION-docker.pkg.dev/$PROJECT_ID/cloud-run-source-deploy/adk-agent-on-cloudrun \
      --delete-tags --quiet
      
    gcloud artifacts docker images delete \
      $REGION-docker.pkg.dev/$PROJECT_ID/cloud-run-source-deploy/mcp-on-cloudrun \
      --delete-tags --quiet
    ```

5.  (Optional) Delete the entire project:

    ```bash
    gcloud projects delete $PROJECT_ID
    ```
