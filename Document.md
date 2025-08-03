# Manus Project Documentation

## 1. Project Overview

Manus is a sophisticated AI agent platform designed to automate a wide range of tasks that can be performed on a computer with internet access. Users can assign high-level goals, and the Manus agent will autonomously create a plan, execute steps using various tools (like a web browser, shell terminal, and file system), and deliver the final result.

The platform is built with a modern web stack, featuring a Python backend for the agent's core logic and a Vue.js frontend for a rich, interactive user experience.

## 2. System Architecture

The system is composed of several key components that work together to provide the agent's functionality.

### High-Level Architecture Diagram

```mermaid
graph TD
    subgraph User
        A["User's Browser"]
    end

    subgraph "Frontend [Frontend (Vue.js)]"
        B["Web UI"]
        C["Live View (noVNC)"]
        D["API Client (Axios, SSE)"]
    end

    subgraph "Backend [Backend (Python/FastAPI)]"
        E["Web API"]
        F["Agent Service"]
        G["Session Service"]
        H["Repositories"]
    end

    subgraph Agent Core
        I["Planner"]
        J["Executor"]
    end

    subgraph Sandbox Environment
        K["Tools: Browser, Shell, Files"]
    end

    subgraph Database
        L["MongoDB"]
    end

    A -- Interacts with --> B
    B -- Uses --> D
    C -- Connects to --> K
    D -- HTTP/SSE --> E
    E -- Uses --> F
    E -- Uses --> G
    F -- Orchestrates --> I
    F -- Orchestrates --> J
    G -- Uses --> H
    H -- R/W --> L
    I -- Creates/Updates Plan --> F
    J -- Executes Step --> K
    K -- Returns Result --> J
```

### Components

*   **Frontend**: A Vue.js single-page application that provides the main user interface. It allows users to manage tasks, interact with the agent, view a live stream of the agent's desktop environment, and browse files generated during a task.
*   **Backend**: A Python application, likely built with FastAPI, that serves the web API and orchestrates the agent's lifecycle.
*   **Agent Core**: The "brain" of the system, composed of two main LLM-powered components:
    *   **Planner**: Analyzes the user's request and generates a high-level, multi-step plan to achieve the goal.
    *   **Executor**: Takes one step from the plan at a time and determines the most appropriate tool and parameters to execute it.
*   **Sandbox Environment**: An isolated environment (e.g., a Docker container) where the agent's tools are run. This includes a web browser (controlled via Playwright), a Linux shell, and a file system. This ensures that agent actions are contained and do not affect the host system.
*   **Database**: A MongoDB instance used for persistence. It stores all data related to user sessions, agent memories, task progress, and generated files.

## 3. Backend Deep Dive

The backend is responsible for managing the entire lifecycle of a task, from creation to completion.

### Task Execution Workflow

The core workflow is event-driven and follows a Planner-Executor pattern. The following sequence diagram illustrates the process of a typical task.

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend API
    participant AgentService
    participant Planner
    participant Executor
    participant Tools

    User->>Frontend: Submits a new task (e.g., "Research X and write a report")
    Frontend->>Backend API: POST /api/sessions (create new task)
    Backend API->>AgentService: start_task(user_message)
    
    AgentService->>Planner: create_plan(user_message)
    note right of Planner: LLM call using PLANNER_SYSTEM_PROMPT
    Planner-->>AgentService: Returns plan (goal, steps)
    AgentService->>Backend API: Persist plan, stream update to Frontend
    
    loop For each step in plan
        AgentService->>Executor: execute_step(current_step)
        note right of Executor: LLM call using EXECUTION_SYSTEM_PROMPT
        Executor-->>AgentService: Chooses tool (e.g., browser.search)
        
        AgentService->>Tools: Run tool action
        Tools-->>AgentService: Return observation (e.g., search results)
        
        AgentService->>Backend API: Persist event, stream update to Frontend
        
        opt Re-planning needed
            AgentService->>Planner: update_plan(observation)
            Planner-->>AgentService: Returns updated steps
        end
    end
    
    AgentService->>Executor: conclude_task()
    note right of Executor: LLM call using CONCLUSION_PROMPT
    Executor-->>AgentService: Returns final message and attachments
    AgentService->>Backend API: Persist conclusion, stream final update
    Backend API-->>Frontend: Task Completed
