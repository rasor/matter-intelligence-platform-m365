USER:
Please continue producing technical specifications

ASSISTANT:
Absolutely. I would formalize this as a **Matter Intelligence Platform (MIP)**: a security-trimmed, Azure-native "NotebookLM-like" layer exposed through your existing legal MCP.

One important update to the earlier architecture: Azure AI Search now has **agentic retrieval with knowledge bases, knowledge sources, citations, and an MCP endpoint**, so you can use Microsoft's retrieval infrastructure rather than implementing every RAG primitive yourself. Some of the richer query-planning/answer-synthesis capabilities are currently preview, so I would isolate that dependency behind your own retrieval API.

## Technical Specification — Part 1: Target Architecture

### 1.1 System boundary

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         LEGAL AI EXPERIENCE                         │
│                                                                     │
│  ChatGPT / Copilot / Web UI / Teams / Future Clients              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ MCP
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         LEGAL MCP GATEWAY                           │
│                                                                     │
│  • Entra authentication                                             │
│  • Matter authorization                                             │
│  • Ethical walls                                                    │
│  • Tool authorization                                               │
│  • Rate limits                                                      │
│  • Audit                                                            │
│  • Session/context                                                  │
└──────────────┬───────────────────┬───────────────────┬──────────────┘
               │                   │                   │
               ▼                   ▼                   ▼
        ┌────────────┐      ┌──────────────┐   ┌──────────────┐
        │ iManage    │      │ Microsoft 365│   │ Matter       │
        │ MCP        │      │ / Graph      │   │ Security     │
        └────────────┘      └──────────────┘   │ Service      │
                                                └──────┬───────┘
                                                       │
                         ┌─────────────────────────────┘
                         ▼
              ┌─────────────────────────┐
              │ MATTER INTELLIGENCE API │
              │                         │
              │ Workspace               │
              │ Retrieval               │
              │ Grounding               │
              │ Analysis                │
              │ Artifact generation     │
              └────────────┬────────────┘
                           │
          ┌────────────────┼───────────────────┐
          ▼                ▼                   ▼
 ┌────────────────┐ ┌─────────────────┐ ┌─────────────────┐
 │ Azure AI       │ │ Azure OpenAI /  │ │ Artifact        │
 │ Search         │ │ Foundry Models   │ │ Services        │
 │                │ │                 │ │                 │
 │ hybrid search  │ │ reasoning       │ │ PPTX            │
 │ vectors        │ │ synthesis       │ │ DOCX            │
 │ semantic rank  │ │ structured JSON │ │ PDF             │
 └────────────────┘ └─────────────────┘ └─────────────────┘
          ▲
          │
 ┌────────┴────────────────────────────────────────────────┐
 │                 KNOWLEDGE INGESTION                      │
 │                                                        │
 │ M365 / SharePoint / OneDrive / Exchange / iManage      │
 │                                                        │
 │ extract → normalize → ACL → chunk → enrich → index     │
 └─────────────────────────────────────────────────────────┘
