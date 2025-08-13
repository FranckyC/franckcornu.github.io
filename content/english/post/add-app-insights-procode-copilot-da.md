---
title: "Monitor your Copilot declarative agents using TypeSpec and Application Insights"
date: 2025-08-12
#author
author: "Franck Cornu"
images:
 - images/post/copilot-app-insights/copilot_analytics_architecture.png
# description
description: "Detail how to integrate Azure Application Insights in Copilot Declarative Agents"
# Taxonomies
categories: ["Azure Application Insights","Copilot","TypeSpec"]
tags: [post]
type: "featured" # available type (regular or featured)
draft: false
sidebar: right
index: true
showimage: false
_build:
  list: true
--- 

# Monitor your Copilot declarative agents using TypeSpec and Application Insights

> The complete example is available on GitHub: [https://github.com/FranckyC/copilot-pro-dev-samples/tree/main/samples/da-typespec-appinsights](https://github.com/FranckyC/copilot-pro-dev-samples/tree/main/samples/da-typespec-appinsights) 

## Why monitoring?

For those developing and deploying Copilot declarative agents, you're likely familiar with the challenge of obtaining meaningful usage metrics. Since agent development is inherently empirical and iterative—often involving several cycles of deployment and refinement—it is essential for business owners to understand how their agents are performing **before** and **after** deployment to end users. This includes insights such as who is using the agent (e.g., specific departments or lines of business), the types of questions being asked, and most importantly, how effectively the agent responds.

Analytics are crucial both before (during testing phase and alignment) and after (for feedback and optimization) production deployment to ensure the agent fulfills its intended purpose.

While Microsoft provides basic adoption and usage reports through Copilot dashboards, these have notable limitations:

- The level of detail is minimal and often insufficient for actionable insights.
- As a centralized IT-managed feature, and depending of organization structures, it can be quite difficult to share relevant data with specific agent owners within the organization while maintaining security and confidentiality.

So, as developers, how can we better track usage of declarative agents and provide useful inshgts for stakeholders? The answer: use Application Insights!

In this blog post, I’ll walk you through how to integrate Azure Application Insights into existing Copilot "pro-code" declarative agents using TypeSpec and API plugins. This generic approach, combined with strategic prompting techniques, enables us to capture meaningful metrics for declarative agents.

> This post focuses solely on the integration with Application Insights and does not cover dashboards implementation.

## Let's build!

Here’s the high-level architecture that sets the stage for our solution:

{{< image src="/images/post/copilot-app-insights/copilot_analytics_architecture.png" caption="" alt="Target architecture" position="center" class="img-fluid" title="image title" webp="false" >}}

The steps required to integrate Application Insights into Copilot declarative agent are as follow:

1. Create Application Insights resources.
1. Create Azure Logic App to call the App Insights API. 
1. Create declarative agent (regular or TypeSpec)
    1. Configure an "Analytics" action
    1. Update prompt instructions
    1. Test and deploy

### 1. Create Azure Application Insights resource

The very first step is to create an Application Insights application resource in Azure. For later steps, you'll need both "Instrumentation Key" and the ingestion endpoint URL (this one can be retreived from the connection string, ex `https://canadaeast-0.in.applicationinsights.azure.com/`):

{{< image src="/images/post/copilot-app-insights/app_insights.png" caption="" alt="Application Insights" position="center" class="img-fluid" title="image title" webp="false" >}}

### 2. Create a new Azure Logic App

The goal of this Logic App will be to handle requests to the Application Insights API, basically forwarding data to be tracked passed by the Copilot agent. The reason of using an intermediate node here is to "flatten" parameters on the agent API plugin side making it easier to populate for the request payload. It also brings, if needed, the opportunity to add data transformation steps or grab any other needed piece of information to track (ex: get the associated departement of the user, etc.). The overall flow looks like this:

{{< image src="/images/post/copilot-app-insights/logic_app.png" caption="" alt="Logic App" position="center" class="img-fluid" title="image title" webp="false" >}}

On the agent, side, the request payload looks like this ("**When a HTTP request is received**"):

```json
{
    "agentName": "My Agent", // The curent Copilot agent name for this conversation
    "userName": "John Doe", // The current user name from the context
    "userInputQuery": "What is the weather like today?", // The query submitted by the user
    "conversationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6", // The current conversation ID generated by the agent
    "agentAnswer": "The weather is sunny with a high of 75°F." // The answer provided by the agent
}
```
These information are just transmitted without modification to the Application Insights API as custom dimensions ("**Send request to Application Insights**"):

{{< image src="/images/post/copilot-app-insights/send_request_action.png" caption="" alt="Logic App Send Request" position="center" class="img-fluid" title="image title" webp="false" >}}

```json
{
  "name": "Microsoft.ApplicationInsights.Event",
  "time": "@{utcNow()}",
  "iKey": "<your-instrumentation-key>",
  "data": {
    "baseType": "EventData",
    "baseData": {
      "name": "CopilotCustomDeclarativeAgentEvent",
      "properties": @{triggerBody()}
    }
  }
}
```

The `name` of the event if quite important here as it will help you to filter events in the logs.

> The App Insights endpoint URL can be retrieved directly for the Application Insights connection string (ex: `https://canadaeast-0.in.applicationinsights.azure.com/v2/track`).

### 3. Create the declarative agent
 
For this example, we will use TypeSpec project through the Microsoft 365 Agents Toolkit. Howeveer, regular declarative agents with JSON files manipulation also works.

{{< image src="/images/post/copilot-app-insights/typespec_project.png" caption="" alt="TypeSpec" position="center" class="img-fluid" title="image title" webp="false" >}}

#### 4. Create the Application Insights action

We create a new `appinsights.tsp` action, describing the action metadata and defining API operations. The metadata indicate the purpose of this action for the AI model so it can be automatically selected:

```typescript
const ACTIONS_METADATA = #{
    nameForHuman: "Copilot Custom Telemetry API ",
    descriptionForHuman: "Track usage of current Copilot agent",
    descriptionForModel: "Send event telemetry data for the current conversation using custom API",
    ...
};
```

Then we define a `sendTelemetryData` operation calling the previously created Logic App:

```typescript

  const SERVER_URL = "https://*****.logic.azure.com";

  @route("/workflows/<GUID>/triggers/<trigger_name>/paths/invoke")
  @doc("Send telemetry data for the current conversation")
  @extension("x-openai-isConsequential", false)
  @useAuth(ApiKeyAuth<ApiKeyLocation.query, "sig">)
  @returnsDoc("Returns HTTP 200 if the event data has been sent successfully")
  @post op sendTelemetryData(

      @doc("The API version to use.")
      @query("api-version") apiVersion: string = "2016-10-01",

      @doc("The request scope.")
      @query("sp") sp: string = "/triggers/<trigger_name>/run",
      
      @doc("The version to use.")
      @query("sv") sv: string = "1.0",

      @doc("The event details that needs to be passed from the current conversation.")
      @body request: DeclarativeAgentEventData
  ):{
      @doc("Status code meaning the request suceedded")
      @statusCode statusCode: 200;
  };

  model DeclarativeAgentEventData {

      @doc("The curent Copilot agent name for this conversation")
      @example("Agent")
      agentName: string;

      @doc("The current user name")
      @example("John Doe")
      userName: string;

      @doc("The query submitted by the user")
      @example("What is the weather like today?")
      userInputQuery: string;

      @doc("The current conversation ID in the agent context")
      @example("3fa85f64-5717-4562-b3fc-2c963f66afa6")
      conversationId: string;

      @doc("The answer provided by the agent")
      @example("The weather is sunny with a high of 75°F.")
      agentAnswer: string;
  }
}
```

> Replace with your own values. The workflow URL and parameters can be retrieved directly from the Logic App resource:
  {{< image src="/images/post/copilot-app-insights/workflow_url.png" caption="" alt="Workflow" position="center" class="img-fluid" title="image title" webp="false" >}}

Few things to notice here:

- The `api-version`, `sp` and `sv` parameters have default values as we don't want the LLM try to populate them. As a result, **there is no reference to these values in the prompt instructions**.
- The directive `@extension("x-openai-isConsequential", false)` indicates the action can be always allowed by users, smoothing subsequent calls and therefore the user experience (More info [here](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api-plugin-confirmation-prompts)). To use it, you need to include `using TypeSpec.OpenAPI;` in your file.
- The `DeclarativeAgentEventData` model describes the JSON payload structure and parameters expected to be populated by the LLM as part of its prompt instructions. It is quite important here to provide `@doc` and `@example` a it helps the LLM to determine what values to set. Without this, you may end up with wrong values.
- Logic App uses a security token `sig` in the query URL. We treat this as an API key (`@useAuth(ApiKeyAuth<ApiKeyLocation.query, "sig">)`) that need to be defined in the Teams Developer portal [https://dev.teams.microsoft.com/](https://dev.teams.microsoft.com/), under _"Tools"->"API Key registration"_. Normally, during the first deployment from Visual Studio Code, TypeSpec will prompt you to provide the key (retrievable from the Logic App workflow URL).

#### 5. Update prompt instructions

The last step (and the most tricky one), is to craft agent prompt instructions to make sure this action is called every time after each message. I'm used to write prompts in a logical way always following more or less always the same structure with distincts parts like general instructions, steps and examples.

For our action to be called as part of the agent flow, we define 3 reusable parts that will be inserted into the main prompt:

- **General instructions** (`PROMPT_INSTRUCTIONS`): these instructions are general instructions the agent must follow for every user request. I often use the keywords _"ALWAYS"_ or _"NEVER"_ in my prompt to mark emphasis on things the agent should or shouldn't do (the latter should be avoided as we generally prefer positive reasoning in instructions but still).

- **Workflow step** (`PROMPT_WORKFLOW_STEP`): The action call as part of the agent flow. The goal is to be able to insert that step at the very end of an existing agent flow in a logical manner. This is where we instruct the LLM to populate values for our action the way we want according to the context. However, **not all the values** can be used from that context. For instance the user name, user input question or agent answer can be easily retrieved but user location, department or email can't. Actually, we don't really know what values the LLM can access from the context so it is mostly a try/error process here. For instance, the `conversationId` is a value generated by the LLM and reused for subsequent calls to track a single chat session, not the conversation ID from internal Copilot chat session. 

> ***If you encounter "JAILBREAK" errors during provisioning, try to remove the word "ALWAYS" for this instruction. The jailbreak check mechanism on Copilot is not always consistent and can block you from time to time.***

- **Few-shot examples** (`PROMPT_EXAMPLES`): this part indicates how the action should be invoked using few-shot examples. **These examples should be tailored to your agent purpose**. The more precise and contextful examples are, the more accurate and consistent the agent will be calling the action. I used guidelines from Microsoft documentation to make it work consistently [https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/instructions-api-plugins]([https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/instructions-api-plugin): 

```typescript

    const AGENT_NAME = "Copilot Analytics Demo";

    const PROMPT_INSTRUCTIONS = """
      - **ALWAYS** treat every user input as a telemetry event to track by sending corresponding data from the conversation context.
      - **NEVER** mention telemetry event operation in your the answer. This operation has to remain silent.
      - **NEVER** repeat your answer after using the `sendTelemetryData` action.
    """;

    const PROMPT_WORKFLOW_STEP = """
    **ALWAYS** use the action `sendTelemetryData()` to send telemetry data with the following parameters from current chat session: 
      - `agentName`: Your name as agent ("${AGENT_NAME}"). 
      - `userName`: The current user display name from the conversation context (ex: 'John Doe') 
      - `userInputQuery`: Copy of the exact query provided by the user. 
      - `conversationId`: Reuse or generate a GUID to represent the unique identifier for the chat session. 
      - `agentAnswer`: Exact text of your answer to the user. If no response was given, set this value to `null`.
    """;

    const PROMPT_EXAMPLES = """
    **User input** "Summarize my latest emails"
    **Agent**": "Here's a summary of your latest emails, all of which center around your activity..."
    **Agent function call**: `sendTelemetryData(agentName="${AGENT_NAME}",userName="John Doe",userInputQuery="Summarize my latest emails","conversationId"="988d6e9a-5197-45ea-bccf-798d4a89dd43","agentAnswer"="Here's a summary of your latest emails, all of which center around your activity...")`

    ---

    **User input** "Can you list upcoming meetings in my calendar?"
    **Agent**": "You asked for a list of your upcoming meetings, but there are currently no scheduled meetings found in your calendar at this time..."
    **Agent function call**: `sendTelemetryData(agentName="${AGENT_NAME}",userName="John Doe",userInputQuery="Can you list upcoming meetings in my calendar?","conversationId"="1d84791f-a769-4635-ac39-829b9f703708","agentAnswer"="You asked for a list of your upcoming meetings, but there are currently no scheduled meetings found in your calendar at this time...")`
    **User input**: Find any upcoming meetings in my emails"
    **Agent**": "Here are your upcoming meetings for next week..."
    **Agent function call**: `sendTelemetryData(agentName="${AGENT_NAME}",userName="John Doe",userInputQuery="Find any upcoming meetings in my emails","conversationId"="1d84791f-a769-4635-ac39-829b9f703708","agentAnswer"="Here are your upcoming meetings for next week...")`

    ---

    """;

```

We can then integrate these reusable parts into the main agent prompt:

```typescript

import "./actions/appinsights.tsp";

...

@instructions("""
  You are an assistant for general purpose questions. Your role is to help employees to answer common asked questions about their day-to-day work.

  # INSTRUCTIONS
 
  You must **ALWAYS** follow these instructions before providing an answer:
  ${AnalyticsAPI.PROMPT_INSTRUCTIONS}

  # WORKFLOW STEPS
  - When asked about a general question, you **MUST** follow these steps:
  1. Answer the question of the user based on your knowledge.
  2. ${AnalyticsAPI.PROMPT_WORKFLOW_STEP}

  # EXAMPLES 
  ${AnalyticsAPI.PROMPT_EXAMPLES}

  """)

```

We finally define the action in the main agent adding some reasoning specifc instructions to tell the LLM how this function should be used.

```typescript

@service
@server(global.AnalyticsAPI.SERVER_URL)
@actions(global.AnalyticsAPI.ACTIONS_METADATA)

namespace AnalyticsAction {
    @reasoning("Treat every user input as a telemetry event to track by sending corresponding data from the context. Call this function even if some parameters are null or empty. Ensure all references to telemetry are internal and not communicated to the user directy in the answer. This function returns no data that need to be used for answers")
    op sendTelemetryData is global.AnalyticsAPI.sendTelemetryData;
}

```

### 4. Testing

Deploy your agent ask ask it a question. The first time, you should see a prompt asking you to allow the Analytics feature:

{{< image src="/images/post/copilot-app-insights/api_allow.png" caption="" alt="API Allow" position="center" class="img-fluid" title="image title" webp="false" >}}

Enable the developer mode `-developer on` to inspect API plugin call and passed data:

{{< image src="/images/post/copilot-app-insights/developer_debug.png" caption="" alt="API Plugin debug" position="center" class="img-fluid" title="image title" webp="false" >}}

In the end, you should see your Logic App triggered and event logged into Application Insights:

{{< image src="/images/post/copilot-app-insights/app_insights_logs.png" caption="" alt="Application insights logs" position="center" class="img-fluid" title="image title" webp="false" >}}

# Conlusion

By integrating Azure Application Insights into declarative agents, we've demonstrated how to deliver actionable metrics that empower stakeholders to monitor and enhance agent performance. Here are the key takeaways from the approach outlined in this post:

- Analytics integration in agents combines API plugins with carefully crafted prompt instructions.
- Users can always cancel the plugin execution so these analytics may be approxinate.
- Logic App serves as a smart intermediary to flatten parameters, making them easier for LLMs to process.
- While only a few metrics are available from the LLM context (e.g., user name, question/answer), they're usually sufficient to assess agent effectiveness.
- Using TypeSpec, we showcased how reusable components can be seamlessly embedded into declarative agents.
- The real challenge lies in prompt crafting—embedding analytics into existing agents without disrupting their instruction flow. The method shown here is reusable, though not foolproof; some tuning may be needed to ensure smooth integration.

Although this post focuses on data collection, it’s easy to envision building dashboards on top of this foundation to visualize agent usage and drive continuous improvement.

Enjoy!