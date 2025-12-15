# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**⚠️ IDE Integration Issues?** See [CLAUDE_IDE_SETUP.md](CLAUDE_IDE_SETUP.md) for troubleshooting VS Code extension and CLI IDE integration in this root/developer user environment.

## Project Overview

This is a **Retrieval-Augmented Generation (RAG) chatbot** for course materials. It uses ChromaDB for vector search, Anthropic's Claude API with tool calling (agentic pattern), and a vanilla JavaScript frontend. Users ask questions about course content and receive AI-powered answers with source attribution.

## Prerequisites

- Python 3.13+
- uv package manager
- Anthropic API key in `.env` file: `ANTHROPIC_API_KEY=your-key-here`
- Git Bash (if on Windows)

## Running the Application

### Start the server
```bash
# Quick start
./run.sh

# Or manually
cd backend
uv run uvicorn app:app --reload --port 8000
```

Access at http://localhost:8000

### Install dependencies
```bash
uv sync
```

### Reset the vector database
If ChromaDB gets corrupted or you need a clean slate:
```bash
rm -rf backend/chroma_db/
# Restart the server - documents will be re-indexed automatically
```

## Architecture

### Core Pattern: Agentic Tool Calling

This is NOT a traditional hard-coded RAG pipeline. Instead:

1. User query → Claude API with tool definitions
2. **Claude decides** whether to use the search tool (autonomous decision)
3. If tool use: execute search → return results → Claude API again
4. Claude synthesizes answer from search results

This two-phase approach (decide + search, then synthesize) is the central design pattern.

### Request Flow

```
Frontend (script.js)
  ↓ POST /api/query
FastAPI Endpoint (app.py)
  ↓ calls
RAGSystem.query() [main orchestrator]
  ↓ calls
AIGenerator.generate_response() [handles Claude API]
  ↓ first API call
Claude decides to use tools
  ↓ stop_reason="tool_use"
ToolManager.execute_tool()
  ↓ calls
CourseSearchTool.execute()
  ↓ calls
VectorStore.search() [ChromaDB semantic search]
  ↓ returns formatted results
AIGenerator makes second API call with results
  ↓ returns synthesized answer
RAGSystem extracts sources from ToolManager
  ↓ saves to SessionManager
Response returns to frontend with sources
```

### Key Components

**RAGSystem (rag_system.py)** - Main orchestrator
- Coordinates all components
- Entry point for queries: `query(query, session_id)`
- Manages conversation history via SessionManager
- Extracts sources from ToolManager after AI generation

**AIGenerator (ai_generator.py)** - Claude API wrapper
- Two-phase generation: `generate_response()` → `_handle_tool_execution()`
- First call: Claude gets tools, decides whether to search
- Second call: Claude receives search results, generates answer
- Temperature: 0 (deterministic), Max tokens: 800

**VectorStore (vector_store.py)** - ChromaDB wrapper
- Two collections: `course_catalog` (metadata), `course_content` (chunks)
- Unified search interface: `search(query, course_name, lesson_number)`
- Fuzzy course name matching for user convenience
- Returns SearchResults dataclass with documents + metadata

**DocumentProcessor (document_processor.py)** - Parses course files
- Expected format: `Course Title:`, `Course Link:`, `Course Instructor:`, then `Lesson N:` markers
- Sentence-based chunking (800 chars, 100 char overlap)
- **Critical**: Adds context to chunks: `"Lesson {n} content: {text}"` or `"Course {title} Lesson {n} content: {text}"`
- Context prefix ensures chunks are self-contained for retrieval

**ToolManager + CourseSearchTool (search_tools.py)** - Tool calling infrastructure
- CourseSearchTool provides `search_course_content` tool definition to Claude
- Parameters: `query` (required), `course_name` (optional), `lesson_number` (optional)
- Tracks sources in `last_sources` for UI display
- ToolManager orchestrates tool registration and execution

**SessionManager (session_manager.py)** - Conversation memory
- Maintains last N exchanges per session (default: 2 exchanges = 4 messages)
- Formats history as: `"User: {q}\nAssistant: {a}\n..."`
- History appended to system prompt for context-aware responses

### Document Processing Format

Course documents in `docs/` must follow this structure:

```
Course Title: Introduction to Python
Course Link: https://example.com/python
Course Instructor: Jane Doe
Lesson 0: Variables and Data Types
Lesson Link: https://example.com/lesson0
[lesson content here...]
Lesson 1: Control Flow
[lesson content here...]
```

- Lines 1-3: Course metadata (title, link, instructor)
- Remaining: Lesson markers (`Lesson N:`) followed by optional `Lesson Link:` then content
- Each lesson is chunked separately with context prefixes

### Configuration (backend/config.py)

Key settings you may need to adjust:

