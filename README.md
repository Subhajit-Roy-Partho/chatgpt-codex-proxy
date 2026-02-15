# Codex OpenAI Proxy

A proxy server that allows CLINE (Claude Code) and other OpenAI-compatible extensions to use ChatGPT Plus tokens from Codex authentication instead of requiring separate OpenAI API keys.

## Overview

This proxy bridges the gap between:
- **Input**: Standard OpenAI Chat Completions API format (what CLINE expects)
- **Output**: ChatGPT Responses API format (what ChatGPT backend uses)

## Features

- ✅ **OpenAI API Compatibility**: Accepts standard OpenAI Chat Completions requests
- ✅ **ChatGPT Plus Integration**: Uses your existing ChatGPT Plus tokens  
- ✅ **Cloudflare Bypass**: Handles ChatGPT's Cloudflare protection with browser-like headers
- ✅ **HTTPS Support**: Works with extensions requiring secure connections (via ngrok)
- ✅ **Streaming Responses**: Full streaming support for real-time responses
- ✅ **CLINE Compatible**: Tested extensively with CLINE VS Code extension
- ✅ **Array Content Support**: Handles both string and array message formats from OpenAI SDK
- ✅ **Universal Routing**: Bulletproof request routing that bypasses complex warp conflicts

## Quick Start

### 1. Build and Run

```bash
git clone https://github.com/Securiteru/codex-openai-proxy.git
cd codex-openai-proxy
cargo build --release
./target/release/codex-openai-proxy --port 8888 --auth-path ~/.codex/auth.json
```

### 2. Setup HTTPS Tunnel (Required for CLINE)

Most VS Code extensions require HTTPS:

```bash
# Install ngrok and create your own static domain at https://dashboard.ngrok.com/domains
# Replace 'your-static-domain' with your unique domain name
ngrok http 8888 --domain=your-static-domain.ngrok-free.app
```

**Security Note**: Always use your own unique ngrok domain. Do not share your domain publicly to prevent unauthorized access to your proxy.

### 3. Configure CLINE Extension

In VS Code CLINE settings:
- **Base URL**: `https://your-static-domain.ngrok-free.app`
- **Model**: Any base model from your proxy allowlist (default includes `gpt-5`, `gpt-5.2`, `gpt-5.3-codex`, `gpt-5.2-codex`, `gpt-5.1-codex-max`, `gpt-5.1-codex-mini`)
- **Reasoning control**: append suffixes like `-low`, `-medium`, `-high`, `-xhigh` (example: `gpt-5.2-xhigh`)
- **API Key**: Any value (not used, but required by extension)

### 4. Test Connection

```bash
# Health check
curl https://your-static-domain.ngrok-free.app/health

# Test completion
curl -X POST https://your-static-domain.ngrok-free.app/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test-key" \
  -d '{
    "model": "gpt-5",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## How It Works

### Request Flow

1. **CLINE** → Chat Completions format → **Proxy**
2. **Proxy** → Converts to Responses API → **ChatGPT Backend**
3. **ChatGPT Backend** → Responses API format → **Proxy**
4. **Proxy** → Converts to Chat Completions → **CLINE**

### Format Conversion

**Chat Completions Request:**
```json
{
  "model": "gpt-5",
  "messages": [
    {"role": "user", "content": "Hello!"}
  ]
}
```

**Responses API Request:**
```json
{
  "model": "gpt-5", 
  "instructions": "You are a helpful AI assistant.",
  "input": [
    {
      "type": "message",
      "role": "user", 
      "content": [{"type": "input_text", "text": "Hello!"}]
    }
  ],
  "tools": [],
  "tool_choice": "auto",
  "store": false,
  "stream": false
}
```

## Configuration

### Command Line Options

```bash
codex-openai-proxy [OPTIONS]

Options:
  -p, --port <PORT>          Port to listen on [default: 8080]
      --auth-path <PATH>     Path to Codex auth.json [default: ~/.codex/auth.json]
  -h, --help                 Print help
  -v, --version              Print version
