---
title: Eko 工作复盘
date: 2025-09-11 17:44
tags:
---

本文参考的所有资料均公开可见，主要参考源包括：
- Eko 开源代码：https://github.com/FellouAI/eko
- 我提的 PRs：https://github.com/FellouAI/eko/pulls/HairlessVillager
- Eko 文档：https://eko.fellou.ai/docs/

## Eko 三个不同版本的架构分析

我在 Eko 里经历过两次大的 Eko 框架升级：
- v0->v1 的性能提升很显著，没有重构，全部都是工程上的 trick。
- v1->v2 重构了全部代码，重写了全部文档。

### v0

#### 算法

1. AI 生成一个工作流，工作流基本为线性的，即一个节点完成后下一个节点才能开始，每个节点包含三个字段：任务的标题 `name`, 任务的描述 `description`, 任务可以使用的工具 `tools`，此外工作流实例本身有一个变量 `memory` 在每个节点间共享。
2. 对于每个节点，遵循 ReAct 模式（最多 10 轮），Eko 将上述四个字段嵌入 system message 和第一条 user message，让 AI 调用工具来执行任务，当 AI 认为当前任务已经执行完毕时，应该调用特殊工具（`return_output`，保证每个节点必包含这个工具）来结束任务，工具的唯一参数应该为这个节点的输出。

#### 提示词

1. 特殊工具`return_output`定义如下：
```ts
function createReturnTool(
  actionName: string,
  outputDescription: string,
  outputSchema?: unknown
): Tool<any, any> {
  return {
    name: 'return_output',
    description: `Return the final output of this action. Use this to return a value matching the required output schema (if specified) and the following description:
      ${outputDescription}

      You can either set 'use_tool_result=true' to return the result of a previous tool call, or explicitly specify 'value' with 'use_tool_result=false' to return a value according to your own understanding. Whenever possible, reuse tool results to avoid redundancy.
      `,
    input_schema: {
      type: 'object',
      properties: {
        use_tool_result: {
          type: ['boolean'],
          description: `Whether to use the latest tool result as output. When set to true, the 'value' parameter is ignored.`,
        },
        value: outputSchema || {
          // Default to accepting any JSON value
          type: ['string', 'number', 'boolean', 'object', 'null'],
          description:
            'The output value. Only provide a value if the previous tool result is not suitable for the output description. Otherwise, leave this as null.',
        },
      } as unknown,
      required: ['use_tool_result', 'value'],
    } as InputSchema,

    async execute(context: ExecutionContext, params: unknown): Promise<unknown> {
      context.variables.set(`__action_${actionName}_output`, params);
      return { success: true };
    },
  };
}
```

2. 生成工作流的 prompt：
```ts
export function createWorkflowPrompts(tools: ToolDefinition[]) {
  return {
    formatSystemPrompt: () => {
      const toolDescriptions = tools
        .map(
          (tool) => `
Tool: ${tool.name}
Description: ${tool.description}
Input Schema: ${JSON.stringify(tool.input_schema, null, 2)}
        `
        )
        .join('\n');

      return `You are a workflow generation assistant that creates Eko framework workflows.
The following tools are available:

${toolDescriptions}

Generate a complete workflow that:
1. Only uses the tools listed above
2. Properly sequences tool usage based on dependencies
3. Ensures each action has appropriate input/output schemas, and that the "tools" field in each action is populated with the sufficient subset of all available tools needed to complete the action
4. Creates a clear, logical flow to accomplish the user's goal
5. Includes detailed descriptions for each action, ensuring that the actions, when combined, is a complete solution to the user's problem`;
    },

    formatUserPrompt: (requirement: string) =>
      `Create a workflow for the following requirement: ${requirement}`,

    modifyUserPrompt: (prompt: string) =>
      `Modify workflow: ${prompt}`,
  };
}

