USER:
Continue specify part 2

ASSISTANT:
The most important design decision for Part 2 is:

> **MCP is the agent-facing contract; the internal REST/event APIs are the service-to-service contract.**

The model should see a relatively small, semantic API. It should **not** see Azure AI Search, Graph, iManage, storage, ACL databases, or model-specific APIs directly.

# 1. API layers

```text
┌─────────────────────────────────────────────────────────────┐
│                     MCP CONTRACT                            │
│                                                             │
│ workspace.*  knowledge.*  analysis.*  artifact.*           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  MATTER INTELLIGENCE API                    │
│                                                             │
│ /workspaces                                                 │
│ /retrieval                                                  │
│ /analysis                                                   │
│ /artifacts                                                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
       Authorization    Retrieval       Generation
          service         service          service
              │             │              │
              ▼             ▼              ▼
         Matter ACL     AI Search       Foundry/
         iManage ACL    M365/iManage    Azure OpenAI
```

# 2. Authentication context

Never accept the user's identity as an MCP argument.

Bad:

```json
{
  "userId": "john.smith",
  "matterId": "MAT-123"
}
```

Instead:

```text
Entra token
     ↓
MCP gateway
     ↓
authenticated principal
     ↓
AuthorizationContext
```

Internally:

```json
{
  "principalId": "aad-object-id",
  "tenantId": "tenant-id",
  "sessionId": "session-id",
  "authenticationMethod": "ENTRA",
  "roles": [
    "LAWYER"
  ]
}
```

The model only supplies the business intent, not the identity.

# 3. AuthorizationContext

```json
{
  "principal": {
    "id": "USER-123",
    "tenantId": "TENANT-1"
  },

  "matter": {
    "id": "MAT-12345"
  },

  "workspace": {
    "id": "WS-987"
  },

  "permissions": {
    "readMatter": true,
    "aiQuery": true,
    "aiGenerate": true,
    "aiExport": true
  },

  "security": {
    "classification": "CONFIDENTIAL",
    "ethicalWalls": [],
    "jurisdiction": "EU"
  }
}
```

Generated server-side.

# 4. Permission model

Make AI permissions explicit:

```text
MATTER_READ
MATTER_WRITE

AI_QUERY
AI_ANALYZE
AI_GENERATE
AI_EXPORT
AI_SHARE
AI_ADMIN
```

Example:

```text
Paralegal
  READ ✓
  AI_QUERY ✓
  AI_GENERATE ✗
  AI_EXPORT ✗

Matter Lawyer
  READ ✓
  AI_QUERY ✓
  AI_GENERATE ✓
  AI_EXPORT ✓

External Counsel
  READ ✓
  AI_QUERY ✓
  AI_GENERATE ✓
  AI_EXPORT restricted
```

# 5. Workspace API

```http
POST /v1/workspaces
```

Request:

```json
{
  "matterId": "MAT-12345",

  "name": "Indemnification Analysis",

  "type": "CURATED",

  "sourceSelection": {
    "mode": "DOCUMENTS",
    "documentIds": [
      "DOC-123",
      "DOC-456",
      "DOC-789"
    ]
  },

  "capabilities": [
    "QNA",
    "SUMMARY",
    "COMPARE",
    "TIMELINE",
    "PRESENTATION"
  ]
}
```

Server verifies:

```text
currentUser ∈ MatterMembership(MAT-12345)
```

# 6. Workspace response

```json
{
  "workspaceId": "WS-987",
  "matterId": "MAT-12345",
  "name": "Indemnification Analysis",

  "status": "READY",

  "sourceCount": 3,

  "capabilities": [
    "QNA",
    "SUMMARY",
    "COMPARE",
    "TIMELINE",
    "PRESENTATION"
  ],

  "createdAt": "2026-10-30T06:00:00Z"
}
```

# 7. Workspace source model

```json
{
  "sourceId": "SRC-123",

  "sourceType": "IMANAGE_DOCUMENT",

  "externalId": "DOC-123",

  "matterId": "MAT-12345",

  "version": 7,

  "title": "Master Agreement",

  "status": "AVAILABLE"
}
```

Support:

```text
IMANAGE_DOCUMENT
M365_DOCUMENT
SHAREPOINT_DOCUMENT
OUTLOOK_MESSAGE
OUTLOOK_THREAD
TEAMS_MESSAGE
M365_ATTACHMENT
WEB_RESOURCE
GENERATED_ARTIFACT
```

# 8. `knowledge.search`

Request:

```json
{
  "workspaceId": "WS-987",

  "query": "indemnification obligations",

  "options": {
    "maxResults": 20,
    "includeCitations": true,
    "includeMetadata": true
  }
}
```

Response:

```json
{
  "queryId": "Q-1001",

  "results": [
    {
      "evidenceId": "EV-1",
      "documentId": "DOC-123",

      "title": "Master Agreement",

      "location": {
        "page": 47,
        "section": "12.3"
      },

      "snippet": "...",

      "score": {
        "retrieval": 0.94,
        "semantic": 0.91
      },

      "citationId": "CIT-1"
    }
  ]
}
```

# 9. `knowledge.ask`

```json
{
  "workspaceId": "WS-987",

  "question":
    "What are the indemnification obligations of the supplier?",

  "answerMode": "LEGAL_ANALYSIS",

  "citationMode": "REQUIRED"
}
```

Response:

```json
{
  "answerId": "ANS-1001",

  "answer": "...",

  "claims": [
    {
      "claimId": "CLM-1",

      "text":
        "The supplier must indemnify the customer for third-party claims arising from IP infringement.",

      "evidence": [
        "EV-1",
        "EV-4"
      ],

      "confidence": "HIGH"
    }
  ],

  "citations": [
    {
      "citationId": "CIT-1",
      "documentId": "DOC-123",
      "page": 47,
      "section": "12.3"
    }
  ]
}
```

The important structure is:

```text
answer
   ↓
claims
   ↓
evidence
   ↓
source
```

# 10. Grounding policy

Support:

```text
STRICT
BALANCED
EXPLORATORY
```

Default legal workspace = **STRICT**.

# 11. Explicit facts vs inference

```json
{
  "claimType": "INFERENCE",

  "text":
    "The supplier may have exposure beyond the contractual cap.",

  "support": [
    "EV-12",
    "EV-19"
  ],

  "reasoning":
    "The carve-out in Section 12.7 appears to exclude IP claims from the cap."
}
```

# 12. `analysis.compare`

```json
{
  "workspaceId": "WS-987",

  "subject": "indemnification",

  "documents": [
    "DOC-123",
    "DOC-456"
  ],

  "dimensions": [
    "scope",
    "cap",
    "exceptions",
    "notice",
    "defence",
    "survival"
  ]
}
```

Response:

```json
{
  "analysisId": "AN-100",

  "rows": [
    {
      "dimension": "cap",

      "documentA": {
        "position": "...",
        "evidence": ["EV-1"]
      },

      "documentB": {
        "position": "...",
        "evidence": ["EV-7"]
      },

      "difference": "...",

      "confidence": "HIGH"
    }
  ]
}
```

# 13. `analysis.timeline`

```json
{
  "workspaceId": "WS-987",

  "topic": "contract dispute",

  "dateRange": {
    "from": "2024-01-01",
    "to": "2026-10-30"
  }
}
```

Response:

```json
{
  "timelineId": "TL-123",

  "events": [
    {
      "date": "2024-02-14",

      "title": "Agreement signed",

      "description": "...",

      "evidence": [
        "EV-101"
      ]
    }
  ]
}
```

# 14. `analysis.find_conflicts`

```json
{
  "workspaceId": "WS-987",
  "subject": "delivery date",
  "entity": "Supplier"
}
```

Response:

```json
{
  "conflicts": [
    {
      "statementA": {
        "text": "...",
        "source": "DOC-123"
      },

      "statementB": {
        "text": "...",
        "source": "EMAIL-456"
      },

      "conflictType": "DATE",

      "assessment": "POTENTIAL_CONTRADICTION"
    }
  ]
}
```

# 15. `artifact.create_presentation`

```json
{
  "workspaceId": "WS-987",

  "request": {
    "title": "Indemnification Dispute",
    "purpose": "CLIENT_UPDATE",
    "audience": "CLIENT",
    "slideCount": 15,

    "instructions": [
      "Explain our position",
      "Highlight key risks",
      "Include contractual provisions",
      "Include relevant chronology"
    ]
  },

  "format": {
    "type": "PPTX",
    "templateId": "LEGAL_CLIENT_STANDARD"
  }
}
```

Response:

```json
{
  "jobId": "JOB-883",
  "artifactId": "ART-991",
  "status": "QUEUED"
}
```

# 16. Presentation pipeline

```text
artifact.create_presentation
             │
             ▼
       Create Job
             │
             ▼
      Retrieve Evidence
             │
             ▼
      Build Evidence Set
             │
             ▼
      Generate Outline
             │
             ▼
       Claim Validation
             │
             ▼
     Generate Slide JSON
             │
             ▼
      Citation Validation
             │
             ▼
       Render PPTX
             │
             ▼
       Visual QA
             │
             ▼
         Artifact
```

