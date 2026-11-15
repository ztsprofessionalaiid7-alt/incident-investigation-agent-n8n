# Autonomous Security Incident Investigation Agent (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `security-investigation-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "Autonomous security investigation agent (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create tables playbooks (with seed rows) and incident_reports in Supabase SQL Editor\n3. Slack channel: incident-war-room (change in the Slack node)\n4. Click Execute workflow: 3 mock incidents, the AI Agent investigates using playbooks and an IOC tool\nReal client use: replace Mock Incidents with a Webhook and playbooks with a vector store",
        "height": 260,
        "width": 580
      },
      "id": "2cbe4615-13dd-4ed0-849d-8ecd0cfd3256",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "5cfa80f2-cf31-4d74-94a8-8495454cb520",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { incident_id: 'INC-501', indicator: 'payroll-verify-login.example.com', report: 'An employee reported an email asking to verify payroll login. Three other staff clicked the link.' } },\n  { json: { incident_id: 'INC-502', indicator: 'encoded-ps-c2.example.net', report: 'EDR flagged encoded PowerShell running on file-server-02 at 3 AM.' } },\n  { json: { incident_id: 'INC-503', indicator: 'customer-exports', report: 'Cloud audit shows the storage bucket customer-exports was changed to public access.' } }\n];"
      },
      "id": "660d354e-5e5e-45f3-813e-9eacd870f9fc",
      "name": "Mock Incidents",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "operation": "select",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "playbooks",
          "mode": "name"
        },
        "options": {},
        "returnAll": true
      },
      "id": "25995754-3d42-49ee-9a14-45fa76badd81",
      "name": "Get Playbooks",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        440,
        200
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      },
      "executeOnce": true
    },
    {
      "parameters": {
        "jsCode": "const books = $input.all().map((b) => b.json);\nreturn $('Mock Incidents').all().map((i) => {\n  const x = i.json;\n  const text = String(x.report).toLowerCase();\n  const hit = books.find((b) => String(b.keywords).split(',').some((k) => text.includes(k.trim().toLowerCase())));\n  return { json: Object.assign({}, x, { playbook: hit ? hit.incident_type + ': ' + hit.steps : 'No matching playbook.' }) };\n});"
      },
      "id": "cc8bd627-8995-492a-8a78-de461290417e",
      "name": "Build Case",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        660,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Investigate this security incident.\nIncident: {{ $json.incident_id }}\nReport: {{ $json.report }}\nKey indicator: {{ $json.indicator }}\n\nMatching playbook:\n{{ $json.playbook }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a security investigation agent. Use the ioc_check tool with the key indicator. Base the containment steps on the playbook you are given. Respond with ONLY a JSON object, no markdown, no extra text: {\"incident_type\":\"phishing|malware|data_exposure|other\",\"priority\":\"P1|P2|P3\",\"likely_cause\":\"one sentence\",\"impact\":\"one sentence\",\"containment\":\"2-4 short steps in one string\"}. P1 means active compromise or exposed data."
        }
      },
      "id": "87f6617f-551c-424d-8659-ac3be27c5cbf",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        900,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "655f7f5d-1cd4-41d7-b366-58a47a198bb9",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        880,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "name": "ioc_check",
        "description": "Checks a domain or resource name for known malicious activity. Input: the indicator only, for example 'bad-domain.example.com'. Returns a verdict and detail as JSON.",
        "jsCode": "const iocs = {\n  'payroll-verify-login.example.com': { verdict: 'malicious', detail: 'Credential-harvesting page registered 2 days ago' },\n  'encoded-ps-c2.example.net': { verdict: 'malicious', detail: 'Known command-and-control domain' },\n  'customer-exports': { verdict: 'not_applicable', detail: 'Internal storage bucket that holds customer export files' }\n};\nreturn JSON.stringify(iocs[String(query).trim().toLowerCase()] || { verdict: 'unknown', detail: 'No data' });"
      },
      "id": "820c3199-5a91-4d43-9c2d-a04d2b08481a",
      "name": "IOC Check Tool",
      "type": "@n8n/n8n-nodes-langchain.toolCode",
      "typeVersion": 1.3,
      "position": [
        1100,
        420
      ]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Build Case').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst priority = ['P1', 'P2', 'P3'].includes(String(r.priority).toUpperCase()) ? String(r.priority).toUpperCase() : 'P2';\nreturn { json: {\n  incident_id: ctx.incident_id,\n  incident_type: r.incident_type || 'other',\n  priority,\n  likely_cause: String(r.likely_cause || '').slice(0, 300),\n  impact: String(r.impact || '').slice(0, 300),\n  containment: String(r.containment || '').slice(0, 600),\n  status: priority === 'P1' ? 'escalated' : 'logged'\n}};"
      },
      "id": "d2de096a-30f3-4bf9-9eb7-2dbfd28c42b7",
      "name": "Parse Report",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1260,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "0bcee7d7-df48-47a1-af7d-58b337806f27",
              "leftValue": "={{ $json.status }}",
              "rightValue": "escalated",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "fef717f2-6c5b-4fa3-8f4c-95378956ba7a",
      "name": "P1 Incident?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1480,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "incident-war-room",
          "mode": "name"
        },
        "text": "=:rotating_light: *{{ $json.priority }} incident {{ $json.incident_id }}* ({{ $json.incident_type }})\nCause: {{ $json.likely_cause }}\nImpact: {{ $json.impact }}\nContainment: {{ $json.containment }}",
        "otherOptions": {}
      },
      "id": "f81372d0-7110-4c9d-949d-b02f0706aa9a",
      "name": "Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1740,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "incident_reports",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "incident_id": "={{ $json.incident_id }}",
            "incident_type": "={{ $json.incident_type }}",
            "priority": "={{ $json.priority }}",
            "likely_cause": "={{ $json.likely_cause }}",
            "impact": "={{ $json.impact }}",
            "containment": "={{ $json.containment }}",
            "status": "={{ $json.status }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "incident_id",
              "displayName": "incident_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "incident_type",
              "displayName": "incident_type",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "priority",
              "displayName": "priority",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "likely_cause",
              "displayName": "likely_cause",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "impact",
              "displayName": "impact",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "containment",
              "displayName": "containment",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "5251af43-b263-4057-a549-183fc8966788",
      "name": "Save Report",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1740,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Incidents",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Incidents": {
      "main": [
        [
          {
            "node": "Get Playbooks",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Playbooks": {
      "main": [
        [
          {
            "node": "Build Case",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build Case": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Report",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Report": {
      "main": [
        [
          {
            "node": "P1 Incident?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "P1 Incident?": {
      "main": [
        [
          {
            "node": "Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Report",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Report",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "IOC Check Tool": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