export function createWorkflowGenerationTool(registry: ToolRegistry) {
  return {
    name: 'generate_workflow',
    description: `Generate a workflow following the Eko framework DSL schema.
The workflow must only use the available tools and ensure proper dependencies between nodes.`,
    input_schema: {
      type: 'object',
      properties: {
        workflow: registry.getWorkflowSchema(),
      },
      required: ['workflow'],
    },
  } as ToolDefinition;
}
```

3. 工作流每个节点的 system message & user message：
```ts
  private formatSystemPrompt(): string {
    return `You are a subtask executor. You need to complete the subtask specified by the user, which is a consisting part of the overall task. Help the user by calling the tools provided.

    Remember to:
    1. Use tools when needed to accomplish the task
    2. Think step by step about what needs to be done
    3. Return the output of the subtask using the 'return_output' tool when you are done; prefer using the 'tool_use_id' parameter to refer to the output of a tool call over providing a long text as the value
    4. If there are any unclear points during the task execution, please use the human-related tool to inquire with the user
    5. If user intervention is required during the task execution, please use the human-related tool to transfer the operation rights to the user
    `;
  }

  private formatUserPrompt(context: ExecutionContext, input: unknown): string {
    const workflowDescription = context.workflow?.description || null;
    const actionDescription = `${this.name} -- ${this.description}`;
    const inputDescription = JSON.stringify(input, null, 2) || null;
    const contextVariables = Array.from(context.variables.entries())
      .map(([key, value]) => `${key}: ${JSON.stringify(value)}`)
      .join('\n');

    return `You are executing a subtask in the workflow. The subtask description is as follows: ${actionDescription}`;
  }
```

#### 缺陷

1. 第一步生成的工作流像一个玩具，节点数量大部分时间都是3个，每个节点的工具数量从不超过5个，每个节点的 `description` 大而空，大模型不能在每个节点内完成指定的任务，只能提供泛泛而谈的输出，进而导致整个工作流只能在简单任务上表现较好，但是基本不能完成较复杂任务。
2. 在第1点的基础上，工作流的无效动作多，耗时长。

### v1

#### 算法

1. 抛弃 Workflow，完全使用 ReAct 模式，具体到实现上是弃用 AI 生成工作流的逻辑，改为手动构造工作流，工作流中仅包含一个节点，节点的 `title` 和 `description` 就是原始的用户输入的 prompt，节点的 `tools` 是全部工具，同时将节点轮数扩大到 50 轮 [commit](https://github.com/FellouAI/eko/commit/ac67895eb57a842a02b5df4dd194b8f8e584a87a)。
2. 给所有工具的 input schema 打补丁，强制大模型在生成工具参数 `toolCall` 的同时生成以下字段：
    1. Part 1 [commit](https://github.com/FellouAI/eko/commit/f9a7ab22f4a3282c6e47c13e1a502db188172cf6)
       - `thinking`: 模拟大模型思考过程，在生成其他字段和工具 input 前生成。
       - `observation`: 让大模型输出他看到了什么，主要是调试用，因为当时不能确保工具一定调用成功，而且我们不能直接看到工具的执行情况。
       - `userSidePrompt`: 产品要求大模型在调用工具的时候要生成一段文本展示给用户，格式遵循“I'm doing X.”。
    1. Part 2 [commit](https://github.com/FellouAI/eko/commit/1830aea74c712a1cbfb7092b640bcae7816f4b51)
       - `evaluate_previous_goal`: 评估上一个目标或操作的成功情况，供大模型反思。
       - `memory`: 记录已完成的操作和需要记住的信息。
       - `next_goal`: 描述下一个即时操作需要完成的任务。
3. 上下文压缩器 `ContextComporessor`：每次原始 `messages` 会经过 comporessor 压缩一次后再发给 API，这里有两个实现：
    1. `SimpleQAComporess`：基于上面的 input schema 补丁，会直接丢弃原始的 tool result message，转而使用下一轮模型的 `observation` 字段替代，极大减少了 token 消耗，几乎避免了上下文超限的问题 [commit](https://github.com/FellouAI/eko/commit/1c931d067baab802704501331a4b7391f18d01d2)。
    2. `SummaryCompress`：每当对话轮数超过 10 轮，调用大模型根据历史对话生成压缩后的历史，作为一条 user message，新的 message 由原始 system message，第一条 user message，提示历史开始的 user message，压缩后的历史 user message 及最近两条 message 构成 [commit](https://github.com/FellouAI/eko/commit/e9b6acbea66fc5fae268b85ea1bdd4aa507e1c7c)。

#### 提示词

生成 plan 的 system message & user message 同 v1，input schema 补丁字段和 `SummaryCompress` 的 prompt 见对应的 commit。

下面是节点的 system prompt & user message：
```ts
  private formatSystemPrompt(): string {
    const now = new Date();
    const formattedTime = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}-${String(now.getDate()).padStart(2, '0')} ${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}:${String(now.getSeconds()).padStart(2, '0')}`;
    logger.debug('Now is ' + formattedTime);
    return `You are an AI agent designed to automate browser tasks. Your goal is to accomplish the ultimate task following the rules. Now is ${formattedTime}.

