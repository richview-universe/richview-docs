# RichView Architecture Overview

**Last Updated:** February 26, 2026
**Author:** OpenClaw AI Analysis

---

## Executive Summary

RichView is a SaaS platform that transforms static business content (reports, presentations) into interactive, AI-powered visual experiences. The core value proposition is **AI-assisted chart creation** - users describe charts in natural language, and the system generates HighCharts visualizations.

**Key Innovation:** GPT-4 powered chart generation with conversation history, enabling iterative refinement of visualizations.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RICHVIEW PLATFORM                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────────┐   │
│  │   FRONTEND   │     │   ML SERVICE │     │     REST API             │   │
│  │   (MVP)      │────▶│   (Flask)    │────▶│   (RichViewPublicRestAPI)│   │
│  │   Next.js    │     │   Python     │     │   Python/Flask           │   │
│  │   TypeScript │     │   OpenAI     │     │   MongoDB                │   │
│  └──────────────┘     └──────────────┘     └──────────────────────────┘   │
│         │                    │                         │                    │
│         │                    │                         │                    │
│         ▼                    ▼                         ▼                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        INFRASTRUCTURE                                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │  │
│  │  │   Auth0     │  │   MongoDB   │  │ Azure Blob  │  │ Google Cloud│ │  │
│  │  │  (Auth)     │  │  (Storage)  │  │  (CSVs)     │  │   Run       │ │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                          CLIENT SDK                                    │  │
│  │                    RichViewSDK (Python)                                │  │
│  │    - Session management                                                │  │
│  │    - Report CRUD                                                       │  │
│  │    - Chart manipulation                                                │  │
│  │    - Pandas DataFrame integration                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

| Repository | Language | Purpose | Key Technologies |
|------------|----------|---------|------------------|
| **mvp** | TypeScript | Frontend web application | Next.js, React, Auth0, EditorJS |
| **mlservice** | Python | AI chart generation service | Flask, OpenAI API, Azure OpenAI |
| **RichViewPublicRestAPI** | Python | Main backend API | Flask, MongoDB, JWT Auth |
| **RichViewSDK** | Python | Python client library | requests, pandas, recordclass |
| **user_management** | Python | Shared auth utilities | Auth0, environment config |

---

## Component Details

### 1. Frontend (mvp)

**Tech Stack:** Next.js, TypeScript, React, Auth0, EditorJS

**Key Features:**
- User authentication via Auth0
- Report editor using EditorJS block-based architecture
- Real-time chart preview with HighCharts
- Multi-tab interface for concurrent editing
- HTML sanitization for security

**Key Files:**
- `pages/api/auth/[...auth0].ts` - Auth0 integration
- `utils/reports.ts` - Report data cleaning
- `utils/htmlSanitizer.ts` - Security sanitization
- `utils/date.ts` - Date formatting

**Deployment:** Google Cloud Run
- Image: `gcr.io/cogent-spirit-380717/nextjs-app`
- Region: europe-west1
- URL: https://therichview.com

**Block Types Supported:**
- Chart (AI-generated HighCharts)
- Paragraph
- Header
- Image
- Table
- Embed
- Code
- List

---

### 2. ML Service (mlservice)

**Tech Stack:** Python, Flask, OpenAI API, Azure OpenAI, MongoDB

**Purpose:** Generate HighCharts JSON from natural language prompts

**Key API Endpoint:**
```
POST /chart
Body: {
  "session_id": "uuid",
  "prompt": "Generate a bar chart of sales by region",
  "model": "gpt-4",
  "data": "base64-encoded-csv"
}
Response: {
  "session_id": "uuid",
  "response": "HighCharts JSON options",
  "prompt": "adjusted prompt with data context"
}
```

**Key Features:**
- **Conversation Memory:** MongoDB stores chat history per session
- **CSV Processing:** Extracts headers, first row, last row for context
- **Data Augmentation:** Appends CSV metadata to prompts
- **Retry Logic:** Re-requests valid JSON if parsing fails
- **Multi-model Support:** OpenAI and Azure OpenAI

**System Prompt Strategy:**
The system prompt instructs GPT-4 to:
1. Output only valid JSON (no commentary)
2. Minimize character count while maintaining functionality
3. Handle follow-up requests by merging with previous options
4. Use CSV URLs instead of embedding data in series

