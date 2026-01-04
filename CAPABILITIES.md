# LLMs API Server - Capabilities & Architecture

> A universal LLM API transformation server that acts as a middleware for standardizing requests and responses across different LLM providers.


## 🎯 Core Purpose

This project provides a unified API gateway that translates between different LLM provider APIs (Anthropic, OpenAI, Gemini, Deepseek, etc.), allowing applications to send requests in a standardized format and receive normalized responses regardless of the underlying provider.

---

## 🏗️ Architecture Overview

### Key Components

The system is built on a **modular transformer-based architecture** with these core services:

#### 1. **Server Core** (`src/server.ts`)
- Fastify-based HTTP server with 50MB body limit
- Handles CORS, logging, and error management
- Initializes all services at startup
- Manages graceful shutdown (SIGINT/SIGTERM)
- Default port: 3000, Host: 127.0.0.1

#### 2. **ConfigService** (`src/services/config.ts`)
- Reads configuration from multiple sources (in priority order):
  - JSON5 format configuration file (`config.json`)
  - `.env` environment variables
  - Programmatic initialization
- Supports custom paths and selective loading
- Provides unified configuration access point

#### 3. **TransformerService** (`src/services/transformer.ts`)
- Automatically registers 22+ built-in transformers
- Supports dynamic loading of external transformers via configuration
- Distinguishes between transformers with/without endpoints
- Auto-mounts routes for transformers with defined endpoints
- Manages transformer lifecycle and dependency injection

#### 4. **ProviderService** (`src/services/provider.ts`)
- Manages LLM provider registration, updates, and deletion
- Maintains provider and model routing tables
- Supports CRUD operations via REST API
- Resolves model routes to correct providers
- Handles provider-specific authentication and configuration

#### 5. **LLMService** (`src/services/llm.ts`)
- Thin wrapper around ProviderService
- Provides external API for provider/model queries
- Used by API routes and transformers

---

## 🔄 Request Processing Pipeline

### Complete Request → Response Flow

```
1. Client Request
   ↓
2. Fastify Middleware
   - Parse model field: "provider,model" → req.provider + req.body.model
   - Logging and validation
   ↓
3. Route Resolution
   - TransformerService finds matching transformer
   - ProviderService retrieves provider configuration
   ↓
4. Request Transformation Phase
   a) transformRequestOut
      - Convert provider-specific format → Unified format
   b) Provider-level transformer chain (transformRequestIn)
      - Apply provider-specific transformations
   c) Model-specific transformer chain (transformRequestIn)
      - Apply model-specific transformations
   d) Authentication handling
   ↓
5. HTTP Request to Provider
   - Build headers with API key
   - Support HTTPS proxy if configured
   - Timeout: 60 minutes (configurable)
   - Streaming support via Content-Type headers
   ↓
6. Response Transformation Phase (reversed)
   a) Model-specific transformer chain (transformResponseOut)
   b) Provider-level transformer chain (transformResponseOut)
   c) transformResponseIn
      - Convert provider response → Unified format
   ↓
7. Response Formatting
   - JSON response (non-streaming)
   - SSE stream (if stream: true)
   ↓
8. Client Response
```

### Bypass Optimization

When a provider has only one transformer that matches the current transformer, the transformation chains are bypassed to improve performance while maintaining original request/header information.

---

## 🔌 Built-in Transformers (22 Total)

### Provider-Specific Transformers

| Transformer | Purpose | Endpoint |
|---|---|---|
| **AnthropicTransformer** | Convert to/from Anthropic Claude API format | `/v1/messages` |
| **OpenAITransformer** | Convert to/from OpenAI API format | `/v1/chat/completions` |
| **GeminiTransformer** | Convert to/from Google Gemini API format | `/v1beta/models/:modelAndAction` |
| **VercelTransformer** | Vercel AI SDK compatibility layer | `/v1/chat/completions` |
| **OpenAIResponsesTransformer** | OpenAI Responses API (batch/async) format | `/v1/responses` |
| **VertexGeminiTransformer** | Google Vertex AI Gemini integration | - |
| **VertexClaudeTransformer** | Google Vertex AI Claude integration | - |
| **DeepseekTransformer** | Deepseek API format conversion | - |
| **GroqTransformer** | Groq API format conversion | - |
| **OpenrouterTransformer** | OpenRouter aggregation API format | - |
| **CerebrasTransformer** | Cerebras API format conversion | - |

