---
sidebar_position: 1
---

# Runtime

The Runtime is the core building block of MAIAR's plugin system. It manages the execution of plugins, handles the event queue, and provides essential operations for plugins to interact with the model and memory services.

## Runtime Overview

The Runtime serves as the central coordinator for all MAIAR components:

1. **Plugin Management** - Registers, initializes, and orchestrates plugins
2. **Pipeline Construction** - Dynamically builds execution pipelines based on context
3. **Model Orchestration** - Coordinates model interactions and capabilities
4. **Memory Integration** - Manages conversation history and context
5. **Event Handling** - Processes user inputs and system events

## Creating a Runtime

To create a runtime, use the `createRuntime` function with your desired configuration:

```typescript
import { createRuntime } from "@maiar-ai/core";
import { OpenAIProvider } from "@maiar-ai/model-openai";
import { SQLiteProvider } from "@maiar-ai/memory-sqlite";
import { PluginTerminal } from "@maiar-ai/plugin-terminal";
import { PluginText } from "@maiar-ai/plugin-text";

const runtime = createRuntime({
  // Model providers that handle text generation, image generation, etc.
  models: [
    new OpenAIProvider({
      apiKey: process.env.OPENAI_API_KEY,
      models: ["gpt-4o"]
    })
  ],
  
  // Memory provider for conversation persistence
  memory: new SQLiteProvider({
    dbPath: "./data/conversations.db"
  }),
  
  // Plugins that extend functionality
  plugins: [
    new PluginText(),
    new PluginTerminal()
  ],
  
  // Optional monitoring providers
  monitor: [
    new ConsoleMonitorProvider()
  ],
  
  // Optional capability aliases (alternative names for capabilities)
  capabilityAliases: [
    ["text-generation", "generate-text"]
  ]
});

// Start the runtime to initialize all components
await runtime.start();
```

## Runtime Architecture

The MAIAR Runtime follows a structured architecture to manage interactions between components:

```
┌────────────────────────────────────────────────────────────────┐
│                        MAIAR Runtime                           │
├────────────┬────────────┬─────────────┬────────────┬───────────┤
│            │            │             │            │           │
│  Plugin    │   Model    │   Memory    │  Monitor   │  Event    │
│  Registry  │  Service   │   Service   │  Service   │  Queue    │
│            │            │             │            │           │
└────────────┴────────────┴─────────────┴────────────┴───────────┘
        ▲            ▲           ▲            ▲           ▲
        │            │           │            │           │
        ▼            ▼           ▼            ▼           ▼
┌────────────┐ ┌────────────┐ ┌─────────────┐ ┌────────────┐ ┌───────────┐
│            │ │            │ │             │ │            │ │           │
│  Plugins   │ │   Models   │ │  Memory     │ │  Monitors  │ │  Contexts │
│            │ │            │ │  Providers  │ │            │ │           │
└────────────┘ └────────────┘ └─────────────┘ └────────────┘ └───────────┘
```

## Core Runtime Components

### 1. Plugin Registry

The Plugin Registry manages all installed plugins and their lifecycle:

```typescript
// Register a new plugin dynamically
await runtime.registerPlugin(new MyCustomPlugin());

// Get all registered plugins
const plugins = runtime.getPlugins();
```

#### Lifecycle Methods

- `registerPlugin(plugin)` - Register a new plugin
- `getPlugin(id)` - Get a specific plugin by ID
- `getPlugins()` - Get all registered plugins

### 2. Model Service

The Model Service manages model providers and their capabilities:

```typescript
// Execute a capability directly
const text = await runtime.executeCapability(
  "text-generation",  // Capability ID
  "Generate a poem about AI",  // Input
  {
    temperature: 0.7,  // Optional config
    maxTokens: 500
  }
);
```

The Model Service automatically routes capability requests to the appropriate model provider based on capability IDs and aliases.

### 3. Memory Service

The Memory Service provides persistent storage for conversations:

