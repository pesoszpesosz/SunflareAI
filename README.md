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
- `super_agent` - Super agent with all tools (default)
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

def send_message(message, agent="super_agent", subscription_key="YOUR_KEY"):
    """
    Send a message to the AI and get a complete response
    Automatically handles both short and long requests
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
        # Check if response is chunked (for long requests)
        if response.headers.get('transfer-encoding') == 'chunked':
            # Handle streaming response - extract final result
            for line in response.iter_lines(decode_unicode=True):
                if line:
                    try:
                        data = json.loads(line)
                        # Return the final response when we get it
                        if data.get("response"):
                            return data
                    except json.JSONDecodeError:
                        continue
        else:
            # Handle normal response (for short requests)
            return response.json()
    else:
        raise Exception(f"Request failed: {response.status_code} - {response.text}")

# Example usage
try:
    result = send_message("Hello! How are you?")
    print(f"AI Response: {result['response']}")
    print(f"Agent used: {result['agentUsed']}")
    print(f"Response time: {result['responseTime']}s")

except Exception as e:
    print(f"Error: {e}")
```

---

## 🟢 Node.js Integration

```javascript
const https = require('https');

function sendMessage(message, agent = 'super_agent', subscriptionKey = 'YOUR_KEY') {
    return new Promise((resolve, reject) => {
        const postData = JSON.stringify({
            message: message,
            agent: agent
        });

        const options = {
            hostname: 'api.sunflareai.com',
            port: 443,
            path: '/api/v2/message',
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Content-Length': Buffer.byteLength(postData),
                'subscription-key': subscriptionKey
            }
        };

        const req = https.request(options, (res) => {
            let data = '';
            let finalResponse = null;

            // Check if response is chunked (for long requests)
            const isStreaming = res.headers['transfer-encoding'] === 'chunked';

            res.on('data', (chunk) => {
                const chunkStr = chunk.toString();
                data += chunkStr;

                if (isStreaming) {
                    // Handle streaming chunks - look for final response
                    const lines = chunkStr.split('\n').filter(line => line.trim());
                    for (const line of lines) {
                        try {
                            const parsed = JSON.parse(line);
                            if (parsed.response) {
                                finalResponse = parsed;
                            }
                        } catch (parseError) {
                            // Ignore non-JSON lines
                        }
                    }
                }
            });

            res.on('end', () => {
                if (isStreaming && finalResponse) {
                    resolve(finalResponse);
                } else {
                    // Handle normal response (for short requests)
                    try {
                        const result = JSON.parse(data);
                        resolve(result);
                    } catch (error) {
                        reject(new Error(`JSON parse error: ${error.message}`));
                    }
                }
            });
        });

        req.on('error', reject);
        req.on('timeout', () => {
            req.destroy();
            reject(new Error('Request timeout'));
        });

        req.write(postData);
        req.end();
    });
}

// Example usage
async function example() {
    try {
        const result = await sendMessage("Hello! How are you?");
        console.log(`AI Response: ${result.response}`);
        console.log(`Agent used: ${result.agentUsed}`);
        console.log(`Response time: ${result.responseTime}s`);

    } catch (error) {
        console.error(`Error: ${error.message}`);
    }
}

example();
```

---

## 💻 cURL Integration

```bash
# Basic request
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
