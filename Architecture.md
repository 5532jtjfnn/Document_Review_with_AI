# Project Architecture Analysis

## 1. Overview
The project is an AI-powered document review application utilizing a modern decoupled web architecture. It leverages Python/FastAPI for the backend to handle AI orchestration and TypeScript/React for the frontend user interface, deployed on Azure infrastructure.

## 2. Tech Stack

### Backend (`app/api`)
- **Language**: Python 3
- **Framework**: FastAPI (High-performance web framework)
- **AI/LLM**: `langchain` (Orchestration), `langchain-openai`, `langchain-deepseek`
- **Data Validation**: `pydantic` v2
- **Database**: `aiosqlite` (Async SQLite for dev), Azure Cosmos DB (Prod)
- **Document Processing**: `pymupdf` (PDF Parsing & Search), `MinerU` (external API)
- **Dependency Management**: `requirements.txt`

### Frontend (`app/ui`)
- **Language**: TypeScript
- **Build Tool**: Vite
- **Framework**: React 18
- **UI System**: Microsoft Fluent UI (`@fluentui/react-components`)
- **PDF Viewing**: `react-pdf` with `annotpdf` (In-memory annotation)
- **Authentication**: Azure MSAL (`@azure/msal-react`)
- **Routing**: `react-router-dom` v7

### Infrastructure & DevOps
- **Task Runner**: `Task` (`Taskfile.yml`)
- **IaC**: Terraform (`infra` directory)
- **Cloud**: Microsoft Azure (AI Hub, App Service, Cosmos DB, Blob Storage)
- **Workflow**: PromptFlow (`flows` directory)

---

## 3. Backend Implementation Detail

### Core Logic Flow (`IssuesService`)
1.  **PDF Parsing (MinerU)**:
    -   `MinerUClient` uploads PDF to v4 API, polls for ZIP artifact.
    -   Downloads and parses JSON layout to get paragraphs and bounding boxes.
2.  **Chunking & AI Analysis (LangChain)**:
    -   Text grouped into chunks to fit context windows.
    -   LLM (DeepSeek/GPT-4) identifies issues ("Grammar", "Definitive Language") with strict negative constraints.
    -   Output parsed into `ReviewIssue` objects via Pydantic.
3.  **Coordinate Mapping (Pinpointing)**:
    -   **Strategy 1**: PyMuPDF (`fitz`) exact text search.
    -   **Strategy 2**: MinerU Layout span-level fuzzy match.
    -   **Strategy 3**: Paragraph bounding box fallback.
4.  **Streaming**: Issues streamed to frontend via Server-Sent Events (SSE).

### Human-in-the-Loop (HITL)
-   **LangGraph Agent**: `HitlIssuesAgent` wraps the `update_issue` tool.
-   **Interrupts**: Uses `HumanInTheLoopMiddleware` to halt execution before database updates.
-   **Resume**: User approval in UI triggers `resume_update` to commit changes.

---

## 4. Frontend Implementation Detail

### Review Workspace (`Review.tsx`)
-   **State**: Manages `pdfData` (Uint8Array) and `issues` list locally.
-   **PDF Rendering**: `react-pdf` renders pages; `annotpdf` injects highlight annotations directly into PDF bytes in memory.
-   **Dual View**: Supports "Compare Mode" (Original vs. Annotated) side-by-side.

### Data & Authentication
-   **Streaming API**: `api.ts` uses `fetchEventSource` for SSE.
-   **Auth**: Azure MSAL handles login and token acquisition for API requests.

---

## 5. Infrastructure & Flows

### Azure Resources (Terraform)
-   **AI**: Azure AI Hub & Project, Online Endpoint (Managed).
-   **Compute**: Azure App Service (Linux Web App).
-   **Data**: Cosmos DB (Serverless, SQL API), Blob Storage (`documents`).
-   **Security**: Network Security Perimeters, Key Vault, User Assigned Identities.

### Data Flow Summary
1.  User uploads PDF to Blob Storage.
2.  Frontend calls `GET /review/...`.
3.  Backend downloads PDF, sends to MinerU, streams AI results.
4.  User reviews highlighed issues in dual-view UI.
5.  Accept/Dismiss actions trigger HITL approval flow to update Cosmos DB.