### Feature/Enhancement Transformers

| Transformer | Purpose |
|---|---|
| **TooluseTransformer** | Tool use and function calling support |
| **EnhanceToolTransformer** | Advanced tool parameter handling |
| **ReasoningTransformer** | Extended reasoning/thinking support |
| **ForceReasoningTransformer** | Force reasoning mode on/off |
| **StreamOptionsTransformer** | Stream behavior configuration |
| **SamplingTransformer** | Sampling parameter normalization |
| **CustomParamsTransformer** | Custom parameter injection |
| **MaxTokenTransformer** | Token limit normalization |
| **MaxCompletionTokens** | Completion token limit handling |
| **CleancacheTransformer** | Cache control management |
| **OpenAIResponsesTransformer** | OpenAI response format normalization |

---

## 📋 Unified Data Types

### Request Format
```typescript
interface UnifiedChatRequest {
  messages: UnifiedMessage[];      // Message history
  model: string;                   // Model identifier
  max_tokens?: number;             // Completion token limit
  temperature?: number;            // Sampling temperature
  stream?: boolean;                // Stream responses
  tools?: UnifiedTool[];           // Function definitions
  tool_choice?: auto|none|required|string|{type,function}; // Tool selection
  reasoning?: {                    // Extended reasoning
    effort?: 'none'|'low'|'medium'|'high';  // OpenAI style
    max_tokens?: number;           // Anthropic style
    enabled?: boolean;
  };
}

interface UnifiedMessage {
  role: 'user'|'assistant'|'system'|'tool';
  content: string | null | MessageContent[];
  tool_calls?: Array<{id, type, function: {name, arguments}}>;
  tool_call_id?: string;
  cache_control?: {type?: string};
  thinking?: {content: string, signature?: string};
}

interface UnifiedTool {
  type: 'function';
  function: {
    name: string;
    description: string;
    parameters: JSONSchema;
  };
}
```

### Response Format
```typescript
interface UnifiedChatResponse {
  id: string;
  model: string;
  content: string | null;
  usage?: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
  };
  tool_calls?: Array<{id, type, function: {name, arguments}}>;
  annotations?: Array<{type: 'url_citation', url_citation: {...}}>;
}
```

### Stream Format
```typescript
interface StreamChunk {
  id: string;
  object: string;
  created: number;
  model: string;
  choices?: Array<{
    index: number;
    delta: {
      role?: string;
      content?: string;
      thinking?: {content?: string, signature?: string};
      tool_calls?: Array<{id, type, function: {name, arguments}}>;
      annotations?: Annotation[];
    };
    finish_reason?: string | null;
  }>;
}
```

---

## 🛣️ API Endpoints

### Health & Info
- `GET /` - API info with version
- `GET /health` - Health check with timestamp

### Provider Management
- `POST /providers` - Register new provider
  - Body: `{name, baseUrl, apiKey, models}`
  - Validates provider doesn't already exist
  
- `GET /providers` - List all providers
- `GET /providers/:id` - Get specific provider
- `PUT /providers/:id` - Update provider configuration
- `DELETE /providers/:id` - Remove provider
- `PATCH /providers/:id/toggle` - Enable/disable provider

### LLM Provider Endpoints
All transformers with an `endPoint` property are automatically mounted as public routes:

**Publicly Exposed Endpoints:**
- `POST /v1/messages` - Anthropic Claude API format
- `POST /v1/chat/completions` - OpenAI Chat Completions format (also used by Vercel adapter)
- `POST /v1beta/models/:modelAndAction` - Google Gemini API format
- `POST /v1/responses` - OpenAI Responses API format (batch/async processing)

**Endpoint Details:**

| Endpoint | Format | Use Case |
|---|---|---|
| `/v1/messages` | Anthropic Claude API | Send Claude/Anthropic-format requests, or route to other providers using `"provider,model"` syntax |
| `/v1/chat/completions` | OpenAI Chat API | Send OpenAI-format requests, compatible with Vercel AI SDK, or route to other providers |
| `/v1beta/models/:modelAndAction` | Google Gemini API | Send native Gemini API requests (generateContent/streamGenerateContent) |
| `/v1/responses` | OpenAI Responses API | Send asynchronous/batch processing requests for extended reasoning and complex operations |

