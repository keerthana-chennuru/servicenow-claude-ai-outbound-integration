# 🤖 ServiceNow + Claude AI — Outbound REST Integration
## Project 1: Claude AI Chat Assistant in ServiceNow

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&pause=1000&color=6A9FBF&width=600&lines=ServiceNow+%2B+Claude+AI+Integration;Outbound+REST+%7C+Script+Include;Auto+Summarize+%7C+RCA+%7C+KB+Article+Generator" alt="Typing SVG" />

![Platform](https://img.shields.io/badge/Platform-ServiceNow-green?style=for-the-badge&logo=servicenow)
![Scope](https://img.shields.io/badge/Scope-Global-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-AI%20Integration-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=for-the-badge)

---

## 📌 Overview

Built a full **ServiceNow ↔ Claude AI integration** using Outbound REST Messages and a reusable Script Include. The integration automatically summarizes incidents, suggests resolutions, performs root cause analysis, and generates Knowledge Base article drafts — all triggered by Business Rules on the Incident table. A custom **Claude AI Chat** module also allows free-form questions answered directly by Claude.

---

## 🎯 Objectives

- ✅ Securely store the Anthropic API key using a Private System Property
- ✅ Configure an Outbound REST Message to call `api.anthropic.com/v1/messages`
- ✅ Build a reusable `ClaudeAI` Script Include with all API logic centralized
- ✅ Create Business Rule that auto-trigger AI responses on Incident events
- ✅ Create a custom Claude AI Chat table for free-form AI queries
- ✅ Validate the integration end-to-end with live incident records

---

## 🏗️ Integration Architecture

| Component | What It Is | Role |
|---|---|---|
| API Key | Anthropic authentication key | Authenticates ServiceNow to Claude API |
| System Property | Secure key storage in ServiceNow | Keeps API key safe, never hardcoded |
| Outbound REST Message | HTTP connection configuration | Defines endpoint + headers for Claude API |
| REST Message Function | POST method + request body | Sends the actual message payload |
| Script Include (`ClaudeAI`) | Reusable JavaScript class | Brain — all Claude API logic lives here |
| Business Rule   | Fires on Incident events | Auto-summarizes, suggest resolution,RCA, and generate KB articles |


---

## 🔄 Integration Flow

```
Incident Event (Create / Update / Resolve)
        ↓
Business Rule fires (Before)
        ↓
Calls ClaudeAI Script Include method
        ↓
Script Include → Outbound REST Message
        ↓
https://api.anthropic.com/v1/messages
        ↓
Claude AI processes & responds
        ↓
Response written to Work Notes / KB Article ✅
```

---

## 📋 Step-by-Step Implementation

### Step 1 — Get Your Claude API Key

1. Go to [https://platform.claude.com](https://platform.claude.com)
2. Sign up or log in
3. Navigate to **API Keys** in the left sidebar
4. Click **Create Key** — name it: `ServiceNow-Integration`
5. Copy the key immediately *(shown ONLY ONCE)* — starts with: `sk-ant-api03-...`

> ⚠️ **WARNING:** Never share your API key publicly or hardcode it in scripts.

---

### Step 2 — Store API Key in ServiceNow (System Property)

**Path:** System Definition > System Properties → New
*(or navigate to: `sys_properties_list.do`)*

| Field | Value |
|---|---|
| Name | `anthropic.api.key` |
| Value | `sk-ant-api03-xxxxxxxxxxxx` *(your actual key)* |
| Description | Claude AI API Key for ServiceNow Integration |
| Type | string |
| Private | ✅ CHECK THIS — hides value from plain-text view |

> 💡 Marking the property as **Private** means even admins cannot read the value in plain text from the UI. This is best practice for storing secrets.

---

### Step 3 — Create Outbound REST Message

**Path:** System Web Services > Outbound > REST Messages → New

| Field | Value |
|---|---|
| Name | `Claude AI API` |
| Endpoint | `https://api.anthropic.com/v1/messages` |
| Authentication type | No authentication *(API key passed manually in headers)* |

**HTTP Request Headers:**

| Header Name | Header Value |
|---|---|
| `Content-Type` | `application/json` |
| `anthropic-version` | `2023-06-01` |
| `x-api-key` | `${anthropic_api_key}` |

> 💡 `${anthropic_api_key}` is a variable placeholder — filled at runtime by the Script Include.

---

### Step 4 — Create REST Message Function (sendMessage)

**Path:** Inside Claude AI API REST Message → HTTP Methods tab → New

| Field | Value |
|---|---|
| Name | `sendMessage` |
| HTTP Method | POST |
| Endpoint | *(leave blank — inherits from parent)* |

**Content (request body):**

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "${user_message}"
    }
  ]
}
```

> 💡 Click **Auto-generate variables** — ServiceNow will detect `${anthropic_api_key}` and `${user_message}` automatically. Save.

---

### Step 5 — Create Script Include (ClaudeAI)

**Path:** System Definition > Script Includes → New

| Field | Value |
|---|---|
| Name | `ClaudeAI` |
| API Name | `ClaudeAI` |
| Active | ✅ Checked |
| Client callable | ❌ Unchecked |
| Description | Script Include to call Claude AI API from ServiceNow |

```javascript
var ClaudeAI = Class.create();
ClaudeAI.prototype = {
  initialize: function() {
    this.apiKey = gs.getProperty('anthropic.api.key');
  },
  sendMessage: function(userMessage) {
    try {
      var rm = new sn_ws.RESTMessageV2();
      rm.setEndpoint('https://api.anthropic.com/v1/messages');
      rm.setHttpMethod('POST');
      rm.setRequestHeader('Content-Type', 'application/json');
      rm.setRequestHeader('anthropic-version', '2023-06-01');
      rm.setRequestHeader('x-api-key', this.apiKey);
      rm.setHttpTimeout(30000);
      var body = JSON.stringify({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 1024,
        messages: [{ role: 'user', content: userMessage }]
      });
      rm.setRequestBody(body);
      var response     = rm.execute();
      var statusCode   = response.getStatusCode();
      var responseBody = response.getBody();
      if (statusCode == 200) {
        var parsed = JSON.parse(responseBody);
        return { success: true, message: parsed.content[0].text,
                 tokens: parsed.usage.output_tokens };
      } else {
        gs.error('ClaudeAI Error ' + statusCode + ': ' + responseBody);
        return { success: false, error: 'HTTP ' + statusCode };
      }
    } catch(ex) {
      gs.error('ClaudeAI Exception: ' + ex.message);
      return { success: false, error: ex.message };
    }
  },
  summarizeIncident:  function(shortDesc, description) { ... },
  suggestResolution:  function(shortDesc, description) { ... },
  criticalRCA:        function(shortDesc, description) { ... },
  generateKBArticle:  function(shortDesc, description, resolutionNotes) { ... },
  type: 'ClaudeAI'
};
```

> 💡 **KEY POINT:** This Script Include is written **once**.Business Rule simply call `new ClaudeAI()` and invoke the relevant method — no duplicated API logic.

---

### Step 6 — Create Custom Table (Claude AI Chat)

**Path:** System Definition > Tables → New

| Field | Value |
|---|---|
| Label | `Claude AI Chat` |
| Name | `u_claude_ai_chat` |
| Add module to menu | ✅ Checked |

**Table Columns:**

| Column Label | Column Name | Type |
|---|---|---|
| Question | `u_question` | String (1000) |
| Answer | `u_answer` | String (4000) |
| Status | `u_status` | String (100) |


---

### Step 7 — Business Rule Script (Claude AI Chat)

```javascript
(function executeRule(current, previous) {
  var question = current.u_question.toString();
  var claude   = new ClaudeAI();

  // Intent: Summarize
  if (question.toLowerCase().indexOf('summarize') != -1) {
    var m = question.match(/INC\d+/i);
    if (m) {
      var inc = new GlideRecord('incident');
      if (inc.get('number', m[0])) {
        var r = claude.summarizeIncident(inc.short_description+'', inc.description+'');
        current.u_answer = r.success ? r.message : r.error;
        current.u_status = r.success ? 'Success' : 'Failed'; return;
      }
    }
  }
  // Intent: Resolve / Fix
  if (question.toLowerCase().indexOf('resolve') != -1 ||
      question.toLowerCase().indexOf('fix') != -1) {
    // ... same pattern, calls suggestResolution()
  }
  // Intent: Root Cause Analysis
  if (question.toLowerCase().indexOf('rca') != -1 ||
      question.toLowerCase().indexOf('root cause') != -1) {
    // ... same pattern, calls criticalRCA()
  }
  // Intent: Knowledge Base article
  if (question.toLowerCase().indexOf('kb') != -1 ||
      question.toLowerCase().indexOf('knowledge') != -1) {
    // ... same pattern, calls generateKBArticle()
  }
  // Fallback: send question directly to Claude
  var gen = claude.sendMessage(question);
  current.u_answer = gen.success ? gen.message : gen.error;
  current.u_status  = gen.success ? 'Success' : 'Failed';
})(current, previous);
```

---

### Step 8 — Testing the Integration

Navigate to the **Claude AI Chat** module in the left navigation menu.

| # | Action |
|---|---|
| 1 | Open a New Record — Click **New** in the Claude AI Chat module |
| 2 | Type a question into the **Question** field |
| 3 | Click **Save** — the Answer field auto-fills with Claude's response |

**Sample Questions:**

| Sample Question | What It Does |
|---|---|
| `summarize this inc INC0000043` | Returns a 2-line AI summary of the incident |
| `how to fix INC0000043` | Returns 3 suggested resolution steps |
| `root cause for INC0000043` | Returns RCA, immediate actions, and escalation advice |
| `generate kb for INC0000043` | Creates a formatted Knowledge Base article draft |
| `what is SAP?` | General question — answered directly by Claude |

**What to Check After Testing:**
- Background script output shows **SUCCESS**
- System Logs > Application — no errors logged
- Transaction Logs — shows outbound call to `api.anthropic.com`
- Create a test incident — work notes should auto-populate with AI summary

---

## ⚙️ Technical Details

| Component | Detail |
|---|---|
| Model | `claude-sonnet-4-20250514` |
| API Endpoint | `https://api.anthropic.com/v1/messages` |
| Anthropic API Version | `2023-06-01` |
| Max Tokens | 1024 |
| Timeout | 30,000 ms |
| Key Storage | Private System Property (`anthropic.api.key`) |
| Script Include | `ClaudeAI` — Global scope, server-side only |