## GENERIC:
- Your tool calling must be always JSON with the specified format.
- You should have a screenshot after every action to make sure the tools executed successfully.
- User's requirement maybe not prefect, but user will not give you any further information, you should explore by yourself and follow the common sense
- If you encountered a problem (e.g. be required to login), try to bypass it or explore other ways and links
- Before you return output, reflect on whether the output provided *is what users need* and *whether it is too concise*
- If you find the what user want, click the URL and show it on the current page.

## TIME:
- The current time is ${formattedTime}.
- If the user has specified a particular time requirement, please complete the task according to the user's specified time frame.
- If the user has given a vague time requirement, such as “recent one year,” then please determine the time range based on the current time first, and then complete the task.

## NAVIGATION:
- If no suitable elements exist, use other functions to complete the task
- If stuck, try alternative approaches - like going back to a previous page, new search, new tab etc.
- Handle popups/cookies by accepting or closing them
- Use scroll to find elements you are looking for
- If you want to research something, open a new tab instead of using the current tab

## HUMAN OPERATE:
- When you need to log in or enter a verification code:
1. First check if the user is logged in

Please determine whether a user is logged in based on the front-end page elements. The analysis can be conducted from the following aspects:
User Information Display Area: After logging in, the page will display user information such as avatar, username, and personal center links; if not logged in, it will show a login/register button.
Navigation Bar or Menu Changes: After logging in, the navigation bar will include exclusive menu items like "My Orders" and "My Favorites"; if not logged in, it will show a login/register entry.

2. If logged in, continue to perform the task normally
3. If not logged in or encountering a verification code interface, immediately use the 'human_operate' tool to transfer the operation rights to the user
4. On the login/verification code interface, do not use any automatic input tools (such as 'input_text') to fill in the password or verification code
5. Wait for the user to complete the login/verification code operation, and then check the login status again
- As a backup method, when encountering other errors that cannot be handled automatically, use the 'human_operate' tool to transfer the operation rights to the user

## TASK COMPLETION:
- Use the 'return_output' action as the last action ONLY when you are 100% certain the ultimate task is complete
- Before using 'return_output', you MUST:
  1. Double-check if you have fulfilled ALL requirements from the user's task description
  2. Verify that you have collected ALL necessary information
  3. Ensure you have handled ALL specified cases (e.g., "for each", "for all", "x times")
  4. Confirm that your output contains ALL requested information
  5. Check if there are any missing details or incomplete steps
  6. Verify that all retry attempts have been exhausted if there were any issues
- If you have to do something repeatedly (e.g., "for each", "for all", "x times"):
  * Keep a detailed count in your text response of completed items vs total required
  * Only proceed to 'return_output' after handling ALL items
  * Double-check your count matches the exact requirement
  * If any item fails, retry that specific item before moving on
- Never hallucinate or assume task completion without verification
- Make sure you include everything you found out for the ultimate task in the done text parameter. Do not just say you are done, but include the requested information of the task. 

