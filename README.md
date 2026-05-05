# AI-First-CRM-HCP-Interaction-Module
This project implements an AI-first Customer Relationship Management (CRM) system focused on Healthcare Professional (HCP) interaction logging.
## Overview
The system uses a split-screen interface where a conversational AI assistant controls all form inputs. Users describe interactions in natural language, and a LangGraph-based agent processes the input to populate structured CRM fields.

---

## Tech Stack

* Frontend: React
* Backend: FastAPI (Python)
* AI Framework: LangGraph
* Database: SQLite (SQL-based, easily portable to MySQL/PostgreSQL)
* State Management: React State (can be extended to Redux)

---

## Features

* AI-driven form filling (no manual input)
* Split-screen UI (Form + Chat Assistant)
* LangGraph agent with 5 tools:

  * Extract Tool
  * Sentiment Tool
  * Follow-up Tool
  * Edit Tool
  * Save Tool
* Persistent data storage

---

## How to Run

### Backend (Jupyter Notebook)

1. Open backend/main.ipynb
2. Run all cells
3. Ensure server is running at:
   http://127.0.0.1:8000

---

### Frontend (React)

1. Navigate to frontend/ai-crm-ui
2. Install dependencies:
   npm install
3. Start app:
   npm start
4. Open:
   http://localhost:3000

---

## How It Works

1. User enters interaction in chat
2. LangGraph agent processes input
3. Tools extract, analyze, and structure data
4. Form updates automatically
5. Data is saved in database

---

## Notes

SQLite is used for simplicity. The system can be easily migrated to MySQL or PostgreSQL by updating the database connection.

---

## Demo Flow

* Enter: "Met Dr. Sharma, discussion went good, change summary"
* Click "Log / Update"
* Form auto-populates via AI agent

---

## Author

[Abir Ghosh]