---

## ✅ Skills Demonstrated

![Outbound REST](https://img.shields.io/badge/Outbound%20REST-API%20Integration-green?style=flat-square)
![Script Include](https://img.shields.io/badge/Script%20Include-ClaudeAI%20Class-blue?style=flat-square)
![Business Rules](https://img.shields.io/badge/Business%20Rule-%20Auto--Triggers-orange?style=flat-square)
![Security](https://img.shields.io/badge/Security-Private%20Sys%20Property-red?style=flat-square)

- 🔧 Outbound REST Message configuration with dynamic headers
- 🧠 Reusable Script Include design (`ClaudeAI` class)
- 📥 Input/Output variable mapping via REST Message Functions
- ⚡ Business Rule triggers on Insert, Update, and State changes
- 🔐 Secure API key storage using Private System Properties
- 📝 Work Notes and KB Article auto-population
- 🗂️ Custom table creation (`u_claude_ai_chat`)
- 🧪 End-to-end integration testing with live records

---

## ⚠️ Common Mistakes to Avoid

| ❌ Mistake | ✅ Best Practice |
|---|---|
| Hardcoding the API key in scripts | Always store in a Private System Property |
| Missing `setWorkflow(false)` in Business Rules | Prevents infinite loop when updating `work_notes` |
| Wrong JSON format in REST Message body | Validate the request body matches the Anthropic API spec |
| Not handling null/failed API responses | Always check `r.success` before using `r.message` |
| Duplicate API logic in each Business Rule | Centralize all logic in the `ClaudeAI` Script Include |

---

## 🔧 Troubleshooting

| Error / Symptom | Likely Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong or expired API key | Re-check `anthropic.api.key` system property |
| `HTTP 400 Bad Request` | Wrong request body format | Check JSON in REST Message Function content |
| No work notes added | BR not firing or condition not met | Check BR conditions and System Logs |
| Infinite loop / stack overflow | Missing `setWorkflow(false)` | Add `current.setWorkflow(false)` before `update()` |
| null or empty response | Claude returned empty content | Add logging: `gs.info(JSON.stringify(result))` |
| Connection timeout | Firewall blocking Claude API | Check ServiceNow outbound firewall rules |
| BR fires but no KB created | Wrong KB `sys_id` | Replace placeholder with actual Knowledge Base `sys_id` |

---

## 💡 Key Learning

> The `ClaudeAI` Script Include acts as a **single source of truth** for all AI logic. Business Rules stay clean and minimal — they simply instantiate `new ClaudeAI()` and call the right method. This pattern makes the integration easy to maintain, extend, and debug.

---

<div align="center">

**Made with ❤️ by Keerthana Chennuru**

![ServiceNow](https://img.shields.io/badge/Built%20on-ServiceNow-green?style=for-the-badge)

