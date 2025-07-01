# 🚀 AI API - Integration Guide

## Overview

Access multiple AI models through a simple API. The system automatically handles both short and long requests with different response patterns.

## 📋 Quick Start

### Base URL
```
https://api.sunflareai.com/api/v2/message
```

 
All requests require a subscription key in the header:
```
subscription-key: YOUR_SUBSCRIPTION_KEY
```

### Supported AI Models
- `super_agent` - GenSpark super agent with all tools (default)
- `claude_opus` - Claude Opus 4 for complex reasoning
- `claude_sonnet` - Claude Sonnet 4 for general use
- `gpt4` - GPT-4.1 latest model
- `o3_pro` - O3 Pro for advanced reasoning
- `gemini` - Gemini 2.5 Pro multimodal model

## 🔄 Response Format

All requests return a complete JSON response with the AI's answer. The code automatically detects and handles both short responses (normal JSON) and long responses (streaming) - you always get the same simple response format.

---

## 🐍 Python Integration

```python
import requests
import json

def send_message(message, agent="super_agent", subscription_key="YOUR_SUBSCRIPTION_KEY_HERE"):
    """
    Send a message to the SunFlare AI API
    Automatically handles both short and long requests with proper SSE parsing

    Args:
        message (str): The message to send to the AI
        agent (str): The AI agent to use (default: 'super_agent')
        subscription_key (str): Your SunFlare AI subscription key

    Returns:
        dict: AI response with response, agentUsed, responseTime, etc.
    """

    url = "https://api.sunflareai.com/api/v2/message"
    headers = {
        "Content-Type": "application/json",
        "subscription-key": subscription_key
    }
    data = {
        "message": message,
        "agent": agent
    }

    response = requests.post(url, headers=headers, json=data, stream=True)

    if response.status_code == 200:
        # Check if response is chunked/streaming (for long requests)
        if response.headers.get('transfer-encoding') == 'chunked':
            print("🔄 Processing streaming response...")

            # Handle Server-Sent Events (SSE) streaming response
            sse_buffer = ""
            final_response = None
            all_content = ""

            for chunk in response.iter_content(chunk_size=1024, decode_unicode=True):
                if chunk:
                    all_content += chunk
                    # Add chunk to SSE buffer
                    sse_buffer += chunk

                    # Process complete SSE lines
                    lines = sse_buffer.split('\n')

                    # Keep the last incomplete line in buffer
                    sse_buffer = lines.pop() if lines else ""

                    for line in lines:
                        line = line.strip()
                        if line.startswith('data: '):
                            try:
                                # Strip "data: " prefix to get JSON payload
                                json_str = line[6:]  # Remove 'data: '
                                parsed = json.loads(json_str)

                                # Look for the final response in 'stream_complete' event
                                if (parsed.get('type') == 'stream_complete' and
                                    parsed.get('data') and
                                    parsed.get('data', {}).get('response')):
                                    final_response = parsed['data']
                                    break
                            except json.JSONDecodeError:
                                # Ignore malformed SSE data
                                continue

                    if final_response:
                        break

            if final_response:
                return final_response
            else:
                # If no SSE events found, try parsing as regular JSON
                try:
                    return json.loads(all_content)
                except json.JSONDecodeError:
                    raise Exception("No valid response found in streaming data")
        else:
            print("📄 Processing standard response...")
            # Handle normal response (for short requests)
            return response.json()
    else:
        raise Exception(f"Request failed: {response.status_code} - {response.text}")

# Example usage
def example():
    try:
        print("🟢 Testing SunFlare AI API Integration")
        print("=" * 50)

        # Example request - the API will automatically choose the best response format
        result = send_message("What are the latest developments in artificial intelligence?")

        # Handle both streaming and regular response formats
        response_text = result.get('response', 'No response received')
        agent_used = result.get('agentUsed', 'AI Assistant')
        response_time = f"{result.get('responseTime', 'N/A')}s" if result.get('responseTime') else 'N/A'
        request_id = result.get('requestId') or result.get('timestamp') or 'N/A'
        success = bool(result.get('response'))

        print("🎉 SUCCESS! Response received:")
        print("=" * 50)
        print(f"🤖 AI Response: {response_text[:500]}{'...' if len(response_text) > 500 else ''}")
        print(f"🔧 Agent used: {agent_used}")
        print(f"⏱️ Response time: {response_time}")
        print(f"🆔 Request ID: {request_id}")
        print(f"✅ Success: {success}")

    except Exception as e:
        print(f"❌ Error: {e}")

if __name__ == "__main__":
    example()
```

