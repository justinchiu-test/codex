# Cohere SDK Integration Plan

## Overview

This document outlines the complete process for integrating the Cohere SDK into codex-cli, including implementation details, testing procedures, and troubleshooting steps.

## Implementation Steps

### 1. Initial Setup

#### Install Cohere SDK

```bash
npm install cohere-ai
```

#### Environment Variables

```bash
export COHERE_API_KEY="your-cohere-api-key"
```

### 2. Core Integration

#### 2.1 Import Cohere Client

In `src/utils/agent/agent-loop.ts`:

```typescript
import { CohereClientV2 } from "cohere-ai";
```

#### 2.2 Initialize Cohere Client

Add to the AgentLoop constructor:

```typescript
// Initialize Cohere client if using Cohere provider
if (this.config.provider?.toLowerCase().includes("cohere")) {
  const cohereApiKey = process.env.COHERE_API_KEY;
  if (!cohereApiKey) {
    throw new Error(
      "COHERE_API_KEY environment variable is required for Cohere provider",
    );
  }

  const cohereBaseUrl =
    getBaseUrl(this.config.provider) || "https://api.cohere.ai";
  this.cohere = new CohereClientV2({
    token: cohereApiKey,
    clientName: "codex-cli",
  });
}
```

#### 2.3 Provider Configuration

Cohere providers are configured in `src/utils/providers.ts`:

