# Multilingual AI Chat Assistant

A comprehensive, full-stack AI Chat Assistant application featuring a **Next.js (React)** frontend and a **FastAPI (Python)** backend, with user sessions stored in a **SQLite** database and language-aware responses powered by the **Groq API** (running Llama 3.3).

The chatbot dynamically detects the language, script, and transliteration style of user prompts (e.g. Romanized/transliterated Hinglish or Telugu, native Devanagari or Telugu scripts, English, French, Spanish, etc.) and replies in the matching form.

---

## 🏗️ Architecture Layout

```
chatbot-anti/
├── backend/                  # FastAPI Application
│   ├── app/
│   │   ├── api/              # API Route Handlers (Auth, Chats, Settings)
│   │   ├── core/             # Configuration, Database Setup, Security
│   │   ├── models/           # SQLAlchemy Models (User, Conversation, Message)
│   │   ├── schemas/          # Pydantic Validation Schemas
│   │   ├── services/         # Third-party Integrations (Groq API Service)
│   │   └── main.py           # Application Entrypoint
│   ├── chatbot.db            # SQLite Database File
│   └── requirements.txt      # Python Package Dependencies
└── frontend/                 # Next.js Application
    ├── src/
    │   ├── app/              # Next.js App Router (Pages, Styling)
    │   ├── context/          # React Context (AuthContext for Sessions)
    │   ├── lib/              # Client API Utilities (Axios/Fetch Wrappers)
    │   └── ...
    ├── package.json          # Node Dependencies & Scripts
    └── .env.local            # Frontend Environment Configuration
```

---

## 🔄 Working Process & Data Flow

The flow chart below illustrates the end-to-end working process including user authentication, session initiation, and real-time response streaming.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Browser Client)
    participant FE as Frontend (Next.js App)
    participant BE as Backend (FastAPI)
    database DB as Database (SQLite)
    participant LLM as Groq API (Llama 3.3)

    %% Registration & Authentication
    rect rgb(240, 240, 255)
        note right of User: 1. Authentication Flow
        User->>FE: Enter credentials (Register/Login)
        FE->>BE: POST /api/auth/register OR /api/auth/login
        BE->>DB: Query or insert user (using password hashing)
        DB-->>BE: Return user record
        BE-->>FE: Return JWT Access Token
        FE->>FE: Store token in LocalStorage & set Authenticated State
    end

    %% Session Initialization
    rect rgb(240, 255, 240)
        note right of User: 2. Chat Session Management
        User->>FE: Click "New Chat"
        FE->>BE: POST /api/chats (with Authorization Header)
        BE->>DB: Create Conversation record (default: "New Conversation")
        DB-->>BE: Return conversation record
        BE-->>FE: Return new conversation object
        FE->>FE: Update active conversation ID & clear message area
    end

    %% Chat Streaming Process
    rect rgb(255, 240, 240)
        note right of User: 3. Message Streaming & Multilingual Response
        User->>FE: Enter message and click "Send"
        FE->>FE: Optimistically render user message in UI
        FE->>BE: POST /api/chats/{chat_id}/messages/stream
        BE->>DB: Write user message to DB
        BE->>DB: Fetch previous message history for context
        DB-->>BE: Return message history
        BE->>LLM: POST Stream Request: History + Current message + Temperature + System Rules
        note over BE,LLM: System Rules dictate replying in the exact matching language/script/transliteration
        
        loop SSE Response Stream
            LLM-->>BE: Stream Chunk (Server-Sent Event)
            BE-->>FE: Yield SSE chunk: data: {"content": "text_chunk"}
            FE->>FE: Concatenate text and update screen in real-time
        end

        BE->>DB: Write accumulated full response message to DB
        
        opt Conversation Title is "New Conversation" or empty
            BE->>LLM: Generate short title (max 5 words) based on prompt & response
            LLM-->>BE: Title output
            BE->>DB: Update conversation title in DB
            BE-->>FE: Yield SSE chunk: data: {"title": "Updated Title"}
            FE->>FE: Update conversation list title in Sidebar
        end
    end
```

---

## 🛠️ Installation & Getting Started

### 1. Backend Setup
1. Open a terminal and navigate to the backend folder:
   ```bash
   cd backend
   ```
2. Set up a virtual environment:
   ```bash
   python -m venv venv
   # Activate on Windows:
   .\venv\Scripts\activate
   # Activate on macOS/Linux:
   source venv/bin/activate
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Verify your environment config `.env` (it will be read by `pydantic-settings`):
   ```env
   DATABASE_URL="sqlite:///./chatbot.db"
   JWT_SECRET="supersecretjwtkeyforchatassistant"
   JWT_ALGORITHM="HS256"
   ACCESS_TOKEN_EXPIRE_MINUTES=1440
   GEMINI_API_KEY="your-groq-api-key"
   ```
5. Start the FastAPI development server:
   ```bash
   uvicorn app.main:app --reload --port 8000
   ```

### 2. Frontend Setup
1. Open another terminal in the root directory and navigate to the frontend folder:
   ```bash
   cd frontend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Configure the backend API base url in `.env.local`:
   ```env
   NEXT_PUBLIC_API_URL="http://localhost:8000/api"
   ```
4. Run the Next.js development server:
   ```bash
   npm run dev
   ```
5. Navigate to [http://localhost:3000](http://localhost:3000) on your web browser to start chatting.
