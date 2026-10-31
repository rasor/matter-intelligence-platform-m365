# matter-intelligence-platform-m365
MCP server having google NotebookLM features for Legal Lifecycle having resources in M356 and iManage

What is `Matter Intelligence Platform`?

It is an architecture utilizing
* Microsoft Foundry
* Azure AI Search
* Documents in M365 and iManage

It is a base for building AI applications for legal matters.
It is a central **Legal knowledge MCP** providing
* Knowledge
  * Ask this matter
  * Search sources
  * Compare documents
  * Find contradictions
* Analysis
  * Timeline
  * Issues
  * Obligations
  * Witnesses
  * Evidence
* Create
  * Case brief
  * Partner briefing
  * Client briefing
  * Slide deck
  * Chronology
  * Hearing preparation

Every generated assertion → source → document → permission → audit trail

M365 ACLs together with iManage and ethical wall check will ensure access is controlled.

With the platform 
* the `Create` features above can be thought of NotebookLM features to create artifacts, that are usable in a legal context.  
* the `Analysis` features are features you would expect from AI.  
* the `Knowledge` features are search features from a RAG search. 

## Technical Specification

### [Target Architecture](./plan/a02-mip-initial-plan-by-chatGPT/003-mip-techspec-target-architecture.md)

### [MCP Contract](./plan/a02-mip-initial-plan-by-chatGPT/004-mip-techspec-mcp-contract.md)

The MCP tools will have individual access rights e.g.

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

An access filter enforced by an authorization service will do something like this;
```text
tenant == currentTenant
AND matterId == requestedMatter
AND document.aiEnabled == true
AND user has READ(document)
AND user has AI_QUERY(matter)
AND ethicalWallAllows(user, document)
```

The design includes several legal context features such as
* Conflicting statements can be found.
* All document versions are stored.
* Returned knowlegde:
  * carries citations to the source.
  * is categorized in chuncks of claim-types, so the applications will know if the knowlegde is a fact or if it is AI guessing. 
    * Default `grounding` policy would be `STRICT`, which would only answer if claims have evidence. 
* Ethical wall check avoid conflict of interest from accessing confidential information related to a specific matter.
* User action are recorded with audit events.
* The generated artifacts carries trail to the sources including source document version.
* Clients are separated in tenats enabling external clients to have restricted access to their workspace.

The design includes AI features such as
* Subscribe on a long running AI job, so that you in the app can see, when the job is done

# References
* ChatGPT [research](https://chatgpt.com/share/6ae59b92-a390-83ed-8b96-45fb1dd16896). 
  * [Similar as markdown](https://chatgpt.com/library/share/libfile_e317f0cc5b2c81918c7b63df4783b984?account_id=personal).

---