# 17. Slide specification

```json
{
  "presentation": {
    "title": "Indemnification Dispute",

    "slides": [
      {
        "id": "S1",
        "layout": "TITLE",
        "title": "Indemnification Dispute"
      },

      {
        "id": "S2",
        "layout": "EXECUTIVE_SUMMARY",

        "claims": [
          {
            "text": "The primary exposure arises from the IP indemnity.",

            "evidence": [
              "EV-12",
              "EV-19"
            ]
          }
        ]
      },

      {
        "id": "S3",
        "layout": "TIMELINE",

        "events": [
          {
            "date": "2024-02-14",
            "title": "Agreement signed",
            "evidence": ["EV-100"]
          }
        ]
      }
    ]
  }
}
```

Renderer owns:

```text
fonts
spacing
colors
layouts
icons
tables
charts
footnotes
citations
```

Model owns:

```text
content
structure
claims
ordering
```

# 18. Artifact lifecycle

Every artifact:

```text
Artifact
│
├── workspaceId
├── matterId
├── creator
├── generationModel
├── generationTime
├── promptVersion
├── sourceSnapshot
├── evidenceSet
├── artifactVersion
├── classification
└── retentionPolicy
```

# 19. Job API

```http
GET /v1/jobs/JOB-883
```

Response:

```json
{
  "jobId": "JOB-883",

  "type": "PRESENTATION_GENERATION",

  "status": "GENERATING",

  "progress": {
    "phase": "RENDERING",
    "percentage": 78
  },

  "createdAt": "...",
  "updatedAt": "..."
}
```

Terminal state:

```json
{
  "status": "COMPLETED",

  "result": {
    "artifactId": "ART-991"
  }
}
```

# 20. MCP handling of long-running jobs

Return:

```json
{
  "jobId": "JOB-883",
  "status": "RUNNING",
  "message":
    "Presentation generation has started."
}
```

Then:

```text
artifact.get_status
```

or generic:

```text
job.get
job.cancel
```

# 21. Artifact access

Never return a permanent blob URL.

```text
GET artifact
      ↓
authorize user
      ↓
verify matter access
      ↓
verify artifact access
      ↓
generate short-lived URL
```

# 22. Workspace sharing

Only allow:

```text
workspace.share
```

when:

```text
currentUser has AI_SHARE
AND
target user has matter access
AND
target user passes ethical-wall check
```

# 23. Workspace authorization

Every operation:

```text
authorize(
    principal,
    action,
    matter,
    workspace,
    sources
)
```

Result:

```json
{
  "allowed": true,

  "effectiveSources": [
    "DOC-123",
    "DOC-456"
  ],

  "deniedSources": [
    "DOC-789"
  ]
}
```

# 24. Workspace source modes

```text
EXPLICIT
DYNAMIC
QUERY
```

EXPLICIT = specific documents.

DYNAMIC = all currently authorized matter documents.

QUERY = dynamic source rule:

```text
matter = 123
AND
documentType = CONTRACT
AND
party = Contoso
```

# 25. Source snapshots

Record:

```json
{
  "evidenceSetId": "ES-883",

  "workspaceId": "WS-987",

  "createdAt": "...",

  "sources": [
    {
      "documentId": "DOC-123",
      "version": 7
    },
    {
      "documentId": "DOC-456",
      "version": 3
    }
  ]
}
```

# 26. Audit event model

```json
{
  "eventType": "AI_QUERY",

  "principalId": "USER-123",

  "matterId": "MAT-12345",

  "workspaceId": "WS-987",

  "operation": "knowledge.ask",

  "queryId": "Q-1001",

  "model": "MODEL-ID",

  "evidenceSetId": "ES-883",

  "timestamp": "..."
}
```

# 27. MCP resources vs tools

Tools for actions:

```text
workspace.create
knowledge.ask
analysis.compare
artifact.create_presentation
```

Resources for stable context:

```text
matter://MAT-12345
workspace://WS-987
artifact://ART-991
```

Potentially:

```text
evidence://EV-123
```

# 28. Tool permission matrix