**Operating Modes:**

1. **Native Format Mode**: Send requests directly to the provider's native endpoint
   ```json
   POST /v1beta/models/gemini-1.5-pro:generateContent
   {
     "contents": [...],
     "generationConfig": {...}
   }
   ```

2. **Unified Format Mode**: Send to any endpoint with provider routing via model field
   ```json
   POST /v1/messages
   {
     "model": "gemini,gemini-1.5-pro",
     "messages": [...],
     "max_tokens": 2048
   }
   ```
   The server transforms the request to the target provider's format.

3. **Adapter Mode**: Use Vercel-compatible format
   ```json
   POST /v1/chat/completions
   {
     "model": "gpt-4",
     "messages": [...]
   }
   ```
   Processed by VercelTransformer which handles image encoding and cache control stripping.

---

## ⚙️ Configuration

### Config File Structure (JSON5)
```json5
{
  PORT: "3000",
  HOST: "127.0.0.1",
  HTTPS_PROXY: "http://proxy:8080",
  
  providers: [
    {
      name: "anthropic",
      api_base_url: "https://api.anthropic.com",
      api_key: "${ANTHROPIC_API_KEY}",
      models: ["claude-3-opus", "claude-3-sonnet"],
      transformer: {
        use: [["AnthropicTransformer", {UseBearer: false}]],
        "claude-3-opus": {
          use: ["MaxTokenTransformer", "ReasoningTransformer"]
        }
      }
    }
  ],
  
  transformers: [
    ["ExternalTransformer", {option: "value"}]
  ]
}
```

### Environment Variables
- `PORT` - Server port (default: 3000)
- `HOST` - Server host (default: 127.0.0.1)
- `HTTPS_PROXY` - HTTPS proxy URL
- Any value referenced in config with `${VAR_NAME}` syntax

---

## 🚀 Development & Deployment

### Development Commands
```bash
npm install              # Install dependencies
npm run dev             # Hot-reload development server (nodemon + tsx)
npm run build           # Production build (CJS + ESM)
npm run build:watch    # Incremental build
npm run lint           # Code quality check
```

### Running

**Development:**
```bash
npm run dev
```

**Production (CJS):**
```bash
npm start
# or: node dist/cjs/server.cjs
```

**Production (ESM):**
```bash
npm run start:esm
# or: node dist/esm/server.mjs
```

### Build Output
- `dist/cjs/` - CommonJS bundle (Node 18+)
- `dist/esm/` - ES Modules bundle
- Both include source maps
- Minified and compressed via esbuild

---

## 🔧 Extending the System

### Creating Custom Transformers

```typescript
import { Transformer, TransformerContext, TransformerOptions } from '@/types/transformer';
import { LLMProvider, UnifiedChatRequest } from '@/types/llm';

export class MyTransformer implements Transformer {
  name = 'MyTransformer';
  endPoint = '/v1/custom';  // Optional - omit if no endpoint needed
  
  constructor(private options?: TransformerOptions) {}
  
  async transformRequestOut(request: any): Promise<UnifiedChatRequest> {
    // Convert from provider format → unified format
    return {...};
  }
  
  async transformRequestIn(
    request: UnifiedChatRequest,
    provider: LLMProvider,
    context: TransformerContext
  ): Promise<any> {
    // Convert from unified format → provider format
    return {...};
  }
  
  async transformResponseOut(response: Response, context?: TransformerContext): Promise<Response> {
    // Convert provider response → normalized response
    return response;
  }
  
  async transformResponseIn(response: Response, context?: TransformerContext): Promise<Response> {
    // Convert normalized response → final response
    return response;
  }
  
  async auth?(request: any, provider: LLMProvider): Promise<any> {
    // Optional: handle authentication
    return request;
  }
}
```

### Registering Transformers

**Built-in:** Edit `src/transformer/index.ts` and import/export your transformer

**External:** Configure in `config.json`:
```json5
{
  transformers: [
    ["./path/to/MyTransformer.ts", {option: "value"}]
  ]
}
```

---

## 💾 Supported Features

### Message Features
- ✅ Multi-turn conversations
- ✅ System prompts with cache control
- ✅ Tool/function calling definitions
- ✅ Tool results in conversation
- ✅ Image inputs (base64 and URL)
- ✅ Extended thinking/reasoning blocks
- ✅ Text content with cache control