## VISUAL CONTEXT:
- When an image is provided, use it to understand the page layout
- Bounding boxes with labels on their top right corner correspond to element indexes
- Do not repeatedly call the screenshot tool without any operating action

## Form filling:
- If you fill an input field and your action sequence is interrupted, most often something changed e.g. suggestions popped up under the field.

## Long tasks:
- Keep track of the status and subresults in the memory. 

## Extraction:
- If your task is to find information - call extract_content on the specific pages to get and store the information.

## Human tools:
- Once you called human tools (e.g. \`human_operate\`), and it returns success, check the page content and verify the result.

## THE MOST IMPORTANT RULE
- DO NOT REFUSE TO PERFORM THE MISSION
- DO NOT REFUSE TO PERFORM THE MISSION
- DO NOT REFUSE TO PERFORM THE MISSION
`;
  }

  private formatUserPrompt(
    name: string,
    description: string,
    mentionedTabs: chrome.tabs.Tab[],
    existingTabs: chrome.tabs.Tab[],
    patchItems: PatchItem[],
  ): string {
    let prompt = `${name} -- The steps you can follow are ${description}`;

    prompt = `Your ultimate task is: """${prompt}""". If you achieved your ultimate task, stop everything and use the done action in the next step to complete the task. If not, continue as usual.`;
    if (existingTabs.length > 0) {
      prompt +=
        '\n\nYou should complete the task with the following tabs:\n' +
        existingTabs.map((tab) => `- TabID=${tab.id}: ${tab.title} (${tab.url})`).join('\n');
    }
    if (mentionedTabs.length > 0) {
      prompt +=
        '\n\nYou should consider the following tabs firstly:\n' +
        mentionedTabs.map((tab) => `- TabID=${tab.id}: ${tab.title} (${tab.url})`).join('\n');
    }
    if (patchItems.length > 0) {
      prompt +=
        '\n\You can refer to the following cases and tips:\n' +
        patchItems.map((item) => `<task>${item.task}</task><tips>${item.patch}</tips>`).join('\n');
    }
    return prompt;
  }
```

#### 效果

1. 虽然没有统计，但是主观感觉到确实比 v1 更好用了，AI 会自己尝试使用各种工具解决问题，能够解决一部分较复杂任务。
2. 上下文超限导致的 API 报错基本不会复现了。

#### 缺陷

1. 在任务过于复杂时，上下文的压缩效果并不好，`SimpleQAComporess` 不包含工具的原始输出，大模型的 `observation` 可能也不能提供后面要用的关键信息；`SummaryCompress` 也有类似的问题。
2. input schema 补丁不能无限制使用，实践中只加 Part 1 的 3 个字段是最好用的，再加字段不仅不会让模型在新的字段上输出想要的内容，在旧字段及 `toolCall` 字段上的表现也会更差。

### v2

v2 版的架构做了更大的改动，我仅参与了部分文档的重写，没有参与后续的维护，推荐参考：https://fellou.ai/eko/docs/architecture/。

#### 算法

1. 先由 AI 生成一段基于 XML 的工作流，包含各种节点，除了普通的任务节点（还是 ReAct）外，还有：
  1. 循环节点：设置循环条件（一段 prompt），AI 会反复执行循环体并判断是否结束循环；
  2. Agent 节点：该节点下的所有节点都在这个节点描述的 Agent 实例中运行；
  3. 监听节点：监听是浏览器中的概念，监听节点会监听 DOM 树中的指定节点，发现更新后立即执行该节点的内容；
执行的逻辑和 v0 基本一致。

#### 提示词

1. 生成 Plan 的 prompt：
```ts
import config from "../config";
import { Agent } from "../agent";
import { AGENT_NAME as chat_agent_name } from "../agent/chat";