| Tool | Lawyer | Paralegal | External Counsel | Client |
|---|---:|---:|---:|---:|
| workspace.list | ✓ | ✓ | ✓ | restricted |
| workspace.create | ✓ | policy | ✓ | ✗ |
| knowledge.search | ✓ | ✓ | ✓ | restricted |
| knowledge.ask | ✓ | ✓ | ✓ | restricted |
| analysis.compare | ✓ | ✓ | ✓ | ✗ |
| analysis.timeline | ✓ | ✓ | ✓ | restricted |
| artifact.presentation | ✓ | policy | ✓ | ✗ |
| artifact.export | ✓ | policy | policy | ✗ |
| workspace.share | ✓ | ✗ | ✗ | ✗ |
| workspace.delete | ✓ | ✗ | ✗ | ✗ |

# 29. Prompt injection defence

Source documents are **untrusted input**.

Retrieved document content must be treated as data, never instructions.

```text
SYSTEM POLICY
      ↓
USER INTENT
      ↓
LEGAL WORKSPACE POLICY
      ↓
RETRIEVED DOCUMENTS
      ↓
MODEL
```

Retrieved documents cannot modify:

```text
authorization
tools
system instructions
security policy
```

# 30. Tool-use policy

Don't expose raw:

```text
iManage.search
M365.search
Azure.Search
```

Expose:

```text
knowledge.search
```

as the sanctioned retrieval interface.

# 31. Complete `knowledge.ask` execution

```text
1. MCP request
       │
       ▼
2. Authenticate Entra token
       │
       ▼
3. Resolve principal
       │
       ▼
4. Resolve workspace
       │
       ▼
5. Resolve matter
       │
       ▼
6. Authorize AI_QUERY
       │
       ▼
7. Resolve effective source set
       │
       ▼
8. Apply document ACL
       │
       ▼
9. Apply ethical wall
       │
       ▼
10. Query planner
       │
       ▼
11. Azure AI Search retrieval
       │
       ▼
12. Reranking
       │
       ▼
13. Evidence construction
       │
       ▼
14. LLM reasoning
       │
       ▼
15. Claim/evidence validation
       │
       ▼
16. Citation generation
       │
       ▼
17. Audit
       │
       ▼
18. MCP response
```

# 32. Complete `create_presentation` execution

```text
MCP
 │
 ▼
Authorize AI_GENERATE
 │
 ▼
Resolve workspace
 │
 ▼
Resolve effective sources
 │
 ▼
Create evidence set
 │
 ▼
Retrieve relevant evidence
 │
 ▼
Generate presentation outline
 │
 ▼
Validate outline against evidence
 │
 ▼
Generate slide specification
 │
 ▼
Validate every claim
 │
 ├── unsupported → regenerate/remove
 │
 └── supported
       │
       ▼
Render PPTX
       │
       ▼
Visual QA
       │
       ▼
Citation QA
       │
       ▼
Security classification
       │
       ▼
Store artifact
       │
       ▼
Audit
```

# 33. API versioning

Use:

```text
/v1/workspaces
/v1/retrieval
/v1/analysis
/v1/artifacts
/v1/jobs
```

MCP tools should remain semantically stable.

# 34. Model routing

```json
{
  "task": "LEGAL_ANALYSIS",

  "policy": {
    "reasoning": "HIGH",
    "dataResidency": "EU",
    "provider": "APPROVED_ONLY"
  }
}
```

# 35. Model governance

Record:

```text
model ID
model version
system prompt version
retrieval configuration
workspace ID
evidence set
generation parameters
```

# 36. Initial MCP surface

```text
WORKSPACE
────────────────────────────
workspace.create
workspace.get
workspace.list
workspace.add_source
workspace.remove_source

KNOWLEDGE
────────────────────────────
knowledge.search
knowledge.ask

ANALYSIS
────────────────────────────
analysis.summarize
analysis.compare
analysis.timeline
analysis.issue_matrix
analysis.find_conflicts
analysis.extract_obligations

ARTIFACT
────────────────────────────
artifact.create_presentation
artifact.create_report
3333
JOB
────────────────────────────
job.get
job.cancel
```

# 37. V1 implementation priority

```text
P0
├── Entra authentication
├── Matter authorization
├── Workspace model
├── Source model
├── Evidence model
├── Azure AI Search integration
├── knowledge.search
├── knowledge.ask
└── citations

P1
├── workspace source management
├── analysis.summarize
├── analysis.compare
├── analysis.timeline
├── audit
└── job service

P2
├── presentation specification
├── PPTX renderer
├── artifact service
├── presentation QA
└── artifact lineage

P3
├── contradiction detection
├── obligation extraction
├── issue analysis
├── witness analysis
└── legal workflow templates
```

The next specification is **Part 3: Security, authorization, data model and permission propagation**, including database schemas and the exact algorithm for determining whether a user can retrieve a particular chunk/document from a matter.