---

## 🟢 Node.js Integration

```javascript
const https = require('https');

/**
 * Send a message to the SunFlare AI API
 * @param {string} message - The message to send to the AI
 * @param {string} agent - The AI agent to use (default: 'super_agent')
 * @param {string} subscriptionKey - Your SunFlare AI subscription key
 * @returns {Promise<Object>} - Promise resolving to the AI response
 */
function sendMessage(message, agent = 'super_agent', subscriptionKey = 'YOUR_SUBSCRIPTION_KEY_HERE') {
    return new Promise((resolve, reject) => {
        const postData = JSON.stringify({
            message: message,
            agent: agent
        });

        // API endpoint configuration
        const options = {
            hostname: 'api.sunflareai.com',    // SunFlare AI API hostname
            port: 443,                         // HTTPS port
            path: '/api/v2/message',           // Message endpoint
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Content-Length': Buffer.byteLength(postData),
                'subscription-key': subscriptionKey  // Your API subscription key
            }
        };

        const req = https.request(options, (res) => {
            let data = '';
            let finalResponse = null;
            let sseBuffer = ''; // Buffer for incomplete SSE lines

            // Detect if this is a streaming response (for long/complex requests)
            // The API automatically uses streaming for requests that take longer to process
            const isStreaming = res.headers['transfer-encoding'] === 'chunked';

            res.on('data', (chunk) => {
                const chunkStr = chunk.toString();
                data += chunkStr;

                if (isStreaming) {
                    // Add chunk to SSE buffer and process complete lines
                    sseBuffer += chunkStr;

                    // Process complete SSE lines (ending with \n\n or \n)
                    const lines = sseBuffer.split('\n');

                    // Keep the last incomplete line in buffer
                    sseBuffer = lines.pop() || '';

                    for (const line of lines) {
                        const trimmedLine = line.trim();
                        if (trimmedLine.startsWith('data: ')) {
                            try {
                                // Strip "data: " prefix to get the JSON payload
                                const jsonStr = trimmedLine.substring(6);
                                const parsed = JSON.parse(jsonStr);

                                // Look for the final response in the 'stream_complete' event
                                if (parsed.type === 'stream_complete' && parsed.data && parsed.data.response) {
                                    finalResponse = parsed.data;
                                }
                            } catch (parseError) {
                                // Ignore malformed SSE data - this is normal for mixed streaming formats
                            }
                        }
                    }
                }
            });

            res.on('end', () => {
                if (isStreaming && finalResponse) {
                    // For streaming responses, use the extracted final response
                    resolve(finalResponse);
                } else {
                    // For normal responses, parse the complete JSON response
                    try {
                        const result = JSON.parse(data);
                        resolve(result);
                    } catch (error) {
                        reject(new Error(`Failed to parse response: ${error.message}`));
                    }
                }
            });
        });

        // Handle request errors
        req.on('error', (error) => {
            reject(error);
        });

        // Optional: Set a timeout if needed (uncomment the lines below)
        // req.setTimeout(300000, () => {  // 5 minutes
        //     req.destroy();
        //     reject(new Error('Request timeout'));
        // });

        req.write(postData);
        req.end();
    });
}

// Example usage
async function example() {
    try {
        // Example request - the API will automatically choose the best response format
        const result = await sendMessage("What are the latest developments in artificial intelligence?");

        // Handle both streaming and regular response formats
        const response = result.response || 'No response received';
        const agentUsed = result.agentUsed || 'AI Assistant';
        const responseTime = result.responseTime ? `${result.responseTime}s` : 'N/A';
        const requestId = result.requestId || result.timestamp || Date.now();
        const success = result.response ? true : false;

        console.log('🎉 SUCCESS! Response received:');
        console.log('🤖 AI Response:', response.substring(0, 500) + (response.length > 500 ? '...' : ''));
        console.log('🔧 Agent used:', agentUsed);
        console.log('⏱️ Response time:', responseTime);
        console.log('🆔 Request ID:', requestId);
        console.log('✅ Success:', success);

    } catch (error) {
        console.error('❌ Error:', error.message);
    }
}

example();
```