**Environment Variables Required:**
- `OPENAI_API_KEY`
- `AZURE_OPENAI_API_KEY`
- `AI_MONGO_DB_CONNECTION_STRING`
- `AZURE_CSV_CONNECTION_STRING`

---

### 3. REST API (RichViewPublicRestAPI)

**Tech Stack:** Python, Flask, MongoDB, JWT

**Purpose:** Core backend for report CRUD operations

**Key Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/login` | Authenticate, returns JWT |
| GET | `/api/reports` | List all user reports |
| GET | `/api/report/<id>` | Get single report |
| POST | `/api/report` | Create new report |
| PUT | `/api/report/<id>` | Update report |
| DELETE | `/api/report/<id>` | Delete report |
| POST | `/api/report/<id>/send` | Transfer report to another user |
| GET | `/api/report/<id>/charts` | List all charts in report |
| GET | `/api/report/<id>/chart/<cid>` | Get single chart |
| POST | `/api/report/<id>/chart` | Add chart to report |
| PUT | `/api/report/<id>/chart/<cid>` | Update chart |
| DELETE | `/api/report/<id>/chart/<cid>` | Delete chart |
| GET | `/api/report/<id>/chart/<cid>/data` | Get chart data as CSV |
| PUT | `/api/report/<id>/chart/<cid>/data` | Update chart data |
| PUT | `/api/report/<id>/chart/<cid>/title` | Update chart title |

**Security Features:**
- JWT token authentication (24-hour expiry)
- URL encryption/decryption for CSV URLs
- User-scoped data access (email-based ownership)

**Data Model:**
```json
{
  "id": "uuid",
  "title": "Report Title",
  "author_id": "user@email.com",
  "lastEdited": 1709000000000,
  "content": {
    "time": 1709000000000,
    "version": "2.28.0",
    "blocks": [
      {
        "id": "abc123",
        "type": "chart",
        "data": {
          "chartOptions": "{...HighCharts JSON...}"
        }
      }
    ]
  }
}
```

---

### 4. Python SDK (RichViewSDK)

**Purpose:** Programmatic access to RichView for automation

**Installation:**
```python
pip install richviewsdk
```

**Key Classes:**

| Class | Purpose |
|-------|---------|
| `RichViewSession` | Authentication and API client |
| `RichViewReport` | Report container with CRUD methods |
| `ChartBlock` | Chart manipulation with pandas integration |
| `ImageBlock` | Image handling |
| `HeaderBlock` | Header text |
| `ParagraphBlock` | Paragraph text |
| `TableBlock` | Table data |
| `CodeBlock` | Code snippets |
| `ListBlock` | List items |
| `EmbedBlock` | Embedded content |

**Usage Example:**
```python
from richviewsdk import RichViewSession, ParagraphBlock

# Authenticate
session = RichViewSession.authenticate_with_password(
    email="user@example.com",
    password="secret"
)

# Get all reports
reports = session.get_reports()

# Get specific report
report = session.get_report("report-uuid")

# Get charts and update data
charts = report.get_charts()
chart = report.get_chart("chart-id")

# Pandas integration
df = chart.get_data()  # Returns DataFrame
chart.set_data(new_df)  # Updates from DataFrame

# Add new block
report.add_block(ParagraphBlock(
    report_id=report.report_id,
    text="New paragraph",
    session=session
))

# Save changes
report.update_online_version()
```

**Chart Data Format:**
- Index: `Category` or `Category X | Category Y` (for 2D)
- Columns: `Series Name | Chart Type`
- Supports multi-index for heatmaps

---

### 5. User Management (user_management)

**Purpose:** Shared authentication utilities

**Key Files:**
- `shared_code/environs.py` - Environment variable loading
- `shared_code/helper.py` - Auth helper functions
- `shared_code/helper_styling.py` - UI styling helpers

---

## Data Flow

### Chart Creation Flow

```
User Input (Natural Language)
         │
         ▼
    ┌─────────┐
    │  MVP    │ (Frontend)
    │Next.js  │
    └────┬────┘
         │ POST /chart with prompt + optional CSV
         ▼
    ┌─────────┐
    │ML Service│ (Python/Flask)
    │         │
    │ 1. Extract CSV headers/rows if URL present
    │ 2. Augment prompt with data context
    │ 3. Retrieve conversation from MongoDB
    │ 4. Call OpenAI API with system prompt
    │ 5. Parse JSON response
    │ 6. Store updated conversation
    └────┬────┘
         │ HighCharts JSON
         ▼
    ┌─────────┐
    │  MVP    │ (Frontend)
    │ Render  │ HighCharts component
    └─────────┘
