# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common commands
- Install dependencies: `npm install` (or `pnpm install`)
- Local development: `npm run dev` (nodemon + tsx hot reload `src/server.ts`)
- Build product: `npm run build` (call `scripts/build.ts`, output to `dist/cjs` and `dist/esm`)
- Incremental build: `npm run build:watch`
- Code inspection: `npm run lint`
- Run the compiled service: `npm start` (CJS) / `npm run start:esm`
- The test script has not been configured yet; if you need to test, please check mocha/chai/sinon in `devDependencies` and customize the command as needed

## Core Architecture Overview
- Fastify server (`src/server.ts`) is responsible for initializing logs, CORS, error handling, and mounting the API, service layer and transformer list at startup
- `ConfigService` reads JSON5 (default `config.json`), optional `.env`, and initial configuration, providing a unified configuration access entrance
- `TransformerService` automatically registers the built-in transformer (implementation under `src/transformer`), and supports dynamically loading additional transformers from the configuration; distinguishes transformers with/without endpoints to automatically mount routes
- `ProviderService` manages LLM providers: registration, update, deletion, model route resolution, and dynamic provider injection based on configuration files
- `LLMService` is only a thin encapsulation of ProviderService and provides external provider/model query capabilities.
- `src/api/routes.ts` registers health checks, provider CRUD, and all transformer endpoints; combines with `utils/request.ts` to send unified HTTP calls and adapt streaming responses
- Tool methods are concentrated in `src/utils/` (message content conversion, Gemini/Vertex feature processing, thinking chain tailoring, tool parameter analysis, etc.) for reuse by transformer in the request/response phase

## Execution process from request to response
1. The client request enters Fastify, and the model splitting logic in `preHandler` parses the `model` field into `provider`+`model`
2. `ProviderService.resolveModelRoute` confirms the target provider and model, and `TransformerService` finds the matching transformer
3. Execute sequentially in `handleTransformerEndpoint`: 
- `transformRequestOut` (unified request → provider format) 
- provider level transformer chain and model level transformer chain (`transformRequestIn`) 
- If bypass is configured, the above chain will be skipped and the original request/header information will be retained.
4. Use `sendUnifiedRequest` to call the target provider, automatically complete the authentication (Bearer API Key), support proxy (`ConfigService.getHttpsProxy`) and be compatible with streaming responses
5. The response phase executes the transformer chain in reverse order (`transformResponseOut` → `transformResponseIn`), and finally returns a unified JSON or SSE stream

## Configuration and expansion points
- By default, JSON5 configuration is read from `config.json` in the project root directory. You can override the path or disable JSON/ENV through the `ConfigService` construction parameter.
- The `providers` array in the configuration supports declaring custom providers (name, base URL, API Key, model, transformer chain), and automatically registers them at startup
- The `transformers` array in the configuration allows external transformer modules to be loaded on demand, by exporting the named class and providing `name` on the instance
- Built-in transformers are registered through the static property `TransformerName` or instance `name`; some transformers provide `endPoint` for automatically creating POST paths

## Build and deploy
- `scripts/build.ts` is packaged based on esbuild, generates compressed Node18 targets with sourcemap, and outputs CJS and ESM versions respectively.
- When running online, you can directly `node dist/cjs/server.cjs` or `node dist/esm/server.mjs`
- The `@` prefix is mapped to `src/` in `tsconfig.json`, please keep the reference path consistent

## Other tips
- Global request logs and error handling are located in `src/server.ts` and `src/api/middleware.ts`, streaming requests will append the `text/event-stream` header
- `node_modules` already includes dependencies such as Fastify, Anthropic SDK, OpenAI SDK, Google GenAI, jsonrepair, etc. Transformer can fully reuse these tools
- If you need to extend streaming or reasoning functions, you can refer to `StreamOptionsTransformer`, `ReasoningTransformer` and other implementation modes