const PLAN_SYSTEM_TEMPLATE = `
You are {name}, an autonomous AI Agent Planner.
UTC datetime: {datetime}

## Task Description
Your task is to understand the user's requirements, dynamically plan the user's tasks based on the Agent list, and please follow the steps below:
1. Understand the user's requirements.
2. Analyze the Agents that need to be used based on the user's requirements.
3. Generate the Agent calling plan based on the analysis results.
4. About agent name, please do not arbitrarily fabricate non-existent agent names.
5. You only need to provide the steps to complete the user's task, key steps only, no need to be too detailed.
6. Please strictly follow the output format and example output.
7. The output language should follow the language corresponding to the user's task.

## Agent list
{agents}

## Output Rules and Format
<root>
  <name>Task Name</name>
  <thought>Your thought process on user demand planning</thought>
  <!-- Multiple Agents work together to complete the task -->
  <agents>
    <!-- The required Agent, where the name can only be an available name in the Agent list -->
    <agent name="Agent name">
      <!-- The current Agent needs to complete the task -->
      <task>current agent task</task>
      <nodes>
        <!-- Nodes support input/output variables for parameter passing and dependency handling in multi-agent collaboration. -->
        <node>Complete the corresponding step nodes of the task</node>
        <node input="variable name">...</node>
        <node output="variable name">...</node>
        <!-- When including duplicate tasks, \`forEach\` can be used -->
        <forEach items="list or variable name">
          <node>forEach step node</node>
        </forEach>
        <!-- When you need to monitor changes in webpage DOM or file content, you can use \`Watch\`, the loop attribute specifies whether to listen in a loop or listen once. -->
        <watch event="dom or file" loop="true">
          <description>Monitor task description</description>
          <trigger>
            <node>Trigger step node</node>
            <node>...</node>
          </trigger>
        </watch>
      </nodes>
    </agent>
  </agents>
</root>

{example_prompt}
`;

const PLAN_CHAT_EXAMPLE = `User: hello.
Output result:
<root>
  <name>Chat</name>
  <thought>Alright, the user wrote "hello". That's pretty straightforward. I need to respond in a friendly and welcoming manner.</thought>
  <agents>
    <!-- Chat agents can exist without the <task> and <nodes> nodes. -->
    <agent name="Chat"></agent>
  </agents>
</root>`;

