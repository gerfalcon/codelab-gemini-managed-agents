author: Google Antigravity Team
summary: Learn how to build custom managed agents, customize persistent environments, and secure outbound connections on the Gemini API.
id: gemini-managed-agents
categories: gemini,ai,agents
environments: Web
status: Draft
feedback link: https://github.com/gerfalcon/gemini-managed-agents/issues

# Building Custom Managed Agents with the Gemini API

## Overview and Prerequisites
Duration: 0:05

Managed agents on the Gemini API let you extend the Antigravity agent with your own instructions, skills, and data. You can customize the agent inline at interaction time, or save the configuration as a managed agent you invoke by ID.

In this codelab, you will build and manage custom agents using persistent remote environments. You will learn to load files declaratively, persist installed packages, lock down outbound network calls, inject authentication headers securely, and manage the lifecycle of registered agents.

### What you'll learn

- How to customize the Antigravity agent inline with custom instructions
- How to declare environment sources (Git, Cloud Storage, and inline files)
- How to reuse and verify persistent remote execution sandboxes
- How to lock down sandbox network outbound requests using allowlists
- How to inject authorization credentials using egress header transforms
- How to register, list, fetch, and delete custom Managed Agents

### What you'll need

- A machine with Python 3.10+ installed
- A Gemini API key (set as the `GEMINI_API_KEY` environment variable)
- Installation of the new Gemini Python SDK (`google-genai`)
- Roughly 60 minutes

Let's get started by setting up the Python environment.

## Inline Customization
Duration: 0:07

The fastest way to build a custom agent is to pass your configuration inline while creating a new interaction with the Gemini API. You don't need any registration step first.

You can customize the agent inline by overriding:
1. **System instructions** using `system_instruction` to shape behavior
2. **Tools** to override the default toolsets or specify custom tools
3. **Environment sources** to mount files directly into the execution workspace

### Step 1: Create a script for inline configuration

Let's write a simple Python script to pass inline instructions and print the agent output.

Create a file named `01_inline_customization.py` and add the following code:

```python
from google import genai

# Initialize the Gemini client. It automatically picks up GEMINI_API_KEY from your environment.
client = genai.Client()

print("Sending interaction with inline customization...")

# Create an interaction using the base Antigravity agent
interaction = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="What is your persona, and what format should you export reports in?",
    system_instruction="You are a data analyst. Always include visualizations and export results as PDF.",
)

print("\n--- Agent Response ---")
print(interaction.output_text)
```

### Step 2: Run the script

Make sure your API key is exported, then execute the script:

```bash
export GEMINI_API_KEY="your_api_key_here"
python 01_inline_customization.py
```

You should see an output confirming the agent's persona:

```console
Sending interaction with inline customization...

--- Agent Response ---
I am a data analyst. Whenever I generate reports, I will ensure they contain clear visualizations and are exported in PDF format.
```

Positive
: Inline configurations are ideal for fast prototyping and one-off tasks where you don't need to persist or reuse the agent settings.

## Declarative Environment Sources
Duration: 0:08

While you can pass configuration inline, we recommend organizing your agent's files in a structured workspace directory. You can mount three types of declarative sources into your environment:
- **Git Repository (`repository`)**: Clones a Git repo into the sandbox
- **Cloud Storage (`gcs`)**: Copies files/directories from Google Cloud Storage
- **Inline content (`inline`)**: Writes text content to a target file in the sandbox

Let's set up a custom environment mounting a repository, GCS bucket data, and an inline file.

### Write the script

Create `02_environment_sources.py`:

```python
from google import genai

client = genai.Client()

print("Creating sandbox with multiple declarative sources...")

# Create an interaction that mounts:
# 1. A public Git repository (target: /workspace/spoon-knife)
# 2. A public GCS bucket with state data (target: /workspace/gcs-data)
# 3. An inline metadata/notes file (target: /workspace/notes/readme.md)
interaction = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="List the files in /workspace/spoon-knife, read /workspace/notes/readme.md, and summarize what you find.",
    environment={
        "type": "remote",
        "sources": [
            {
                "type": "repository",
                "source": "https://github.com/octocat/Spoon-Knife",
                "target": "/workspace/spoon-knife",
            },
            {
                "type": "gcs",
                "source": "gs://cloud-samples-data/bigquery/us-states/",
                "target": "/workspace/gcs-data",
            },
            {
                "type": "inline",
                "target": "/workspace/notes/readme.md",
                "content": "# Project Notes\n\n- Analyze state population data.\n- Create visualizations.\n",
            },
        ],
    },
)

print("\n--- Agent Response ---")
print(interaction.output_text)
```

Run the script:

```bash
python 02_environment_sources.py
```

The agent clones the repository, downloads the GCS data, mounts the inline README, and presents a summary of the files in your workspace.