```

### Report Save Flow

```
EditorJS Content
         │
         ▼
    ┌─────────┐
    │  MVP    │ Clean report (remove undefined charts)
    └────┬────┘
         │ PUT /api/report
         ▼
    ┌─────────┐
    │REST API │ (Python/Flask)
    │         │
    │ 1. Decrypt URLs
    │ 2. Update timestamp
    │ 3. Validate structure
    │ 4. Save to MongoDB
    │ 5. Encrypt URLs in response
    └─────────┘
```

---

## Infrastructure

### Cloud Services

| Service | Provider | Purpose |
|---------|----------|---------|
| Hosting | Google Cloud Run | Frontend container hosting |
| Authentication | Auth0 | User auth, JWT tokens |
| Database | MongoDB Atlas | Report storage, conversation history |
| Blob Storage | Azure Blob | CSV file storage |
| AI | OpenAI/Azure OpenAI | GPT-4 chart generation |

### URLs

- **Production:** https://therichview.com
- **API:** https://api.therichview.com
- **ML Service:** Internal (port 5000)

---

## Security Considerations

1. **Authentication:** Auth0 + JWT with 24-hour expiry
2. **URL Encryption:** CSV URLs encrypted before client delivery
3. **User Scoping:** All data scoped to author email
4. **Input Sanitization:** HTML sanitization with `sanitize-html`
5. **CORS:** Configured for production domain

---

## Known Limitations

1. **No Real-time Collaboration:** Single-user editing only
2. **No Version History:** Updates overwrite previous versions
3. **CSV Size Limits:** Row cap of 100,000, column cap of 100
4. **No Offline Support:** Requires internet connection
5. **Limited Export:** No PDF/PowerPoint export

---

## Technology Decisions

### Why EditorJS?
- Block-based architecture matches report structure
- Extensible for custom block types (ChartBlock)
- Clean JSON output for storage

### Why HighCharts?
- Industry-standard charting library
- Extensive chart type support
- CSV URL loading capability
- Good documentation

### Why MongoDB?
- Flexible schema for varying block types
- Good for document storage (reports)
- Easy to query by author_id

### Why Separate ML Service?
- Scalability: Can scale independently
- Language choice: Python better for ML/AI
- Isolation: Failures don't affect core API
- Cost: Can optimize compute for ML workloads

---

## Potential Improvements

1. **Real-time Collaboration:** Add WebSockets/Crdt
2. **Version History:** Implement document versioning
3. **Export Options:** Add PDF, PowerPoint, Excel export
4. **Offline Mode:** Service worker caching
5. **Chart Templates:** Pre-built chart configurations
6. **Data Connectors:** Direct database/file connectors
7. **Embedding:** White-label embedding for enterprises
8. **Analytics:** Usage tracking and insights

---

## Getting Started

### Prerequisites
- Node.js 18+ (for MVP)
- Python 3.9+ (for services)
- Docker (for local development)
- MongoDB Atlas account
- Auth0 account
- OpenAI API key
- Azure account (for blob storage)

### Local Development

```bash
# Clone all repos
git clone https://github.com/richview-universe/mvp.git
git clone https://github.com/richview-universe/mlservice.git
git clone https://github.com/richview-universe/RichViewPublicRestAPI.git

# MVP
cd mvp
npm install
npm run dev

# ML Service
cd mlservice
pip install -r requirements.txt
python app.py

# REST API
cd RichViewPublicRestAPI
pip install -r requirements.txt
python app.py
```

---

## Contact & Resources

- **GitHub Organization:** https://github.com/richview-universe
- **Production Site:** https://therichview.com
- **API Base URL:** https://api.therichview.com

---

*This document was auto-generated by OpenClaw AI based on source code analysis.*