```

### Key Models and Repositories

Data is structured using Pydantic models and persisted to MongoDB via repository classes. This follows the Repository Pattern, decoupling the domain logic from the data access layer.

```mermaid
classDiagram
    direction LR

    class Session {
        +string id
        +string title
        +SessionStatus status
        +list~BaseEvent~ events
        +list~FileInfo~ files
        +datetime created_at
        +datetime updated_at
    }

    class Agent {
        +string id
        +dict~str, Memory~ memories
    }

    class Memory {
        +list~Message~ messages
    }

    class FileInfo {
        +string file_id
        +string file_path
        +int size
    }

    class BaseEvent {
        <<Abstract>>
        +string event_type
        +dict data
    }

    class SessionRepository {
        +save(Session)
        +find_by_id(string) Optional~Session~
        +add_event(string, BaseEvent)
        +add_file(string, FileInfo)
    }

    class AgentRepository {
        +save(Agent)
        +find_by_id(string) Optional~Agent~
        +add_memory(string, string, Memory)
        +get_memory(string, string) Memory
    }

    SessionRepository ..> Session : Manages
    AgentRepository ..> Agent : Manages
    Session "1" *-- "0..*" BaseEvent
    Session "1" *-- "0..*" FileInfo
    Agent "1" *-- "0..*" Memory
```

*   **`MongoSessionRepository`**: Manages the `Session` documents. Each session corresponds to a single task, tracking its status, title, a complete log of all events (tool calls, results, messages), and a list of associated files.
*   **`MongoAgentRepository`**: Manages the `Agent` documents. This is used for storing the agent's long-term memories, allowing it to learn and improve over time.

### Browser Tool (`PlaywrightBrowser`)

A crucial tool for the agent is the `PlaywrightBrowser`. It's more than a simple wrapper around Playwright; it's designed specifically for AI control.

*   **Initialization**: It connects to an existing Chrome instance over the Chrome DevTools Protocol (CDP), allowing it to control a browser running within the sandbox.
*   **Content Extraction**: The `_extract_content` method uses an LLM to summarize the visible HTML content into clean, readable Markdown. This abstracts the complexity of raw HTML from the Executor agent.
*   **Interactive Elements**: The `_extract_interactive_elements` method scans the viewport for clickable or interactive elements (`<a>`, `<button>`, `<input>`, etc.). It assigns a unique `data-manus-id` to each, returning a simple list like `0:<button>Login</button>`. The Executor can then issue a command like `browser.click(index=0)` without needing to know complex selectors.

## 4. Frontend Deep Dive

The frontend provides a comprehensive and user-friendly interface for interacting with the Manus agent.

### Key Features

*   **Task Management**: Create new tasks and view a history of past tasks.
*   **Real-time Progress**: The UI receives a stream of events from the backend, showing the agent's thoughts, the tools it's using, and the results in real-time.
*   **Live View**: An embedded noVNC client (`@novnc/novnc`) streams the agent's desktop from the sandbox, allowing the user to see exactly what the agent is doing in the browser or terminal.
*   **File Management**: A file explorer lists all files created or modified by the agent during a task. It provides previews for common file types (code, markdown, images) and a download option.
*   **Internationalization**: The UI supports multiple languages (English and Chinese are provided) using `vue-i18n`.

### File Handling (`fileType.ts`)

The `getFileType` and `getFileTypeText` utilities in `/frontend/src/utils/fileType.ts` are central to the file preview feature.

*   They maintain lists of file extensions for different categories (code, images, documents, etc.).
*   `getFileType(filename)`: Based on the file extension, this function returns the appropriate Vue components for the file's icon (`FileIcon`, `CodeFileIcon`) and its previewer (`MarkdownFilePreview`, `ImageFilePreview`, `CodeFilePreview`).
*   `getFileTypeText(filename)`: Returns a localized, human-readable string for the file type (e.g., "Image", "PDF", "代码").

## 5. Development & Tooling

### Documentation Sync (`update_doc.sh`)

The repository includes a utility script, `update_doc.sh`, to help keep documentation synchronized with the source code.

*   **Purpose**: It finds all `.md` files in the project.
*   **Mechanism**: It looks for special comment blocks, like `<!-- .env.example -->` and `<!-- /.env.example -->`.
*   **Action**: It replaces the content between these tags with the full, up-to-date content of the specified file (e.g., `.env.example`), wrapping it in a markdown code block with the appropriate language identifier. This is extremely useful for ensuring that code examples in documentation are never stale.