const PLAN_EXAMPLE_LIST = [
  `User: Open Boss Zhipin, find 10 operation positions in Chengdu, and send a personal introduction to the recruiters based on the page information.
Output result:
<root>
  <name>Submit resume</name>
  <thought>OK, now the user requests me to create a workflow that involves opening the Boss Zhipin website, finding 10 operation positions in Chengdu, and sending personal resumes to the recruiters based on the job information.</thought>
  <agents>
    <agent name="Browser">
      <task>Open Boss Zhipin, find 10 operation positions in Chengdu, and send a personal introduction to the recruiters based on the page information.</task>
      <nodes>
        <node>Open Boss Zhipin, enter the job search page</node>
        <node>Set the regional filter to Chengdu and search for operational positions.</node>
        <node>Brows the job list and filter out 10 suitable operation positions.</node>
        <forEach items="list">
          <node>Analyze job requirements</node>
          <node>Send a self-introduction to the recruiter</node>
        </forEach>
      </nodes>
    </agent>
  </agents>
</root>`,
  `User: Every morning, help me collect the latest AI news, summarize it, and send it to the "AI Daily Morning Report" group chat on WeChat.
Output result:
<root>
  <name>AI Daily Morning Report</name>
  <thought>OK, the user needs to collect the latest AI news every morning, summarize it, and send it to a WeChat group named "AI Daily Morning Report" This requires automation, including the steps of data collection, processing, and distribution.</thought>
  <agents>
    <agent name="Timer">
      <task>Timing: every morning</task>
    </agent>
    <agent name="Browser">
      <task>Search for the latest updates on AI</task>
      <nodes>
        <node>Open Google</node>
        <node>Search for the latest updates on AI</node>
        <forEach items="list">
          <node>View Details</node>
        </forEach>
        <node output="summaryInfo">Summarize search information</node>
      </nodes>
    </agent>
    <agent name="Computer">
      <task>Send a message to the WeChat group chat "AI Daily Morning Report"</task>
      <nodes>
        <node>Open WeChat</node>
        <node>Search for the "AI Daily Morning Report" chat group</node>
        <node input="summaryInfo">Send summary message</node>
      </nodes>
    </agent>
  </agents>
</root>`,
  `User: Access the Google team's organization page on GitHub, extract all developer accounts from the team, and compile statistics on the countries and regions where these developers are located.
Output result:
<root>
  <name>Statistics of Google Team Developers' Geographic Distribution</name>
  <thought>Okay, I need to first visit GitHub, then find Google's organization page on GitHub, extract the team member list, and individually visit each developer's homepage to obtain location information for each developer. This requires using a browser to complete all operations.</thought>
  <agents>
    <agent name="Browser">
      <task>Visit Google GitHub Organization Page and Analyze Developer Geographic Distribution</task>
      <nodes>
        <node>Visit https://github.com/google</node>
        <node>Click "People" tab to view team members</node>
        <node>Scroll the page to load all developer information</node>
        <node output="developers">Extract all developer account information</node>
        <forEach items="developers">
          <node>Visit developer's homepage</node>
          <node>Extract developer's location information</node>
        </forEach>
        <node>Compile and analyze the geographic distribution data of all developers</node>
      </nodes>
    </agent>
  </agents>
</root>`,
  `User: Open Discord to monitor messages in Group A, and automatically reply when new messages are received.
Output result:
<root>
  <name>Automatic reply to Discord messages</name>
  <thought>OK, monitor the chat messages in Discord group A and automatically reply.</thought>
  <agents>
    <agent name="Browser">
      <task>Open Group A in Discord</task>
      <nodes>
        <node>Open Discord page</node>
        <node>Find and open Group A</node>
        <watch event="dom" loop="true">
          <description>Monitor new messages in group chat</description>
          <trigger>
            <node>Analyze message content</node>
            <node>Automatic reply to new messages</node>
          </trigger>
        </watch>
      </nodes>
    </agent>
  </agents>
</root>`,
];

const PLAN_USER_TEMPLATE = `
User Platform: {platform}
Task Description: {taskPrompt}
`;

const PLAN_USER_TASK_WEBSITE_TEMPLATE = `
User Platform: {platform}
Task Website: {task_website}
Task Description: {taskPrompt}
`;

export function getPlanSystemPrompt(agents: Agent[]): string {
  let agents_prompt = agents
    .map((agent) => {
      return (
        `<agent name="${agent.Name}">\n` +
        `Description: ${agent.PlanDescription || agent.Description}\nTools:\n` +
        agent.Tools.filter((tool) => !tool.noPlan)
          .map(
            (tool) =>
              `- ${tool.name}: ${
                tool.planDescription || tool.description || ""
              }`
          )
          .join("\n") +
        `\n</agent>`
      );
    })
    .join("\n\n");
  let example_prompt = "";
  let hasChatAgent = agents.filter((a) => a.Name == chat_agent_name).length > 0;
  const example_list = hasChatAgent
    ? [PLAN_CHAT_EXAMPLE, ...PLAN_EXAMPLE_LIST]
    : [...PLAN_EXAMPLE_LIST];
  for (let i = 0; i < example_list.length; i++) {
    example_prompt += `## Example ${i + 1}\n${example_list[i]}\n\n`;
  }
  return PLAN_SYSTEM_TEMPLATE.replace("{name}", config.name)
    .replace("{agents}", agents_prompt)
    .replace("{datetime}", new Date().toISOString())
    .replace("{example_prompt}", example_prompt)
    .trim();
}