Negative
: When specifying target paths, you cannot mount files to the environment root `/`. You must always specify a sub-directory target (e.g., `/workspace/...` or `/backend-app`).

## Reusing Persistent Environments
Duration: 0:07

Environments are decoupled from the interaction context. When you create an interaction with `"environment": "remote"`, the Gemini API provisions a Linux sandbox and returns an `environment_id`. 

You can capture this ID and reuse the environment across subsequent calls. Any changes, file creations, or installed packages will persist.

### Write the multi-turn persistent script

Create `03_persistent_environment.py`:

```python
from google import genai

client = genai.Client()

# Step 1: Create a fresh sandbox environment, install dependencies
print("Interaction 1: Creating fresh sandbox and installing matplotlib...")
interaction_1 = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="Install pandas and matplotlib. Verify they work and print the installed versions.",
    environment="remote",
)
print(interaction_1.output_text)

env_id = interaction_1.environment_id
print(f"\nCaptured Environment ID: {env_id}\n")

# Step 2: Reuse the same environment, verify package persistence
print("Interaction 2: Reusing the same environment...")
interaction_2 = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="Create a simple line plot using matplotlib, save it to /workspace/plot.png, and list the files in /workspace.",
    environment=env_id,
    previous_interaction_id=interaction_1.id,
)

print("\n--- Agent Response (Interaction 2) ---")
print(interaction_2.output_text)
```

Run the script:

```bash
python 03_persistent_environment.py
```

### Expected Output

During the second interaction, you should see that matplotlib is already available without reinstalling:

```console
Interaction 1: Creating fresh sandbox and installing matplotlib...
Installing pandas and matplotlib...
pandas version: 2.2.x, matplotlib version: 3.8.x verified successfully.

Captured Environment ID: env_abc123xyz...

Interaction 2: Reusing the same environment...

--- Agent Response (Interaction 2) ---
Plot created and saved to /workspace/plot.png.
Workspace files:
- /workspace/plot.png
```

Positive
: Persisting environments avoids rebuilding work environments and enables developers to run interactive multi-turn workflows where intermediate state matters.

## Security & Network Allowlists
Duration: 0:08

By default, sandboxes have unrestricted outbound internet access. For production workloads, you should restrict outbound network traffic to trusted APIs and endpoints.

You can configure a network `allowlist` containing matching rules:
- Exact hostnames (e.g., `api.github.com`)
- Wildcards for subdomains (e.g., `*.github.com`)
- A catch-all wildcard (`*`) to permit other unlisted traffic

### Write the security allowlist script

Create `04_network_security.py`:

```python
from google import genai

client = genai.Client()

print("Launching interaction with restricted outbound network rules...")

# In this example, we restrict the agent to only contact api.github.com and PyPI,
# and block all other domains.
interaction = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="Try to fetch python packages from pypi.org, and also try to fetch google.com. Report which ones succeed.",
    environment={
        "type": "remote",
        "network": {
            "allowlist": [
                {"domain": "pypi.org"},
                {"domain": "files.pythonhosted.org"},
            ]
        },
    },
)

print("\n--- Agent Response ---")
print(interaction.output_text)
```

Run the script:

```bash
python 04_network_security.py
```

You should see the agent report that connection to `pypi.org` succeeds, while the outbound attempt to `google.com` is blocked and fails.

Negative
: A subdomain wildcard like `*.example.com` matches subdomains but does **not** match the root domain `example.com`. If you need to access both, you must specify them as separate entries in your allowlist.

## Secure Credential Injection
Duration: 0:08

When custom agents need to access private resources (such as private Git repositories or access-controlled databases), they require credentials.

However, writing keys to files or setting environment variables inside the sandbox exposes them to potential leakage. Gemini API resolves this by using **egress header transforms** in the egress proxy. The proxy injects credentials on-the-fly, so the raw keys are never visible inside the sandbox itself.

### Base64 Encoding Credentials

For private Git repositories, we use Basic Authentication with a GitHub Personal Access Token (PAT).
Encode the token on your terminal using the username `x-oauth-basic`:

```bash
echo -n "x-oauth-basic:ghp_YourGitHubTokenHere" | base64
```

This returns a base64 string, which we will use as `YOUR_BASE64_TOKEN`.

### Write the credential injection script

Create `05_credential_injection.py`:

```python
from google import genai

client = genai.Client()

print("Launching agent with secure proxy credential injection...")

# The Authorization header is injected by the secure egress proxy
# for all requests directed to github.com
interaction = client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="Clone the private repository https://github.com/your-org/private-repo and list its root files.",
    environment={
        "type": "remote",
        "sources": [
            {
                "type": "repository",
                "source": "https://github.com/your-org/private-repo",
                "target": "/workspace/private-repo",
            }
        ],
        "network": {
            "allowlist": [
                {
                    "domain": "github.com",
                    "transform": {
                        "Authorization": "Basic YOUR_BASE64_TOKEN"
                    },
                },
                {
                    "domain": "*" # Permitting other outbound traffic (such as package managers)
                }
            ]
        },
    },
)

print("\n--- Agent Response ---")
print(interaction.output_text)
```