---

## 💻 cURL Integration

```bash
# Basic request (short responses - returns JSON)
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Hello! How are you?",
    "agent": "super_agent"
  }'

# Pretty printed response
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Hello! How are you?",
    "agent": "super_agent"
  }' | jq '.'

# Long/complex request (automatically uses streaming)
# Shows real-time Server-Sent Events (SSE) as the AI works
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Research the latest AI developments and compare API pricing",
    "agent": "super_agent"
  }'

# Extract final response from streaming (using grep and tail)
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Research the latest AI developments",
    "agent": "super_agent"
  }' | grep '"type":"stream_complete"' | tail -1
```

### Understanding cURL Streaming Responses

When you send a complex request, the API automatically switches to **Server-Sent Events (SSE)** format. You'll see output like:

```
data: {"type":"stream_start","data":{"message":"Request received, processing...","timestamp":1751376835475}}

data: {"type":"stream_chunk","data":{"content":"I'll help you research...","timestamp":1751376842692}}

data: {"type":"stream_chunk","data":{"content":"More detailed information...","timestamp":1751376849055}}

data: {"type":"stream_complete","data":{"response":"Complete AI response here","responseTime":106.548,"agentUsed":"super_agent","timestamp":1751376942022}}
```

**Event Types:**
- `stream_start` - Request acknowledged and processing begins
- `stream_chunk` - Progressive content as the AI works
- `stream_complete` - Final response with complete data and metadata

```bash
# Different AI Models
# Claude Opus for complex reasoning
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Analyze the philosophical implications of artificial intelligence.",
    "agent": "claude_opus"
  }'

# GPT-4 for general tasks
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Write a Python script for data analysis.",
    "agent": "gpt4"
  }'

# Gemini for multimodal tasks
curl -X POST https://api.sunflareai.com/api/v2/message \
  -H "Content-Type: application/json" \
  -H "subscription-key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "message": "Explain computer vision concepts with examples.",
    "agent": "gemini"
  }'
```

---

## 📊 Response Format

### Standard Response
```json
{
  "success": true,
  "response": "AI generated response text here...",
  "responseTime": 5.2,
  "requestId": "uuid-here",
  "agentUsed": "super_agent"
}
```

---

## ⚡ Features

### Rate Limiting
- **10 requests per minute**
- **Headers included**: `ratelimit-limit`, `ratelimit-remaining`, `ratelimit-reset`

### Error Handling
```json
// Rate limit exceeded
{
  "error": "Too many requests",
  "message": "Rate limit exceeded. Please try again later.",
  "retryAfter": 60
}

// Invalid model
{
  "error": "Invalid agent specified",
  "availableAgents": ["super_agent", "claude_opus", "claude_sonnet", "gpt4", "o3_pro", "gemini"]
}

// Authentication error
{
  "error": "Invalid subscription key",
  "statusCode": 401
}
```

---

## 🔧 Best Practices

### 1. Monitor Rate Limits
Check rate limit headers:
```bash
# cURL example - check headers (use -I for headers only)
curl -I https://api.sunflareai.com/api/v2/message \
  -H "subscription-key: YOUR_KEY"
```

### 2. Choose Appropriate Models
- **super_agent**: Complex tasks with tools (default, recommended)
- **claude_opus**: Deep reasoning and analysis
- **claude_sonnet**: General purpose, balanced
- **gpt4**: Code generation and technical tasks
- **o3_pro**: Advanced problem solving
- **gemini**: Multimodal and creative tasks

### Support
For technical support and API questions, contact the development team.