### Model Parameters
- ✅ Temperature (sampling control)
- ✅ Max tokens / max completion tokens
- ✅ Tool choice selection
- ✅ Streaming responses (SSE format)
- ✅ Extended reasoning (effort levels)
- ✅ Custom parameters per model
- ✅ Sampling configuration

### Provider Features
- ✅ Dynamic provider registration
- ✅ Per-model transformer chains
- ✅ Provider-level transformer chains
- ✅ API key authentication (Bearer or custom headers)
- ✅ HTTPS proxy support
- ✅ Request timeout configuration
- ✅ Provider enable/disable toggle

### Streaming Support
- ✅ SSE (Server-Sent Events) format
- ✅ Delta content delivery
- ✅ Tool call streaming
- ✅ Thinking/reasoning content streaming
- ✅ Proper stream termination

---

## 📦 Dependencies

### Core
- `fastify` (^5.4.0) - HTTP server framework
- `@fastify/cors` (^11.0.1) - CORS support

### LLM SDKs
- `openai` (^5.6.0) - OpenAI API client
- `@anthropic-ai/sdk` (^0.54.0) - Anthropic API client
- `@google/genai` (^1.7.0) - Google GenAI client
- `google-auth-library` (^10.1.0) - Google authentication

### Utilities
- `json5` (^2.2.3) - JSON5 parsing
- `jsonrepair` (^3.13.0) - Malformed JSON repair
- `dotenv` (^16.5.0) - Environment variable loading
- `uuid` (^11.1.0) - UUID generation
- `undici` (^7.10.0) - HTTP client with proxy support

### Development
- `typescript` (^5.8.3) - TypeScript compiler
- `tsx` (^4.20.3) - TypeScript executor
- `nodemon` (^3.1.10) - File watcher for development
- `esbuild` (^0.25.5) - Build tool
- `eslint` - Code linting

---

## 🔐 Security Considerations

- ✅ API keys stored in configuration (external management recommended)
- ✅ HTTPS proxy support for compliance
- ✅ Bearer or custom authentication header support
- ✅ Request validation and error handling
- ✅ CORS enabled (configurable)
- ✅ Request timeout enforcement (default 60 minutes)
- ✅ Input sanitization for JSON responses

**Recommendations:**
- Use environment variables for sensitive data (`.env` file)
- Deploy with HTTPS
- Implement rate limiting at reverse proxy level
- Monitor request logs for anomalies
- Validate API keys at provider level

---

## 🐛 Error Handling

### Error Response Format
```json
{
  "statusCode": 400,
  "error": "Invalid Request",
  "message": "Missing model in request body",
  "code": "invalid_request"
}
```

### Common Error Codes
- `provider_not_found` - Provider doesn't exist
- `provider_response_error` - Provider API error
- `invalid_request` - Malformed request
- `provider_exists` - Provider already registered

### Logging
- Global request/response logging
- Provider error logging with details
- Transformer chain execution logging
- Available via Fastify logger (Pino-based)

---

## 📊 Use Cases

1. **Multi-Provider Abstraction**
   - Use same API for different LLM providers
   - Easy provider switching without code changes

2. **Model Router**
   - Route requests to different models based on criteria
   - Load balancing across providers

3. **Feature Normalization**
   - Use extended features (reasoning, tool use) across different providers
   - Transparent capability adaptation

4. **API Gateway**
   - Single entry point for LLM access
   - Centralized authentication and monitoring

5. **Cost Optimization**
   - Route to most cost-effective provider per request
   - Monitor usage per model/provider

6. **Development Flexibility**
   - Switch between providers in development vs production
   - Test compatibility without changing client code

---

## 📝 Recent Features (v1.0.51)

- Extended reasoning and thinking block support
- Multiple provider transformer chains
- Model-specific transformer configuration
- Stream options normalization
- Cache control management
- Enhanced tool use handling
- Improved error reporting

---

## 🤝 Contributing

This project is actively maintained. See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

---

## 📚 Additional Resources

- [CLAUDE.md](./CLAUDE.md) - Development guidelines for AI-assisted coding
- [AGENTS.md](./AGENTS.md) - Agent/multi-AI coordination patterns
- [GEMINI.md](./GEMINI.md) - Gemini-specific implementation details
- [README.md](./README.md) - Quick start guide