```typescript
// Store a user message
await runtime.memory.storeUserInteraction(
  "user123",  // User ID
  "discord",  // Platform
  "Hello AI!",  // Message content
  Date.now(),  // Timestamp
  "msg123"  // Optional message ID
);

// Get conversation history
const history = await runtime.memory.getRecentConversationHistory(
  "user123",  // User ID
  "discord",  // Platform
  10  // Limit (optional)
);
```

### 4. Monitor Service

The Monitor Service broadcasts events to registered monitor providers:

```typescript
// Monitor events are published automatically, but can be manually triggered
MonitorService.publishEvent({
  type: "custom_event",
  message: "Something important happened",
  metadata: {
    key: "value"
  }
});
```

### 5. Event Queue

The Event Queue manages the flow of events through the system:

```typescript
// Create a new event
await runtime.createEvent({
  // Initial context data
  id: `${this.id}-${Date.now()}`,
  pluginId: this.id,
  action: "receive_message",
  type: "user_input",
  content: message,
  timestamp: Date.now(),
  rawMessage: message,
  user: "user123"
}, {
  // Optional platform context
  platform: "telegram",
  responseHandler: (result) => sendTelegramMessage(chatId, result),
  metadata: {
    chatId: chatId,
    messageId: messageId
  }
});
```

## The Pipeline Execution Model

One of MAIAR's most powerful features is its dynamic pipeline execution. Here's how it works:

1. **Event Trigger** - A user message or system event creates an initial context
2. **Pipeline Generation** - The runtime uses the model to generate an execution plan
3. **Plugin Execution** - Plugins are executed in sequence according to the plan
4. **Context Transformation** - Each step enhances the context with new information
5. **Dynamic Modification** - The pipeline can be modified during execution based on new information
6. **Response Generation** - The final context is used to generate a response

### Pipeline Generation

When an event occurs, the runtime generates a pipeline like this:

```typescript
// Example of a dynamically generated pipeline
[
  { pluginId: "plugin-character", action: "inject_character" },
  { pluginId: "plugin-text", action: "generate_text" },
  { pluginId: "plugin-terminal", action: "send_response" }
]
```

This pipeline is created based on:
- Available plugins and their executors
- The user's message and conversation history
- The current context chain

### Pipeline Execution

The runtime executes each step in the pipeline, creating a chain of context items:

```typescript
// Initial context
[{ type: "user_input", content: "What time is it?" }]

// After character injection
[
  { type: "user_input", content: "What time is it?" },
  { type: "character", character: "I am a helpful assistant" }
]

// After time plugin
[
  { type: "user_input", content: "What time is it?" },
  { type: "character", character: "I am a helpful assistant" },
  { type: "current_time", currentTime: "2025-03-28T15:30:45" }
]

// After text generation
[
  { type: "user_input", content: "What time is it?" },
  { type: "character", character: "I am a helpful assistant" },
  { type: "current_time", currentTime: "2025-03-28T15:30:45" },
  { type: "generated_text", text: "The current time is 3:30 PM." }
]
```

## Operations API

The Runtime provides several essential operations that plugins can utilize:

### getObject

The `getObject` operation extracts structured data using Zod schemas:

```typescript
const schema = z.object({
  city: z.string().describe("The name of the city"),
  country: z.string().describe("The name of the country"),
  population: z.number().optional().describe("The population of the city")
});

// Extract structured data from text
const locationData = await runtime.operations.getObject(
  schema,
  "Extract the city and country from: 'I live in Paris, France'",
  { temperature: 0.1 }
);

// Result: { city: "Tokyo", country: "Japan"}
```

The `getObject` operation:
- Uses Zod schemas for type safety and validation
- Handles retries automatically for malformed responses
- Formats schema descriptions for the model
- Can extract complex nested objects

### executeCapability

The `executeCapability` operation invokes model capabilities:

```typescript
// Execute text generation
const text = await runtime.operations.executeCapability(
  "text-generation",  // Capability ID
  "Write a haiku about mountains",  // Input
  {
    temperature: 0.7,  // Optional configuration
    maxTokens: 100
  }
);

// Execute image generation
const imageUrls = await runtime.operations.executeCapability(
  "image-generation",  // Capability ID
  "A serene mountain landscape at sunset",  // Input
  {
    n: 1,  // Generate one image
    size: "1024x1024"  // Image size
  }
);
```