```

# 1. Core design principle: three planes

### Control plane

Defines **what exists and who can use it**.

```text
Matter
Workspace
User
Matter membership
Security classification
Ethical wall
Retention
AI policy
```

### Knowledge plane

Defines **what can be retrieved**.

```text
Documents
Emails
Attachments
Contracts
Precedents
Chronologies
Metadata
Embeddings
ACLs
Citations
```

### Intelligence plane

Defines **what can be done with retrieved information**.

```text
Ask
Summarize
Compare
Analyze
Generate timeline
Generate issues
Generate presentation
Generate report
```

# 2. Domain model

Start with:

```text
Matter
MatterMembership
MatterPolicy
KnowledgeWorkspace
KnowledgeSource
Document
DocumentVersion
DocumentPermission
EvidenceChunk
Citation
Conversation
AnalysisJob
Artifact
ArtifactVersion
AuditEvent
```

### Matter

```json
{
  "matterId": "MAT-12345",
  "externalIds": {
    "imanage": "123456",
    "m365": "..."
  },
  "classification": "CONFIDENTIAL",
  "status": "OPEN",
  "region": "EU",
  "aiPolicy": "LEGAL_STANDARD"
}
```

### MatterMembership

```json
{
  "matterId": "MAT-12345",
  "principalId": "entra-user-id",
  "role": "MATTER_LAWYER",
  "validFrom": "...",
  "validTo": null,
  "permissions": [
    "READ",
    "AI_QUERY",
    "AI_GENERATE"
  ]
}
```

Do not infer AI rights simply from document rights.

Potential AI permissions:

```text
READ
AI_QUERY
AI_GENERATE
AI_EXPORT
AI_SHARE
AI_ADMIN
```

# 3. Knowledge Workspace

Your NotebookLM analogue:

```json
{
  "workspaceId": "WS-123",
  "matterId": "MAT-12345",
  "name": "Indemnification Analysis",
  "type": "CURATED",
  "ownerId": "user-123",

  "sourcePolicy": {
    "mode": "EXPLICIT",
    "sources": [
      "DOC-100",
      "DOC-101",
      "DOC-102"
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

Support:

### MATTER

Everything the user is entitled to see in the matter.

### CURATED

Explicitly selected sources.

### ANALYSIS

Temporary workspace assembled for a question.

# 4. Don't duplicate documents unnecessarily

Source systems remain authoritative:

```text
iManage
Microsoft 365
     │
     │ document references
     ▼
Knowledge Layer
```

The search index contains an AI representation:

```text
Document
 ├── metadata
 ├── extracted text
 ├── chunks
 ├── vectors
 ├── ACL metadata
 └── source reference
```

Azure Search is not the system of record.

# 5. Retrieval API

Create:

```http
POST /v1/retrieval/query
```

Request:

```json
{
  "principalId": "USER-123",
  "matterId": "MAT-12345",
  "workspaceId": "WS-987",

  "query": "What obligations does the indemnification clause impose?",

  "options": {
    "reasoning": "medium",
    "maxSources": 20,
    "includeCitations": true,
    "includeMetadata": true
  }
}
```

Response:

```json
{
  "queryId": "Q-123",
  "results": [
    {
      "chunkId": "CHK-123",
      "documentId": "DOC-456",
      "score": 0.91,

      "source": {
        "title": "Master Agreement",
        "page": 47,
        "section": "12.3"
      },

      "content": "...",

      "citation": {
        "documentId": "DOC-456",
        "location": "page 47",
        "sourceUri": "..."
      }
    }
  ]
}
```

The retrieval layer returns **grounding objects**, not merely strings.

# 6. Security trimming

Every retrieval request gets:

```json
{
  "principalId": "USER-123",
  "tenantId": "TENANT-1",
  "matterId": "MAT-12345",
  "workspaceId": "WS-987",
  "roles": [
    "MATTER_LAWYER"
  ],
  "securityContext": {
    "ethicalWalls": [],
    "clearances": [
      "CONFIDENTIAL"
    ]
  }
}
```

Effective filter:

```text
tenant == currentTenant
AND matterId == requestedMatter
AND document.aiEnabled == true
AND user has READ(document)
AND user has AI_QUERY(matter)
AND ethicalWallAllows(user, document)
```

**Never let the LLM construct this filter.**

# 7. Use Azure AI Search agentic retrieval carefully

Azure AI Search's agentic retrieval can:

1. plan multiple subqueries,
2. execute them in parallel,
3. perform keyword/vector/hybrid retrieval,
4. semantically rerank,
5. retain references,
6. return grounding data.

Use your own interface in front:

```text
Legal Retrieval API
        │
        ▼
Azure AI Search Knowledge Base
        │
        ├── iManage index
        ├── M365 index
        ├── SharePoint source
        └── other approved sources
```

# 8. MCP interface

Divide tools into:

### Workspace

```text
workspace.create
workspace.get
workspace.list
workspace.update
workspace.delete
workspace.add_source
workspace.remove_source
```

### Retrieval

```text
knowledge.search
knowledge.ask
knowledge.get_source
knowledge.get_citation
```

### Analysis

```text
analysis.summarize
analysis.compare
analysis.timeline
analysis.issue_matrix
analysis.find_conflicts
analysis.extract_obligations
```

### Legal workflows

```text
legal.build_case_brief
legal.build_witness_profile
legal.build_chronology
legal.build_hearing_brief
legal.build_partner_brief
```

### Artifact generation

```text
artifact.create_presentation
artifact.create_report
artifact.create_document
artifact.create_pdf
artifact.get
artifact.download
```

### Lifecycle

```text
workspace.refresh
workspace.reindex
workspace.validate_access
workspace.revoke_access
```

# 9. MCP tool example

```json
{
  "name": "knowledge.ask",
  "description": "Answer a question using only knowledge the caller is authorized to access.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "workspaceId": {
        "type": "string"
      },
      "question": {
        "type": "string"
      },
      "answerMode": {
        "enum": [
          "concise",
          "detailed",
          "legal_analysis"
        ]
      },
      "citationMode": {
        "enum": [
          "required",
          "optional"
        ]
      }
    },
    "required": [
      "workspaceId",
      "question"
    ]
  }
}
```

Server derives:

```text
principalId
tenantId
matterId
authorization
```

from the authenticated session.

# 10. `knowledge.ask`

Request:

```json
{
  "workspaceId": "WS-987",
  "question": "What are the indemnification obligations of the supplier?",
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
      "text": "The supplier must indemnify the customer for third-party claims arising from IP infringement.",
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

The important addition is **claims**:

```text
answer
   ↓
claims
   ↓
evidence
   ↓
source
```

# 11. Grounding policy

Support:

```text
STRICT
BALANCED
EXPLORATORY
```

### STRICT

Only answer if claims have evidence.

### BALANCED

Reason across evidence but identify inference.

### EXPLORATORY

Generate hypotheses/questions but distinguish:

```text
SOURCE FACT
INFERENCE
HYPOTHESIS
```

Default legal workspace = **STRICT**.

# 12. Facts vs inference

```json
{
  "claimType": "INFERENCE",
  "text": "The supplier may have exposure beyond the contractual cap.",
  "support": [
    "EV-12",
    "EV-19"
  ],
  "reasoning": "The carve-out in Section 12.7 appears to exclude IP claims from the cap."
}
```

# 13. `analysis.compare`

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

# 14. `analysis.timeline`

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

# 15. `analysis.find_conflicts`

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

Use:

```text
CONSISTENT
POTENTIAL_CONTRADICTION
LIKELY_CONTRADICTION
UNRESOLVED
```

# 16. `artifact.create_presentation`

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

# 17. Presentation pipeline

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

LLM generates content; deterministic software generates PowerPoint.

# 18. Slide specification

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
        "title": "Executive Summary",
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
        "title": "Key Events",
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

# 19. Artifact lifecycle

Every artifact has lineage:

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

# 20. Long-running jobs

```text
Analysis Job
│
├── QUEUED
├── RETRIEVING
├── ANALYZING
├── GENERATING
├── VALIDATING
├── RENDERING
├── COMPLETED
└── FAILED
```

Use your own job abstraction even if Foundry/MCP supports long-running operations.

# 21. Artifact authorization

Never expose a permanent blob URL.

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

# 23. Dynamic permission trimming

Workspace is an intent/source collection, not an authorization cache.

If a user loses access to a document:

```text
WS-1
  DOC-1 ✓
  DOC-2 ✗
  DOC-3 ✓
```

The workspace need not necessarily be rebuilt. Authorization trims retrieval dynamically.

# 24. Workspace source modes

Support:

```text
EXPLICIT
DYNAMIC
QUERY
```

### EXPLICIT

Specific documents.

### DYNAMIC

All currently authorized documents in matter.

### QUERY

A dynamic source rule, e.g.:

```text
matter = 123
AND
documentType = CONTRACT
AND
party = Contoso
```

# 25. Source snapshots

For reproducibility, record an evidence set:

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

For sensitive environments, keep immutable audit metadata separate from full prompt/response content.

# 27. MCP resources vs tools

### Tools

Actions:

```text
workspace.create
knowledge.ask
analysis.compare
artifact.create_presentation
```

### Resources

Stable contextual information:

```text
matter://MAT-12345
workspace://WS-987
artifact://ART-991
evidence://EV-123
```

Don't expose raw document content automatically through a resource; retrieve it through authorization-aware operations.

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

A contract or email can contain malicious instructions. Retrieved document content must be treated as data, never as instructions.

Security policy must be outside retrieved context:

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

to the model.

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

Keep MCP tools semantically stable. Avoid model-specific tool names.

# 34. Model routing

Use a model abstraction:

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

Then route internally:

```text
LEGAL_ANALYSIS → approved reasoning model
SUMMARIZATION → faster model
EMBEDDING → embedding model
OCR → document intelligence
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

A good v1 contract:

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
artifact.get

JOB
────────────────────────────
job.get
job.cancel
```

Keep the MCP surface small; the internal platform can be much richer.

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

The next major specification is **Part 3: Security, authorization, data model and permission propagation**, including database schemas and the exact algorithm for determining whether a user can retrieve a particular chunk/document from a matter.

---

