# How to deploy a secure MCP server on Cloud Run

**Last updated:** Oct 28, 2025
**Written by:** Luke Schlangen, Jack Wotherspoon

-----

## 1\. Introduction

### Overview

In this lab, you will build and deploy a Model Context Protocol (MCP) server. MCP servers are useful for providing LLMs with access to external tools and services. You will configure it as a secure, production-ready service on Cloud Run that can be accessed from multiple clients. Then you will connect to the remote MCP server from Gemini CLI.

### What you'll do

We will use FastMCP to create a zoo MCP server that has two tools: `get_animals_by_species` and `get_animal_details`. FastMCP provides a quick, Pythonic way to build MCP servers and clients.

### What you'll learn

  * Deploy the MCP server to Cloud Run.
  * Secure your server's endpoint by requiring authentication for all requests, ensuring only authorized clients and agents can communicate with it.
  * Connect to your secure MCP server endpoint from Gemini CLI

-----

## 2\. Project Setup

  * If you don't already have a Google Account, you must create a Google Account.
  * Use a personal account instead of a work or school account. Work and school accounts may have restrictions that prevent you from enabling the APIs needed for this lab.
  * Sign-in to the Google Cloud Console.
  * Enable billing in the Cloud Console.
  * Completing this lab should cost less than $1 USD in Cloud resources. You can follow the steps at the end of this lab to delete resources to avoid further charges. New users are eligible for the $300 USD Free Trial.
  * Create a new project or choose to reuse an existing project.

-----

## 3\. Open Cloud Shell Editor

1.  Click this link to navigate directly to Cloud Shell Editor

2.  If prompted to authorize at any point today, click **Authorize** to continue.

3.  If the terminal doesn't appear at the bottom of the screen, open it:

      * Click **View**
      * Click **Terminal**
      * Click **Open new terminal in Cloud Shell Editor**

4.  In the terminal, set your project with this command:

    Format:

    ```bash
    gcloud config set project [PROJECT_ID]
    ```

    Example:

    ```bash
    gcloud config set project lab-project-id-example
    ```

5.  If you can't remember your project id:

      * You can list all your project ids with:
        ```bash
        gcloud projects list | awk '/PROJECT_ID/{print $2}'
        ```

6.  You should see this message:

    ```
    Updated property [core/project].
    ```

    If you see a WARNING and are asked `Do you want to continue (Y/n)?`, then you have likely entered the project ID incorrectly. Press `n`, press `Enter`, and try to run the `gcloud config set project` command again.

-----

## 4\. Enable APIs

1.  In the terminal, enable the APIs:
    ```bash
    gcloud services enable \
      run.googleapis.com \
      artifactregistry.googleapis.com \
      cloudbuild.googleapis.com
    ```
2.  If prompted to authorize, click **Authorize** to continue.
3.  This command may take a few minutes to complete, but it should eventually produce a successful message similar to this one:
    ```
    Operation "operations/acf.p2-73d90d00-47ee-447a-b600" finished successfully.
    ```

-----

## 5\. Prepare your Python project

1.  Create a folder named `mcp-on-cloudrun` to store the source code for deployment:
    ```bash
    mkdir mcp-on-cloudrun && cd mcp-on-cloudrun
    ```
2.  Create a Python project with the `uv` tool to generate a `pyproject.toml` file:
    ```bash
    uv init --description "Example of deploying an MCP server on Cloud Run" --bare --python 3.13
    ```
3.  The `uv init` command creates a `pyproject.toml` file for your project. To view the contents of the file run the following:
    ```bash
    cat pyproject.toml
    ```
4.  The output should look like the following:
    ```toml
    [project]
    name = "mcp-on-cloudrun"
    version = "0.1.0"
    description = "Example of deploying an MCP server on Cloud Run"
    requires-python = ">=3.13"
    dependencies = []
    ```

-----

## 6\. Create the zoo MCP server

To provide valuable context for improving the use of LLMs with MCP, set up a zoo MCP server with FastMCP — a standard framework for working with the Model Context Protocol. FastMCP provides a quick way to build MCP servers and clients with Python. This MCP server provides data about animals at a fictional zoo. For simplicity, we store the data in memory. For a production MCP server, you probably want to provide data from sources like databases or APIs.

1.  Run the following command to add FastMCP as a dependency in the `pyproject.toml` file:

    ```bash
    uv add fastmcp==2.12.4 --no-sync
    ```

    This will add a `uv.lock` file to your project.