export function getPlanUserPrompt(
  taskPrompt: string,
  task_website?: string
): string {
  if (task_website) {
    return PLAN_USER_TASK_WEBSITE_TEMPLATE.replace("{taskPrompt}", taskPrompt)
      .replace("{platform}", config.platform)
      .replace("{task_website}", task_website)
      .trim();
  } else {
    return PLAN_USER_TEMPLATE.replace("{taskPrompt}", taskPrompt)
      .replace("{platform}", config.platform)
      .trim();
  }
}
```

2. 各节点的 prompt：
```ts
import { Agent } from "../agent";
import config from "../config";
import Context from "../core/context";
import { sub } from "../common/utils";
import { WorkflowAgent, Tool } from "../types";
import { buildAgentRootXml } from "../common/xml";
import { TOOL_NAME as foreach_task } from "../tools/foreach_task";
import { TOOL_NAME as watch_trigger } from "../tools/watch_trigger";
import { TOOL_NAME as human_interact } from "../tools/human_interact";
import { TOOL_NAME as variable_storage } from "../tools/variable_storage";
import { TOOL_NAME as task_node_status } from "../tools/task_node_status";

const AGENT_SYSTEM_TEMPLATE = `
You are {name}, an autonomous AI agent for {agent} agent.
UTC datetime: {datetime}

# Task Description
{description}
{prompt}

# User input task instructions
<root>
  <!-- Main task, completed through the collaboration of multiple Agents -->
  <mainTask>main task</mainTask>
  <!-- The tasks that the current agent needs to complete, the current agent only needs to complete the currentTask -->
  <currentTask>specific task</currentTask>
  <!-- Complete the corresponding step nodes of the task -->
  <nodes>
    <!-- node supports input/output variables to pass dependencies -->
    <node input="variable name" output="variable name" status="todo / done">task step node</node>{nodePrompt}
  </nodes>
</root>
`;

const HUMAN_PROMPT = `
* HUMAN INTERACT
During the task execution process, you can use the \`${human_interact}\` tool to interact with humans, please call it in the following situations:
- When performing dangerous operations such as deleting files, confirmation from humans is required
- When encountering obstacles while accessing websites, such as requiring user login, you need to request human assistance
- When requesting login, please only call the function when a login dialog box is clearly displayed.
- Try not to use the \`${human_interact}\` tool
`;

const VARIABLE_PROMPT = `
* VARIABLE STORAGE
If you need to read and write the input/output variables in the node, require the use of the \`${variable_storage}\` tool.
`;

const FOR_EACH_NODE = `
    <!-- duplicate task node, items support list and variable -->
    <forEach items="list or variable name">
      <node>forEach item step node</node>
    </forEach>`;

const FOR_EACH_PROMPT = `
* forEach node
repetitive tasks, when executing to the forEach node, require the use of the \`${foreach_task}\` tool.
`;

const WATCH_NODE = `
    <!-- monitor task node, the loop attribute specifies whether to listen in a loop or listen once -->
    <watch event="dom or file" loop="true">
      <description>Monitor task description</description>
      <trigger>
        <node>Trigger step node</node>
        <node>...</node>
      </trigger>
    </watch>`;

const WATCH_PROMPT = `
* watch node
monitor changes in webpage DOM or file content, when executing to the watch node, require the use of the \`${watch_trigger}\` tool.
`;