```

### Allowed Models

The proxy enforces an allowlist for `model` values:

- Default base-model allowlist: `gpt-5,gpt-5.2,gpt-5.3-codex,gpt-5.2-codex,gpt-5.1-codex-max,gpt-5.1-codex-mini`
- Override with `ALLOWED_MODELS` (comma-separated list)
- These defaults were validated against the ChatGPT Codex backend for this setup.

### Model Naming And Meaning

This proxy supports reasoning control by model naming convention:

| Requested model pattern | Meaning in proxy/backend payload |
|---|---|
| `<base-model>` | Uses base model with no explicit reasoning override (`reasoning` omitted) |
| `<base-model>-low` | Sets `reasoning.effort` to `low` |
| `<base-model>-medium` | Sets `reasoning.effort` to `medium` |
| `<base-model>-high` | Sets `reasoning.effort` to `high` |
| `<base-model>-xhigh` | Sets `reasoning.effort` to `xhigh` |
| `<base-model>-extra-high` | Alias for `xhigh` |
| `<base-model>-extra_high` | Alias for `xhigh` |

### Current Default Available Models

The following request-model IDs are available by default:

```text
gpt-5
gpt-5-low
gpt-5-medium
gpt-5-high
gpt-5-xhigh
gpt-5.2
gpt-5.2-low
gpt-5.2-medium
gpt-5.2-high
gpt-5.2-xhigh
gpt-5.3-codex
gpt-5.3-codex-low
gpt-5.3-codex-medium
gpt-5.3-codex-high
gpt-5.3-codex-xhigh
gpt-5.2-codex
gpt-5.2-codex-low
gpt-5.2-codex-medium
gpt-5.2-codex-high
gpt-5.2-codex-xhigh
gpt-5.1-codex-max
gpt-5.1-codex-max-low
gpt-5.1-codex-max-medium
gpt-5.1-codex-max-high
gpt-5.1-codex-max-xhigh
gpt-5.1-codex-mini
gpt-5.1-codex-mini-low
gpt-5.1-codex-mini-medium
gpt-5.1-codex-mini-high
gpt-5.1-codex-mini-xhigh
```

Model routing examples:

- Request model `gpt-5.2-xhigh` -> backend model `gpt-5.2` with `reasoning.effort: xhigh`
- Request model `gpt-5.3-codex-high` -> backend model `gpt-5.3-codex` with `reasoning.effort: high`
- Request model `gpt-5.2-extra-high` -> backend model `gpt-5.2` with `reasoning.effort: xhigh`

```bash
ALLOWED_MODELS="gpt-5,gpt-5.2,gpt-5.3-codex,gpt-5.1-codex-max" cargo run -- --port 8080
```

`GET /models` and `GET /v1/models` return base models plus canonical suffix variants (`-low`, `-medium`, `-high`, `-xhigh`).

`-extra-high` and `-extra_high` aliases are accepted in requests but are not listed in `/models`.

Unknown base models or unsupported suffix combinations return `400` with `model_not_allowed`.

### Authentication

The proxy automatically reads authentication from your Codex `auth.json` file:

```json
{
  "access_token": "eyJ...",
  "account_id": "db1fc050-5df3-42c1-be65-9463d9d23f0b",
  "api_key": "sk-proj-..."
}
```

**Priority**: Uses `access_token` + `account_id` for ChatGPT Plus accounts, falls back to `api_key` for standard OpenAI accounts.

## API Endpoints

### Health Check
- **GET** `/health`
- Returns service status

### Models
- **GET** `/models` and `/v1/models`
- Returns the expanded request-model list derived from the base allowlist

### Chat Completions
- **POST** `/v1/chat/completions`
- OpenAI-compatible chat completions endpoint
- Supports: messages, model, temperature, max_tokens, stream, tools

## Troubleshooting

### Common Issues

**Connection Refused:**
```bash
# Check if proxy is running
curl http://localhost:8080/health
```

**Authentication Errors:**
```bash
# Verify auth.json exists and has valid tokens
cat ~/.codex/auth.json | jq .
```

**Backend Errors:**
```bash
# Check proxy logs for detailed error messages
RUST_LOG=debug cargo run
```

### Debug Mode

```bash
# Run with debug logging
RUST_LOG=debug cargo run -- --port 8080

# Test with verbose curl
curl -v -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5", "messages": [{"role": "user", "content": "Test"}]}'
```

## Development

### Building

```bash
cargo build
cargo test
cargo clippy
cargo fmt
```

### Adding Features

The proxy is designed to be extensible:

- **New endpoints**: Add routes in `main.rs`
- **Format conversion**: Modify conversion functions
- **Authentication**: Extend `AuthData` structure
- **Streaming**: Add SSE support for real-time responses

## License

This project is part of the Codex ecosystem and follows the same licensing as the main Codex repository.