Positive
: Header transforms support standard header schemas. For Google Cloud Storage buckets, use a standard OAuth token: `"Authorization": "Bearer GCLOUD_OAUTH_TOKEN"`.

## Creating and Managing Reusable Agents
Duration: 0:10

Once you've tuned your custom instructions, environments, and network configurations, you can save them as a registered Managed Agent. Registering the configuration allows you to invoke the agent using a custom ID, bypassing the need to pass instructions and configurations inline with every call.

### Step 1: Register a new Managed Agent

Create `06_manage_agents.py` to create, list, inspect, and delete agents:

```python
from google import genai

client = genai.Client()

agent_id = "my-custom-data-analyst"

print(f"1. Creating managed agent: {agent_id}...")

# Create the saved agent definition
agent = client.agents.create(
    id=agent_id,
    base_agent="antigravity-preview-05-2026",
    system_instruction="You are a senior data analyst. Format all outputs as markdown tables.",
    base_environment={
        "type": "remote",
        "sources": [
            {
                "type": "inline",
                "target": ".agents/AGENTS.md",
                "content": "Always focus on precision and performance metrics."
            }
        ]
    }
)
print(f"Agent successfully registered. ID: {agent.id}")

# Step 2: List registered agents
print("\n2. Listing your managed agents:")
agents = client.agents.list()
for a in agents.agents:
    print(f" - {a.id}: {a.description or 'No description'}")

# Step 3: Fetch the agent definition
print(f"\n3. Fetching details for agent {agent_id}...")
fetched_agent = client.agents.get(id=agent_id)
print(f"Fetched System Instruction: '{fetched_agent.system_instruction}'")

# Step 4: Invoke the managed agent
print(f"\n4. Invoking managed agent '{agent_id}'...")
result = client.interactions.create(
    agent=agent_id,
    input="Analyze our Q1 server load: 2000 requests/sec peak, 0.05% error rate, 45ms median response latency.",
)
print("\n--- Agent Response ---")
print(result.output_text)

# Step 5: Clean up the agent definition
print(f"\n5. Deleting managed agent '{agent_id}'...")
client.agents.delete(id=agent_id)
print("Agent deleted successfully.")
```

Run the lifecycle script:

```bash
python 06_manage_agents.py
```

### Expected Output

You should see the agent successfully created, listed, fetched, invoked, and cleaned up:

```console
1. Creating managed agent: my-custom-data-analyst...
Agent successfully registered. ID: my-custom-data-analyst

2. Listing your managed agents:
 - my-custom-data-analyst: No description

3. Fetching details for agent my-custom-data-analyst...
Fetched System Instruction: 'You are a senior data analyst. Format all outputs as markdown tables.'

4. Invoking managed agent 'my-custom-data-analyst'...

--- Agent Response ---
| Metric | Value | Status |
|---|---|---|
| Peak Server Load | 2000 requests/sec | High |
| Error Rate | 0.05% | Healthy |
| Median Latency | 45ms | Optimal |

5. Deleting managed agent 'my-custom-data-analyst'...
Agent deleted successfully.
```

Positive
: When you delete an agent configuration, only the registered definition is removed. Any active execution environments or running instances created by the agent will not be deleted.

## Wrap up
Duration: 0:02

Congratulations! You have completed the Managed Agents codelab.

You have built and configured custom remote environments, managed persistent states across interaction boundaries, secured outbound traffic using network allowlists, injected API credentials safely, and managed registered custom agents throughout their lifecycles.

### What you covered

- **Inline Configuration:** Customizing agent properties dynamically at execution time.
- **Persistent Environments:** Reusing remote execution sandboxes to maintain states, files, and installed libraries.
- **Declarative Workspace Mounting:** Declaring files inline, cloning repository sources, and downloading files from Cloud Storage buckets.
- **Egress Network Security:** Controlling outbound network traffic via allowlists and securely injecting credentials without exposure.
- **Agent Lifecycle Management:** Creating, listing, retrieving, and deleting saved managed agents.

### Where to go next

For deeper insights, check out the following references:
- [Managed Agents documentation](https://ai.google.dev/gemini-api/docs/custom-agents)
- [Environments in Managed Agents](https://ai.google.dev/gemini-api/docs/agent-environment)
- [Gemini API Python SDK reference](https://github.com/google-gemini/cookbook)
- [Antigravity Agent tools reference](https://ai.google.dev/gemini-api/docs/antigravity-agent)