## Context Management

The Context Chain is central to MAIAR's operation, representing the ongoing state of a conversation or process:

```typescript
// Access current context in a plugin executor
execute: async (context: AgentContext): Promise<PluginResult> => {
  // Get existing context items
  const userInput = context.contextChain.find(item => item.type === "user_input");
  
  // Add new context data
  context.contextChain.push({
    id: `${this.id}-${Date.now()}`,
    pluginId: this.id,
    action: "weather_data",
    type: "weather",
    content: JSON.stringify(weatherData),
    timestamp: Date.now(),
    location: userInput?.location,
    temperature: weatherData.temperature,
    conditions: weatherData.conditions,
    forecast: weatherData.forecast
  });
  
  return { success: true };
}
```

### Context Chain Structure

Each context item follows this pattern:

```typescript
interface BaseContextItem {
  id: string;           // Unique identifier
  pluginId: string;     // Source plugin
  action: string;       // Action that created it
  type: string;         // Type for model understanding
  content: string;      // Serialized content
  timestamp: number;    // Creation time
  [key: string]: any;   // Additional data
}
```

## Pipeline Modification

The Runtime can dynamically modify pipelines during execution:

```typescript
// Example of pipeline modification during execution
// This happens automatically based on context changes
{
  shouldModify: true,
  explanation: "User asked for weather information, need to add weather plugin step",
  modifiedSteps: [
    { pluginId: "plugin-weather", action: "get_weather" },
    { pluginId: "plugin-text", action: "generate_text" }
  ]
}
```

## Error Handling

The Runtime provides robust error handling mechanisms:

```typescript
try {
  await runtime.start();
} catch (error) {
  // Handle startup errors
  console.error("Failed to start runtime:", error);
  process.exit(1);
}

// Plugin execution errors are added to the context chain
{
  id: `error-${Date.now()}`,
  pluginId: "plugin-weather",
  type: "error",
  action: "get_weather",
  content: "API key not configured",
  timestamp: Date.now(),
  error: "API key not configured",
  failedStep: { pluginId: "plugin-weather", action: "get_weather" }
}
```

Error types include:
- **Initialization Errors** - Problems during startup
- **Plugin Execution Errors** - Issues when executing a plugin
- **Pipeline Generation Errors** - Failures when creating execution plans
- **Model Capability Errors** - Issues when invoking model capabilities
- **Memory Service Errors** - Problems with persistence operations

## Advanced Uses

### Custom Monitor Integration

```typescript
import { createRuntime } from "@maiar-ai/core";
import { WebSocketMonitorProvider } from "@maiar-ai/monitor-websocket";
import { ConsoleMonitorProvider } from "@maiar-ai/monitor-console";

const runtime = createRuntime({
  // ... other configuration
  monitors: [
    new ConsoleMonitorProvider(),  // Terminal output
    new WebSocketMonitorProvider({ port: 3001 })  // Dashboard
  ]
});
```

### Multiple Model Integration

```typescript
import { createRuntime } from "@maiar-ai/core";
import { OpenAIProvider } from "@maiar-ai/model-openai";
import { OllamaProvider } from "@maiar-ai/model-ollama";

const runtime = createRuntime({
  models: [
    // Primary model for text generation
    new OpenAIProvider({
      apiKey: process.env.OPENAI_API_KEY,
      models: ["gpt-4o"]
    }),
    // Local model for image generation
    new OllamaProvider({
      baseUrl: "http://localhost:11434",
      model: "llama2"
    })
  ],
  // ... other configuration
});
```

### Custom Event Scheduling

```typescript
// Schedule periodic events
setInterval(async () => {
  await runtime.createEvent({
    pluginId: "plugin-scheduler",
    action: "hourly_check",
    timestamp: Date.now(),
    type: "scheduled_event",
    content: "Hourly system check",
    user: "system"
  });
}, 60 * 60 * 1000); // Every hour
```

## Complete Runtime Example

Here's a comprehensive example of setting up a runtime with multiple plugins, models, and error handling:

```typescript
import { createRuntime } from "@maiar-ai/core";
import { OpenAIProvider } from "@maiar-ai/model-openai";
import { SQLiteProvider } from "@maiar-ai/memory-sqlite";
import { ConsoleMonitorProvider } from "@maiar-ai/monitor-console";
import { WebSocketMonitorProvider } from "@maiar-ai/monitor-websocket";
import { PluginTerminal } from "@maiar-ai/plugin-terminal";
import { PluginText } from "@maiar-ai/plugin-text";
import { PluginTime } from "@maiar-ai/plugin-time";
import { PluginCharacter } from "@maiar-ai/plugin-character";
import path from "path";

async function main() {
  // Create data directories if they don't exist
  const dataDir = path.join(process.cwd(), "data");
  await fs.promises.mkdir(dataDir, { recursive: true });
  
  try {
    // Create runtime with multiple components
    const runtime = createRuntime({
      // Configure models
      models: [
        new OpenAIProvider({
          apiKey: process.env.OPENAI_API_KEY,
          models: ["gpt-4o"]
        })
      ],
      
      // Configure memory
      memory: new SQLiteProvider({
        dbPath: path.join(dataDir, "conversations.db")
      }),
      
      // Configure plugins
      plugins: [
        new PluginCharacter({
          character: "I am MAIAR, a helpful AI assistant that specializes in providing accurate, thoughtful responses."
        }),
        new PluginText(),
        new PluginTime(),
        new PluginTerminal({
          user: "user",
          agentName: "MAIAR"
        })
      ],
      
      // Configure monitors
      monitors: [
        new ConsoleMonitorProvider(),
        new WebSocketMonitorProvider({ port: 3001 })
      ]
    });

    // Start the runtime
    console.log("Starting MAIAR runtime...");
    await runtime.start();
    console.log("MAIAR runtime started successfully!");

    // Handle graceful shutdown
    const shutdown = async () => {
      console.log("Shutting down MAIAR runtime...");
      await runtime.stop();
      console.log("MAIAR runtime stopped safely.");
      process.exit(0);
    };

    // Register shutdown handlers
    process.on("SIGINT", shutdown);
    process.on("SIGTERM", shutdown);
    
  } catch (error) {
    console.error("Failed to start MAIAR runtime:", error);
    process.exit(1);
  }
}

// Start the application
main().catch(error => {
  console.error("Unhandled error:", error);
  process.exit(1);
});
```

## Best Practices

### Runtime Initialization

- **Order Matters**: Initialize models before plugins that depend on them
- **Error Handling**: Always wrap runtime initialization in try/catch blocks
- **Memory Configuration**: Ensure memory directories exist before initialization
- **Graceful Shutdown**: Implement proper shutdown handlers

### Plugin Development

- **Context Awareness**: Design plugins to read from and enhance the context chain
- **Focused Responsibility**: Each plugin should have a clear, single responsibility
- **Error Recovery**: Handle errors gracefully and provide useful error information
- **Documentation**: Document plugin requirements and context enhancements

### Model Integration

- **Temperature Settings**: Use lower temperatures (0.1-0.3) for structured data extraction
- **Capability Selection**: Choose the appropriate capability for each task
- **Error Handling**: Handle model errors and provide fallbacks where possible

### Event Management

- **Resource Control**: Monitor event queue length to prevent memory issues
- **Context Cleanup**: Remove unnecessary context data when no longer needed
- **Error Propagation**: Ensure errors in one event don't affect others

### Monitoring

- **Event Logging**: Use monitor providers to track important state changes
- **Performance Tracking**: Monitor execution times for optimization
- **Debugging**: Use the console monitor during development for visibility

## Next Steps

- Learn about [createEvent](./createEvent) for event handling
- Explore [Building Plugins](../building-plugins/philosophy) for creating custom plugins
- Check out [Model Providers](../model-providers/overview) for model configuration
- Read about [Executors](../building-plugins/executors) for practical plugin usage examples
- See [Memory Providers](../memory-providers/overview) for persistent storage options
- Understand [Monitor Providers](../monitor-providers/overview) for observability options