2.  Create and open a new `server.py` file for the MCP server source code:

    ```bash
    cloudshell edit ~/mcp-on-cloudrun/server.py
    ```

3.  The `cloudshell edit` command will open the `server.py` file in the editor above the terminal.

4.  Add the following zoo MCP server source code in the `server.py` file:

    ```python
    import asyncio
    import logging
    import os
    from typing import List, Dict, Any
    from fastmcp import FastMCP

    logger = logging.getLogger(__name__)
    logging.basicConfig(format="[%(levelname)s]: %(message)s", level=logging.INFO)

    mcp = FastMCP("Zoo Animal MCP Server 🦁🐧🐻")

    # Dictionary of animals at the zoo
    ZOO_ANIMALS = [
      {
        "species": "lion",
        "name": "Leo",
        "age": 7,
        "enclosure": "The Big Cat Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "lion",
        "name": "Nala",
        "age": 6,
        "enclosure": "The Big Cat Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "lion",
        "name": "Simba",
        "age": 3,
        "enclosure": "The Big Cat Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "lion",
        "name": "King",
        "age": 8,
        "enclosure": "The Big Cat Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "penguin",
        "name": "Waddles",
        "age": 2,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "penguin",
        "name": "Pip",
        "age": 4,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "penguin",
        "name": "Skipper",
        "age": 5,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "penguin",
        "name": "Chilly",
        "age": 3,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "penguin",
        "name": "Pingu",
        "age": 6,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "penguin",
        "name": "Noot",
        "age": 1,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "elephant",
        "name": "Ellie",
        "age": 15,
        "enclosure": "The Pachyderm Sanctuary",
        "trail": "Savannah Heights"
      },
      {
        "species": "elephant",
        "name": "Peanut",
        "age": 12,
        "enclosure": "The Pachyderm Sanctuary",
        "trail": "Savannah Heights"
      },
      {
        "species": "elephant",
        "name": "Dumbo",
        "age": 5,
        "enclosure": "The Pachyderm Sanctuary",
        "trail": "Savannah Heights"
      },
      {
        "species": "elephant",
        "name": "Trunkers",
        "age": 10,
        "enclosure": "The Pachyderm Sanctuary",
        "trail": "Savannah Heights"
      },
      {
        "species": "bear",
        "name": "Smokey",
        "age": 10,
        "enclosure": "The Grizzly Gulch",
        "trail": "Polar Path"
      },
      {
        "species": "bear",
        "name": "Grizzly",
        "age": 8,
        "enclosure": "The Grizzly Gulch",
        "trail": "Polar Path"
      },
      {
        "species": "bear",
        "name": "Barnaby",
        "age": 6,
        "enclosure": "The Grizzly Gulch",
        "trail": "Polar Path"
      },
      {
        "species": "bear",
        "name": "Bruin",
        "age": 12,
        "enclosure": "The Grizzly Gulch",
        "trail": "Polar Path"
      },
      {
        "species": "giraffe",
        "name": "Gerald",
        "age": 4,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "giraffe",
        "name": "Longneck",
        "age": 5,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "giraffe",
        "name": "Patches",
        "age": 3,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "giraffe",
        "name": "Stretch",
        "age": 6,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "antelope",
        "name": "Speedy",
        "age": 2,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "antelope",
        "name": "Dash",
        "age": 3,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "antelope",
        "name": "Gazelle",
        "age": 4,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "antelope",
        "name": "Swift",
        "age": 5,
        "enclosure": "The Tall Grass Plains",
        "trail": "Savannah Heights"
      },
      {
        "species": "polar bear",
        "name": "Snowflake",
        "age": 7,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "polar bear",
        "name": "Blizzard",
        "age": 5,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "polar bear",
        "name": "Iceberg",
        "age": 9,
        "enclosure": "The Arctic Exhibit",
        "trail": "Polar Path"
      },
      {
        "species": "walrus",
        "name": "Wally",
        "age": 10,
        "enclosure": "The Walrus Cove",
        "trail": "Polar Path"
      },
      {
        "species": "walrus",
        "name": "Tusker",
        "age": 12,
        "enclosure": "The Walrus Cove",
        "trail": "Polar Path"
      },
      {
        "species": "walrus",
        "name": "Moby",
        "age": 8,
        "enclosure": "The Walrus Cove",
        "trail": "Polar Path"
      },
      {
        "species": "walrus",
        "name": "Flippers",
        "age": 9,
        "enclosure": "The Walrus Cove",
        "trail": "Polar Path"
      }
    ]

    @mcp.tool()
    def get_animals_by_species(species: str) -> List[Dict[str, Any]]:
      """
      Retrieves a list of animals of a given species.

      Args:
        species: The species of animal to retrieve (e.g. "lion", "penguin", "bear").

      Returns:
        A list of dictionaries, where each dictionary represents an animal
        and contains its name, age, enclosure, and trail.
        Returns an empty list if no animals of the given species are found.
      """
      logger.info(f"Tool call: get_animals_by_species(species=\"{species}\")")
      return [animal for animal in ZOO_ANIMALS if animal["species"] == species.lower()]

    @mcp.tool()
    def get_animal_details(name: str) -> Dict[str, Any]:
      """
      Retrieves the details of a specific animal by its name.

      Args:
        name: The name of the animal to retrieve (e.g. "Leo", "Waddles", "Smokey").

      Returns:
        A dictionary containing the animal's species, name, age, enclosure, and trail.
        Returns an empty dictionary if no animal with the given name is found.
      """
      logger.info(f"Tool call: get_animal_details(name=\"{name}\")")
      for animal in ZOO_ANIMALS:
        if animal["name"].lower() == name.lower():
          return animal
      return {}

    app = mcp.build_app()

    if __name__ == "__main__":
      import uvicorn
      port = int(os.environ.get("PORT", 8080))
      asyncio.run(mcp.run_async(port=port, log_level="info"))
    ```