```python
ANTHROPIC_MODEL = "claude-sonnet-4-20250514"  # Claude model
EMBEDDING_MODEL = "all-MiniLM-L6-v2"          # Sentence transformers
CHUNK_SIZE = 800                               # Chars per chunk
CHUNK_OVERLAP = 100                            # Overlap between chunks
MAX_RESULTS = 5                                # Top-K search results
MAX_HISTORY = 2                                # Conversation exchanges to keep
CHROMA_PATH = "./chroma_db"                    # Vector DB location
```

### Data Models (backend/models.py)

```python
Course                  # Full course with lessons
  - title (unique ID)
  - course_link
  - instructor
  - lessons: List[Lesson]

Lesson                  # Individual lesson
  - lesson_number
  - title
  - lesson_link

CourseChunk            # Text segment for vector storage
  - content           # Includes context prefix
  - course_title
  - lesson_number
  - chunk_index
```

### Frontend (frontend/)

- **Vanilla JavaScript** - no frameworks
- `script.js` handles API calls, session management, markdown rendering
- `index.html` provides chat interface + course stats sidebar
- `style.css` implements dark theme
- Uses `marked.js` library for markdown → HTML conversion

## Common Development Scenarios

### Adding a New Tool for Claude

1. Create tool class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` - returns Anthropic tool schema
3. Implement `execute(**kwargs)` - tool logic
4. Register in `RAGSystem.__init__()`: `self.tool_manager.register_tool(YourTool(...))`
5. Tool will be automatically available to Claude

### Modifying Chunking Strategy

Edit `DocumentProcessor.chunk_text()` in `document_processor.py`. Current strategy:
- Sentence-based splitting (regex at line 34)
- Accumulate sentences until CHUNK_SIZE reached
- Backtrack to include CHUNK_OVERLAP chars from previous chunk
- Ensures no mid-sentence cuts

### Changing Vector Search Behavior

Edit `VectorStore.search()` in `vector_store.py`:
- Line 66: Course name fuzzy matching threshold (0.6)
- Line 103: ChromaDB query parameters (n_results, where filters)
- Embedding happens automatically via ChromaDB + sentence-transformers

### Adjusting AI Response Style

Edit `AIGenerator.SYSTEM_PROMPT` in `ai_generator.py` (lines 8-30). Current instructions:
- Use search only for course-specific questions
- One search per query maximum
- No meta-commentary about search process
- Brief, concise, educational responses

### Adding Course Documents

Place `.txt`, `.pdf`, or `.docx` files in `docs/` folder. On server startup (or manual call to `rag_system.add_course_folder()`):
- Files auto-detected and processed
- Duplicate courses skipped (checks existing titles in ChromaDB)
- New courses parsed, chunked, embedded, and indexed

## API Endpoints

- `POST /api/query` - Process user question
  - Request: `{ query: str, session_id?: str }`
  - Response: `{ answer: str, sources: List[str], session_id: str }`

- `GET /api/courses` - Get course statistics
  - Response: `{ total_courses: int, course_titles: List[str] }`

- `GET /docs` - Swagger UI (FastAPI auto-generated)

## Important Implementation Details

### Why Two API Calls to Claude?

The tool calling pattern requires:
1. **Call #1**: Claude sees the query + available tools, returns `tool_use` blocks
2. **Tool execution**: Backend executes the search
3. **Call #2**: Claude sees the search results, generates final answer

This cannot be collapsed into one call - it's how Anthropic's tool calling API works.

### Session Management

- First query from frontend sends `session_id: null`
- Backend creates new session (e.g., `session_1`)
- Session ID returned in response
- Frontend stores it for subsequent queries
- Server maintains conversation history per session

### Source Tracking

Sources are tracked separately from the main response flow:
1. `CourseSearchTool.execute()` populates `self.last_sources`
2. After AI generation, `RAGSystem.query()` calls `tool_manager.get_last_sources()`
3. Sources returned alongside answer
4. `tool_manager.reset_sources()` clears for next query

### Context Prefixing

**Critical for retrieval quality**: Each chunk gets prefixed with context during processing:
- First chunk of lesson: `"Lesson {n} content: {text}"`
- Last lesson chunks: `"Course {title} Lesson {n} content: {text}"`

Without these prefixes, chunks lack context when retrieved. Don't remove them.

### ChromaDB Collections

Two separate collections serve different purposes:
- `course_catalog`: Stores course/lesson metadata for course name resolution
- `course_content`: Stores actual text chunks for semantic search

Both are queried during search for comprehensive results.

## Troubleshooting

**Server won't start**: Check `.env` file has `ANTHROPIC_API_KEY`

**No documents loaded**: Ensure `docs/` folder exists with course files

**ChromaDB errors**: Delete `backend/chroma_db/` and restart

**Tool not being called**: Check `AIGenerator.SYSTEM_PROMPT` - Claude may think it doesn't need to search

**Poor search results**: Adjust `CHUNK_SIZE`, `CHUNK_OVERLAP`, or improve context prefixes in `DocumentProcessor`

**Session history not working**: Verify `SessionManager.MAX_HISTORY` isn't set too low in config