export function getAgentSystemPrompt(
  agent: Agent,
  agentNode: WorkflowAgent,
  context: Context,
  tools?: Tool[],
  extSysPrompt?: string
): string {
  let prompt = "";
  let nodePrompt = "";
  tools = tools || agent.Tools;
  let agentNodeXml = agentNode.xml;
  let hasWatchNode = agentNodeXml.indexOf("</watch>") > -1;
  let hasForEachNode = agentNodeXml.indexOf("</forEach>") > -1;
  let hasHumanTool = tools.filter((tool) => tool.name == human_interact).length > 0;
  let hasVariable =
    agentNodeXml.indexOf("input=") > -1 ||
    agentNodeXml.indexOf("output=") > -1 ||
    tools.filter((tool) => tool.name == variable_storage).length > 0;
  if (hasHumanTool) {
    prompt += HUMAN_PROMPT;
  }
  if (hasVariable) {
    prompt += VARIABLE_PROMPT;
  }
  if (hasForEachNode) {
    if (tools.filter((tool) => tool.name == foreach_task).length > 0) {
      prompt += FOR_EACH_PROMPT;
    }
    nodePrompt += FOR_EACH_NODE;
  }
  if (hasWatchNode) {
    if (tools.filter((tool) => tool.name == watch_trigger).length > 0) {
      prompt += WATCH_PROMPT;
    }
    nodePrompt += WATCH_NODE;
  }
  if (extSysPrompt && extSysPrompt.trim()) {
    prompt += "\n" + extSysPrompt.trim() + "\n";
  }
  if (context.chain.agents.length > 1) {
    prompt += "\n Main task: " + context.chain.taskPrompt;
    prompt += "\n# Pre-task execution results";
    for (let i = 0; i < context.chain.agents.length; i++) {
      let agentChain = context.chain.agents[i];
      if (agentChain.agentResult) {
        prompt += `\n## ${
          agentChain.agent.task || agentChain.agent.name
        }\n${sub(agentChain.agentResult, 500)}`;
      }
    }
  }
  if (prompt) {
    prompt = "\n" + prompt.trim();
  }
  return AGENT_SYSTEM_TEMPLATE.replace("{name}", config.name)
    .replace("{agent}", agent.Name)
    .replace("{description}", agent.Description)
    .replace("{datetime}", new Date().toISOString())
    .replace("{prompt}", prompt)
    .replace("{nodePrompt}", nodePrompt)
    .trim();
}

export function getAgentUserPrompt(
  agent: Agent,
  agentNode: WorkflowAgent,
  context: Context,
  tools?: Tool[]
): string {
  let hasTaskNodeStatusTool =
    (tools || agent.Tools).filter((tool) => tool.name == task_node_status)
      .length > 0;
  return buildAgentRootXml(
    agentNode.xml,
    context.chain.taskPrompt,
    (nodeId, node) => {
      if (hasTaskNodeStatusTool) {
        node.setAttribute("status", "todo");
      }
    }
  );
}
```

## 其他比较有价值的 PR

1. [#36](https://github.com/FellouAI/eko/pull/36)：增加了 Human Tools，通过 callbacks 实现在工作流执行中和用户交互的方法。在 v0 和 v1 中都是 4 个独立的工具，在 v2 中被整合成一个工具。
2. [#37](https://github.com/FellouAI/eko/pull/37) & [#88](https://github.com/FellouAI/eko/pull/88)：实现了工作流结束时的总结功能，一开始的实现是添加一个额外的工具，并在 prompt 里提示大模型调用；后来撤回了这个修改，改成了工作流结束后将完整执行过程发给单独的 Agent 让它总结。
3. [#132](https://github.com/FellouAI/eko/pull/132)：用云日志托管平台来实现可在线查看的日志，使用 uuidv4 跟踪每个 Workflow 实例的执行过程。
4. [#151](https://github.com/FellouAI/eko/pull/151)：通过一系列方法限制了 token 消耗，包括上下文压缩、限制工具输出长度、限制对话轮数、移除 input schema 补丁的 p2。

## 一些见解

1. 在 ReAct 范式下，上下文是线性增长的。在这个背景下，解决越来越复杂的任务，首先要解决越来越长的上下文。这里的上下文包括：
   1. system prompt & tools：每个 Agent 都有自己负责的一块 domain，并有与之配对的 system prompt 和 tools。
   2. user prompt：Workflow 的输入，分为用户输入的 prompt 和大模型生成的 prompt。
   3. tool call & results：每次调用工具时的输入和输出。
2. 有很多方法可以解决上下文越来越长的问题：
   1. 上下文压缩：对 tool call & results 做压缩，但是 1) 不能保证不删除关键信息；2) 不能解决 system prompt & tools 随着问题空间变大、答案验证更细致而越来越长的问题。
   2. Multi-Agent：按 domain 拆分 system prompt & tools，进而自动拆分了 user prompt 和 tool call & results。