-----

## 7\. Deploying to Cloud Run

1.  Set the `PROJECT_ID` environment variable in your terminal:
    ```bash
    export PROJECT_ID=$(gcloud config get-value project)
    ```
2.  Set the `REGION` environment variable in your terminal:
    ```bash
    export REGION=us-central1
    ```
    You can also choose a different region, like `us-west1` or `europe-west1`.
3.  Deploy the MCP server to Cloud Run:
    ```bash
    gcloud run deploy mcp-on-cloudrun \
      --source . \
      --region $REGION \
      --allow-unauthenticated \
      --project $PROJECT_ID
    ```
4.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
5.  This command will take a few minutes to complete. It builds your container image using Cloud Build, stores it in Artifact Registry, and then deploys it to Cloud Run.
6.  Once the deployment is complete, you should see a success message that includes the service URL:
    ```
    Service [mcp-on-cloudrun] revision [mcp-on-cloudrun-00001-abc] has been deployed and is serving 100 percent of traffic.
    Service URL: https://mcp-on-cloudrun-xxxxxxxxxx-uc.a.run.app
    ```
7.  Copy the `Service URL` from the output. We will use this in the next step.
8.  Set the `MCP_SERVER_URL` environment variable in your terminal:
    Format:
    ```bash
    export MCP_SERVER_URL=[SERVICE_URL]
    ```
    Example:
    ```bash
    export MCP_SERVER_URL=https://mcp-on-cloudrun-xxxxxxxxxx-uc.a.run.app
    ```
9.  Now that your MCP server is deployed, secure it by removing unauthenticated access:
    ```bash
    gcloud run services update mcp-on-cloudrun \
      --region $REGION \
      --no-allow-unauthenticated \
      --project $PROJECT_ID
    ```
10. When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
11. You should see a success message indicating that the traffic settings have been updated.

-----

## 8\. Add the Remote MCP Server to Gemini CLI

1.  Return to the `mcp-on-cloudrun` directory:
    ```bash
    cd ~/mcp-on-cloudrun
    ```
2.  Install the Gemini CLI and `google-auth` library:
    ```bash
    uv pip install google-generativeai google-auth
    ```
3.  Add the remote MCP server to Gemini CLI, replacing `[MCP_SERVER_URL]` with the URL you copied earlier (or use the environment variable):
    ```bash
    gemini context add $MCP_SERVER_URL
    ```
4.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
5.  You should see a success message that includes the list of tools available on the server:
    ```
    [INFO]: Successfully added context from: https://mcp-on-cloudrun-xxxxxxxxxx-uc.a.run.app
    [INFO]: Tools:
    [INFO]:   - get_animals_by_species
    [INFO]:   - get_animal_details
    ```
6.  You can verify that the context was added by running:
    ```bash
    gemini context list
    ```
7.  You should see the URL of your MCP server in the list.
8.  Now you can use the tools from your remote MCP server in your prompts with Gemini CLI.
9.  Try running the following command to get a list of penguins at the zoo:
    ```bash
    gemini -m 1.5-pro "How many penguins are at the zoo? What are their names?"
    ```