- `cohere`: Production API (https://api.cohere.ai)
- `coherestaging`: Staging API (https://stg.api.cohere.ai/compatibility/v1)

### 3. API Integration

#### 3.1 Main API Call Flow

In the `run()` method, add Cohere API handling:

```typescript
if (this.config.provider?.toLowerCase().includes("cohere")) {
  const cohereResponse = await this.callCohereAPI(turnInput, instructions);
  // Process response...
}
```

#### 3.2 Message Format Conversion

Cohere V2 uses a different message format than OpenAI:

```typescript
private convertToCohereV2Messages(
  turnInput: Array<ResponseInputItem>,
  instructions: string,
): Array<{
  role: string;
  content?: string;
  toolCalls?: Array<{
    id: string;
    type: string;
    function: {
      name: string;
      arguments: Record<string, unknown>;
    };
  }>;
  toolResults?: Array<{
    call: {
      name: string;
      parameters: Record<string, unknown>;
    };
    outputs: Array<Record<string, unknown>>;
  }>;
  toolCallId?: string;
}>
```

Key differences:

- System instructions go in the first message
- Tool results use `outputs` array format
- Tool calls have nested `function` structure
- Arguments are objects (not JSON strings)

#### 3.3 Tool Format Conversion

Cohere V2 tools use OpenAI-compatible format:

```typescript
private convertToCohereV2Tools(): Array<{
  type: string;
  function: {
    name: string;
    description: string;
    parameters?: Record<string, unknown>;
  };
}> {
  return [
    {
      type: "function",
      function: {
        name: "shell",
        description: "Runs a shell command, and returns its output.",
        parameters: {
          type: "object",
          properties: {
            command: {
              type: "array",
              items: { type: "string" },
              description: "The command to execute as an array of strings",
            },
            workdir: {
              type: "string",
              description: "The working directory for the command.",
            },
            timeout: {
              type: "number",
              description: "The maximum time to wait for the command to complete in milliseconds.",
            },
          },
          required: ["command"],
          additionalProperties: false,
        },
      },
    },
  ];
}
```

#### 3.4 Response Processing

Convert Cohere responses to OpenAI format:

```typescript
private convertCohereResponseToStream(response: {
  finish_reason?: string;
  text?: string;
  message?: {
    toolPlan?: string;  // Chain-of-thought explanation
    toolCalls?: Array<{
      id: string;
      type?: string;
      function: {
        name: string;
        arguments: string;  // JSON string
      };
    }>;
    content?: Array<{ text?: string }>;
  };
  error?: string;
})
```

Key features:

- Handle `toolPlan` for chain-of-thought reasoning
- Extract tool calls from `message.toolCalls`
- Convert arguments to JSON strings if needed
- Create appropriate message items for UI

### 4. Error Handling

#### 4.1 Common Errors and Solutions

**422 Unprocessable Entity - Invalid Tool Generation**

- Some Cohere models/endpoints don't support tools
- Solution: Catch 422 errors and retry without tools

```typescript
catch (error) {
  if (err.status === 422 && err.message?.includes("invalid tool generation")) {
    // Retry without tools
    const response = await this.cohere.chat({
      model: this.model,
      messages: cohereMessages,
      // No tools parameter
    });
    return this.convertCohereResponseToStream(response);
  }
  throw error;
}
```

**Missing Required Keys**

- V1 vs V2 API format mismatch
- Solution: Use V2 message format with `messages` array

**Unrecognized Role**

- Cohere uses lowercase roles ("user", "assistant", "system", "tool")
- OpenAI uses uppercase in some contexts
- Solution: Convert to lowercase in message conversion

### 5. Testing

#### 5.1 Test Script for Tool Calling

Create `test-cohere-tools.js`:

```javascript
const { CohereClient } = require("cohere-ai");

async function testToolCalling() {
  const cohere = new CohereClient({
    token: process.env.COHERE_API_KEY,
    clientName: "tool-test",
  });

  const tools = [
    {
      name: "sales_database",
      description: "Connects to a database about sales volumes",
      parameterDefinitions: {
        day: {
          description:
            "Retrieves sales data from this day, formatted as YYYY-MM-DD.",
          type: "str",
          required: true,
        },
      },
    },
  ];

  // Test tool calling
  const response = await cohere.chat({
    message: "How good were the sales on September 29?",
    tools,
  });

  console.log("Tool calls:", response.toolCalls);
}

testToolCalling();
```

#### 5.2 Testing with codex-cli

```bash
# Build the project
npm run build

# Test with Cohere provider
node dist/cli.js -m command-r --provider cohere "List the files in the current directory"

# Test with staging provider
node dist/cli.js -m c3-sweep-ecsydrkq-690h-fp16 --provider coherestaging "Run ls -la and show me the output"

# Test tool plans
node dist/cli.js -m command-r --provider cohere "Find all JavaScript files and count how many lines of code they have"
```

#### 5.3 Verify Tool Plans in History

Tool plans (Cohere's chain-of-thought explanations) are:

- Displayed as assistant messages before tool execution
- Saved to conversation history/transcript
- Available as context for future turns

### 6. Key Implementation Details

#### 6.1 Tool Result Format

Cohere expects tool results in this format:

```typescript
{
  call: {
    name: string;
    parameters: Record<string, unknown>;
  },
  outputs: Array<Record<string, unknown>>;
}
```

The `outputs` array allows multiple result objects, which is useful for commands that produce structured output.

#### 6.2 Streaming Support

Currently implemented as non-streaming (using `cohere.chat()`). For streaming support, use `cohere.chatStream()` and handle the event stream appropriately.

#### 6.3 Model Compatibility

Not all Cohere models support tools. Known working models:

- `command-r`
- `command-r-plus`
- `c3-sweep-*` models (after V2 format fix)

### 7. Troubleshooting

#### Commands Not Producing Output

1. Check if the model supports tools
2. Verify tool format is correct (V2 format)
3. Check response parsing for `message.toolCalls`
4. Enable debug logging to see raw API responses

#### 422 UnprocessableEntity Error in Interactive Mode

If you get a 422 error "your request resulted in an invalid tool generation":

1. **Model Compatibility** - Not all Cohere models support tools. The error often means:
   - The specific model doesn't support tool/function calling
   - The model is still in development and tools aren't enabled yet
   - Try using `command-r` or `command-r-plus` with the production provider
2. **Disable tools temporarily** - You can disable tools for testing:

   ```bash
   export COHERE_DISABLE_TOOLS=1
   node dist/cli.js -m c3-sweep-ecsydrkq-690h-fp16 --provider coherestaging "Hello"
   ```

3. **Tool format for V2** - Cohere V2 uses OpenAI-compatible format:

   ```typescript
   {
     type: "function",
     function: {
       name: "shell",
       description: "Runs a shell command",
       parameters: {
         type: "object",
         properties: {
           command: {
             type: "string",
             description: "The command to execute"
           }
         },
         required: ["command"]
       }
     }
   }
   ```

4. **Check logs for details** - The logs are in `$TMPDIR/oai-codex/`:

   ```bash
   tail -f $TMPDIR/oai-codex/codex-cli-latest.log
   ```

   Look for:

   - "Using native Cohere V2 API" to confirm V2 is being used
   - "Cohere chat params" to see the exact request
   - Error details with the 422 response

5. **Known working models**:
   - `command-r` and `command-r-plus` (with production provider)
   - Some experimental models (like c3-sweep-\*) may not support tools yet

#### ESLint Errors

Run `npm run lint` and fix:

- Unused imports
- Missing type annotations
- Array syntax (`Array<T>` instead of `T[]`)
- Prefix unused parameters with underscore

#### API Authentication

- Ensure `COHERE_API_KEY` is set
- Check if using correct base URL for provider
- Verify API key has necessary permissions

### 8. Future Enhancements

1. **Streaming Support**: Implement `chatStream()` for real-time responses
2. **Additional Tools**: Add more tool types beyond shell commands
3. **Better Error Messages**: Provide more context-specific error handling
4. **Tool Confirmation**: Add user confirmation for tool execution
5. **Response Caching**: Cache responses when `disableResponseStorage` is false

## Summary

The Cohere integration adds support for Cohere's language models to codex-cli, with full tool calling capabilities using the V2 API format. The implementation handles format conversion between OpenAI and Cohere APIs, preserves tool plans in conversation history, and provides robust error handling for common issues.
