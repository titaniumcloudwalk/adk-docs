# Agent Observability with Cloud Trace

With ADK, you’ve already capable of inspecting and observing your agent interaction locally utilizing the powerful web development UI discussed in [here](https://google.github.io/adk-docs/evaluate/#debugging-with-the-trace-view). However, if we aim for cloud deployment, we will need a centralized dashboard to observe real traffic.

Cloud Trace is a component of Google Cloud Observability. It is a powerful tool for monitoring, debugging, and improving the performance of your applications by focusing specifically on tracing capabilities. For Agent Development Kit (ADK) applications, Cloud Trace enables comprehensive tracing, helping you understand how requests flow through your agent's interactions and identify performance bottlenecks or errors within your AI agents.

## Overview

Cloud Trace is built on [OpenTelemetry](https://opentelemetry.io/), an open-source standard that supports many languages and ingestion methods for generating trace data. This aligns with observability practices for ADK applications, which also leverage OpenTelemetry-compatible instrumentation, allowing you to :

- Trace agent interactions : Cloud Trace continuously gathers and analyzes trace data from your project, enabling you to rapidly diagnose latency issues and errors within your ADK applications. This automatic data collection simplifies the process of identifying problems in complex agent workflows.
- Debug issues : Quickly diagnose latency issues and errors by analyzing detailed traces. Crucial for understanding issues that manifest as increased communication latency across different services or during specific agent actions like tool calls.
- In-depth Analysis and Visualization: Trace Explorer is the primary tool for analyzing traces, offering visual aids like heatmaps for span duration and line charts for request/error rates. It also provides a spans table, groupable by service and operation, which gives one-click access to representative traces and a waterfall view to easily identify bottlenecks and sources of errors within your agent's execution path

The following example will assume the following agent directory structure

```
working_dir/
├── weather_agent/
│   ├── agent.py       # Python
│   └── __init__.py
├── weather_agent_ts/
│   ├── agent.ts       # TypeScript
│   └── package.json
└── deploy_agent_engine.py
└── deploy_fast_api_app.py
└── agent_runner.py
└── agent_runner.ts
```

=== "Python"

    ```python
    # weather_agent/agent.py

    import os
    from google.adk.agents import Agent

    os.environ.setdefault("GOOGLE_CLOUD_PROJECT", "{your-project-id}")
    os.environ.setdefault("GOOGLE_CLOUD_LOCATION", "global")
    os.environ.setdefault("GOOGLE_GENAI_USE_VERTEXAI", "True")


    # Define a tool function
    def get_weather(city: str) -> dict:
        """Retrieves the current weather report for a specified city.

        Args:
            city (str): The name of the city for which to retrieve the weather report.

        Returns:
            dict: status and result or error msg.
        """
        if city.lower() == "new york":
            return {
                "status": "success",
                "report": (
                    "The weather in New York is sunny with a temperature of 25 degrees"
                    " Celsius (77 degrees Fahrenheit)."
                ),
            }
        else:
            return {
                "status": "error",
                "error_message": f"Weather information for '{city}' is not available.",
            }


    # Create an agent with tools
    root_agent = Agent(
        name="weather_agent",
        model="gemini-2.5-flash",
        description="Agent to answer questions using weather tools.",
        instruction="You must use the available tools to find an answer.",
        tools=[get_weather],
    )
    ```

=== "TypeScript"

    ```typescript
    // weather_agent_ts/agent.ts

    import { LlmAgent, FunctionTool } from '@google/adk';
    import { z } from 'zod';

    // Define the tool schema
    const getWeatherSchema = z.object({
        city: z.string().describe('The name of the city for the weather report'),
    });

    // Define a tool function
    const getWeatherTool = new FunctionTool({
        name: 'get_weather',
        description: 'Retrieves the current weather report for a specified city.',
        parameters: getWeatherSchema,
        execute: async ({ city }) => {
            if (city.toLowerCase() === 'new york') {
                return {
                    status: 'success',
                    report: 'The weather in New York is sunny with a temperature of 25 degrees Celsius (77 degrees Fahrenheit).',
                };
            } else {
                return {
                    status: 'error',
                    error_message: `Weather information for '${city}' is not available.`,
                };
            }
        },
    });

    // Create an agent with tools
    export const rootAgent = new LlmAgent({
        name: 'weather_agent',
        model: 'gemini-2.5-flash',
        description: 'Agent to answer questions using weather tools.',
        instruction: 'You must use the available tools to find an answer.',
        tools: [getWeatherTool],
    });
    ```

## Cloud Trace Setup

### Setup for Agent Engine Deployment

#### Agent Engine Deployment - from ADK CLI

You can enable cloud tracing by adding `--trace_to_cloud` flag when deploying your agent using the `adk deploy agent_engine` command for agent engine deployment.

=== "Python"

    ```bash
    adk deploy agent_engine \
        --project=$GOOGLE_CLOUD_PROJECT \
        --region=$GOOGLE_CLOUD_LOCATION \
        --staging_bucket=$STAGING_BUCKET \
        --trace_to_cloud \
        $AGENT_PATH
    ```

=== "TypeScript"

    ```bash
    npx @google/adk-devtools deploy agent_engine \
        --project=$GOOGLE_CLOUD_PROJECT \
        --region=$GOOGLE_CLOUD_LOCATION \
        --staging_bucket=$STAGING_BUCKET \
        --trace_to_cloud \
        $AGENT_PATH
    ```

#### Agent Engine Deployment - from Python SDK

If you prefer using Python SDK, you can enable cloud tracing by adding `enable_tracing=True` when initialize the `AdkApp` object

```python
# deploy_agent_engine.py

from vertexai.preview import reasoning_engines
from vertexai import agent_engines
from weather_agent.agent import root_agent

import vertexai

PROJECT_ID = "{your-project-id}"
LOCATION = "{your-preferred-location}"
STAGING_BUCKET = "{your-staging-bucket}"

vertexai.init(
    project=PROJECT_ID,
    location=LOCATION,
    staging_bucket=STAGING_BUCKET,
)

adk_app = reasoning_engines.AdkApp(
    agent=root_agent,
    enable_tracing=True,
)


remote_app = agent_engines.create(
    agent_engine=adk_app,
    extra_packages=[
        "./weather_agent",
    ],
    requirements=[
        "google-cloud-aiplatform[adk,agent_engines]",
    ],
)
```

### Setup for Cloud Run Deployment

#### Cloud Run Deployment - from ADK CLI

You can enable cloud tracing by adding `--trace_to_cloud` flag when deploying your agent using the `adk deploy cloud_run` command for cloud run deployment.

=== "Python"

    ```bash
    adk deploy cloud_run \
        --project=$GOOGLE_CLOUD_PROJECT \
        --region=$GOOGLE_CLOUD_LOCATION \
        --trace_to_cloud \
        $AGENT_PATH
    ```

=== "TypeScript"

    ```bash
    npx @google/adk-devtools deploy cloud_run \
        --project=$GOOGLE_CLOUD_PROJECT \
        --region=$GOOGLE_CLOUD_LOCATION \
        --trace_to_cloud \
        $AGENT_PATH
    ```

If you want to enable cloud tracing and using a customized agent service deployment on Cloud Run, you can refer to the [Setup for Customized Deployment](#setup-for-customized-deployment) section below

### Setup for Customized Deployment

#### From Built-in `get_fast_api_app` Module

If you want to customize your own agent service, you can enable cloud tracing by initialize the FastAPI app using built-in `get_fast_api_app` module and set `trace_to_cloud=True`

```python
# deploy_fast_api_app.py

import os
from google.adk.cli.fast_api import get_fast_api_app
from fastapi import FastAPI

# Set GOOGLE_CLOUD_PROJECT environment variable for cloud tracing
os.environ.setdefault("GOOGLE_CLOUD_PROJECT", "alvin-exploratory-2")

# Discover the `weather_agent` directory in current working dir
AGENT_DIR = os.path.dirname(os.path.abspath(__file__))

# Create FastAPI app with enabled cloud tracing
app: FastAPI = get_fast_api_app(
    agents_dir=AGENT_DIR,
    web=True,
    trace_to_cloud=True,
)

app.title = "weather-agent"
app.description = "API for interacting with the Agent weather-agent"


# Main execution
if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8080)
```


#### From Customized Agent Runner

If you want to fully customize your ADK agent runtime, you can enable cloud tracing by using the OpenTelemetry Cloud Trace exporter.

=== "Python"

    ```python
    # agent_runner.py

    from google.adk.runners import Runner
    from google.adk.sessions import InMemorySessionService
    from weather_agent.agent import root_agent as weather_agent
    from google.genai.types import Content, Part
    from opentelemetry import trace
    from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
    from opentelemetry.sdk.trace import export
    from opentelemetry.sdk.trace import TracerProvider

    APP_NAME = "weather_agent"
    USER_ID = "u_123"
    SESSION_ID = "s_123"

    provider = TracerProvider()
    processor = export.BatchSpanProcessor(
        CloudTraceSpanExporter(project_id="{your-project-id}")
    )
    provider.add_span_processor(processor)
    trace.set_tracer_provider(provider)

    session_service = InMemorySessionService()
    runner = Runner(agent=weather_agent, app_name=APP_NAME, session_service=session_service)


    async def main():
        session = await session_service.get_session(
            app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
        )
        if session is None:
            session = await session_service.create_session(
                app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
            )

        user_content = Content(
            role="user", parts=[Part(text="what's weather in paris?")]
        )

        final_response_content = "No response"
        async for event in runner.run_async(
            user_id=USER_ID, session_id=SESSION_ID, new_message=user_content
        ):
            if event.is_final_response() and event.content and event.content.parts:
                final_response_content = event.content.parts[0].text

        print(final_response_content)


    if __name__ == "__main__":
        import asyncio

        asyncio.run(main())
    ```

=== "TypeScript"

    ```typescript
    // agent_runner.ts

    import { Runner, InMemorySessionService, getGcpExporters, maybeSetOtelProviders } from '@google/adk';
    import { rootAgent as weatherAgent } from './weather_agent_ts/agent';

    const APP_NAME = 'weather_agent';
    const USER_ID = 'u_123';
    const SESSION_ID = 's_123';

    async function main() {
        // Set up Google Cloud Trace exporters
        const gcpExporters = await getGcpExporters({ enableTracing: true });
        maybeSetOtelProviders([gcpExporters]);

        const sessionService = new InMemorySessionService();
        const runner = new Runner({
            agent: weatherAgent,
            appName: APP_NAME,
            sessionService,
        });

        let session = await sessionService.getSession({
            appName: APP_NAME,
            userId: USER_ID,
            sessionId: SESSION_ID,
        });

        if (!session) {
            session = await sessionService.createSession({
                appName: APP_NAME,
                userId: USER_ID,
                sessionId: SESSION_ID,
            });
        }

        const userContent = {
            role: 'user' as const,
            parts: [{ text: "what's weather in paris?" }],
        };

        let finalResponseContent = 'No response';
        for await (const event of runner.runAsync({
            userId: USER_ID,
            sessionId: SESSION_ID,
            newMessage: userContent,
        })) {
            if (event.isFinalResponse() && event.content?.parts) {
                const textPart = event.content.parts.find(p => 'text' in p);
                if (textPart && 'text' in textPart) {
                    finalResponseContent = textPart.text;
                }
            }
        }

        console.log(finalResponseContent);
    }

    main();
    ```

## Inspect Cloud Traces

After the setup is complete, whenever you interact with the agent it will automatically send trace data to Cloud Trace. You can inspect the traces by going to [console.cloud.google.com](https://console.cloud.google.com) and visit the Trace Explorer on the configured Google Cloud Project

![cloud-trace](../assets/cloud-trace1.png)

And then you will see all available traces produced by ADK agent which configured in several span names such as `invocation` , `agent_run` . `call_llm` and `execute_tool`

![cloud-trace](../assets/cloud-trace2.png)

If you click on one of the traces, you will see the waterfall view of the detailed process, similar to what we see in the web development UI with `adk web` command.

![cloud-trace](../assets/cloud-trace3.png)

## Resources

- [Google Cloud Trace Documentation](https://cloud.google.com/trace)