10. You should see the model call the `get_animals_by_species` tool and then provide a response similar to this:
    ```
    There are 6 penguins at the zoo. Their names are Waddles, Pip, Skipper, Chilly, Pingu, and Noot.
    ```
11. Try running the following command to get details about a specific animal:
    ```bash
    gemini -m 1.5-pro "Where can I find Leo the lion?"
    ```
12. You should see the model call the `get_animal_details` tool and then provide a response similar to this:
    ```
    Leo the lion is in The Big Cat Plains enclosure, which is on the Savannah Heights trail.
    ```

-----

## 9\. (Optional) Verify Tool Calls in Server Logs

1.  Open the Cloud Run logs in the Google Cloud Console by navigating to:
      * **Cloud Run**
      * Click on the `mcp-on-cloudrun` service
      * Click on the **LOGS** tab
2.  You should see logs from the MCP server, including the tool calls made by Gemini CLI.
3.  Look for logs similar to this:
    ```
    [INFO]: Tool call: get_animals_by_species(species="penguin")
    [INFO]: Tool call: get_animal_details(name="Leo")
    ```
    This confirms that Gemini CLI is successfully calling the tools on your secure Cloud Run service.

-----

## 10\. (Optional) Add MCP prompt to Server

1.  Open the `server.py` file in the editor:
    ```bash
    cloudshell edit ~/mcp-on-cloudrun/server.py
    ```
2.  Add the following code to the `server.py` file, just before the `app = mcp.build_app()` line:
    ```python
    @mcp.prompt
    def zoo_prompt(request):
      """
      Provides context about the zoo and its trails.
      """
      return """
      The zoo has two trails:
      1. Savannah Heights: This trail features animals from the plains,
         including The Big Cat Plains, The Pachyderm Sanctuary, and
         The Tall Grass Plains.
      2. Polar Path: This trail features animals from colder climates,
         including The Arctic Exhibit, The Grizzly Gulch, and The Walrus Cove.
      """
    ```
3.  Redeploy the MCP server to Cloud Run:
    ```bash
    gcloud run deploy mcp-on-cloudrun \
      --source . \
      --region $REGION \
      --project $PROJECT_ID
    ```
4.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
5.  Wait for the deployment to complete.
6.  Update the context in Gemini CLI:
    ```bash
    gemini context add $MCP_SERVER_URL --update
    ```
7.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
8.  You should see a success message indicating that the context was updated.
9.  Now, try asking a question that uses the prompt context:
    ```bash
    gemini -m 1.5-pro "Which trail are the bears on?"
    ```
10. You should see a response similar to this:
    ```
    The bears are on the Polar Path.
    ```
11. The model was able to answer this question because the `zoo_prompt` provided the necessary context about the trails and enclosures.

### (Optional) Use Gemini Flash Lite for faster responses

1.  Try running the previous command again, but this time using the `gemini-1.5-flash-lite` model:
    ```bash
    gemini "Which trail are the bears on?"
    ```
    (Note: `gemini-1.5-flash-lite` is the default model, so you don't need to specify it with the `-m` flag.)
2.  You should see a similar response, but it should be much faster. This is because `gemini-1.5-flash-lite` is optimized for speed and is a great choice for tool-calling use cases.

-----

## 11\. Conclusion

Congratulations\! You have successfully deployed a secure, production-ready MCP server on Cloud Run and connected to it from Gemini CLI.

You now have a pattern that you can use to provide LLMs with access to your own tools and services, enabling you to build powerful and intelligent applications.

### (Optional) Clean up

To avoid incurring charges to your Google Cloud account for the resources used in this lab, you can delete the resources.

1.  Delete the Cloud Run service:
    ```bash
    gcloud run services delete mcp-on-cloudrun \
      --region $REGION \
      --project $PROJECT_ID
    ```
2.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
3.  Delete the Artifact Registry container image:
    ```bash
    gcloud artifacts docker images delete \
      $REGION-docker.pkg.dev/$PROJECT_ID/cloud-run-source-deploy/mcp-on-cloudrun \
      --delete-tags \
      --project $PROJECT_ID
    ```
4.  When prompted `Do you want to continue (Y/n)?`, type `Y` and press `Enter`.
5.  Remove the context from Gemini CLI:
    ```bash
    gemini context remove $MCP_SERVER_URL
    ```
6.  (Optional) Delete the project:
    If you don't want to keep the project, you can delete it:
    ```bash
    gcloud projects delete $PROJECT_ID
    ```
