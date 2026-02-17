# Mini-Claw Architecture Diagram (Simplified for PNG Export)

## System Architecture

```mermaid
graph TB
    %% External Layer
    User([👤 Telegram User])

    %% Bot Layer
    subgraph bot["Mini-Claw Bot"]
        direction TB
        Entry[🚀 Entry Point<br/>Config • Auth Check • Startup]
        Commands[🤖 Bot Commands<br/>/start /help /pwd /cd<br/>/shell /session /new]
        Handler[💬 Message Handler<br/>Rate Limit • Workspace]
    end

    %% Services Layer
    subgraph services["Core Services"]
        direction LR
        Rate[⏱️ Rate Limiter]
        Work[📁 Workspace]
        Sess[📚 Sessions]
    end

    %% Pi Layer
    subgraph pi["Pi Agent Layer"]
        direction TB
        Lock[🔒 AsyncLock]
        Exec[⚙️ Pi Executor]
        Stream[📡 Activity Stream]
    end

    %% Processing
    subgraph proc["Processing"]
        direction LR
        Files[📎 File Detector]
        MD[📝 Markdown]
    end

    %% AI Providers
    subgraph ai["AI Providers"]
        direction LR
        Claude[Claude AI]
        GPT[ChatGPT]
    end

    %% Storage
    subgraph storage["Storage"]
        direction TB
        Sessions[(Sessions<br/>JSONL)]
        WS[(Workspace<br/>State)]
        FS[(User Files)]
    end

    %% Flow
    User -->|Message| Commands
    User -->|Message| Handler
    Entry --> Commands
    Handler --> Rate
    Rate --> Work
    Handler --> Lock
    Lock --> Exec
    Exec --> Stream
    Exec --> Claude
    Exec --> GPT
    Exec --> Files
    Exec --> MD
    Stream -.->|Updates| User
    Files --> User
    MD --> User

    Commands --> Sess
    Work <--> WS
    Exec <--> Sessions
    Exec --> FS
    Sess <--> Sessions

    %% Styling
    classDef userStyle fill:#e7f5ff,stroke:#1971c2,stroke-width:3px
    classDef botStyle fill:#d3f9d8,stroke:#2f9e44,stroke-width:2px
    classDef serviceStyle fill:#e5dbff,stroke:#5f3dc4,stroke-width:2px
    classDef piStyle fill:#ffe3e3,stroke:#c92a2a,stroke-width:2px
    classDef procStyle fill:#c5f6fa,stroke:#0c8599,stroke-width:2px
    classDef aiStyle fill:#ffe8cc,stroke:#d9480f,stroke-width:2px
    classDef storageStyle fill:#fff4e6,stroke:#e67700,stroke-width:2px

    class User userStyle
    class Entry,Commands,Handler botStyle
    class Rate,Work,Sess serviceStyle
    class Lock,Exec,Stream piStyle
    class Files,MD procStyle
    class Claude,GPT aiStyle
    class Sessions,WS,FS storageStyle
```

## Message Flow Diagram

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant T as Telegram API
    participant B as Bot Handler
    participant R as Rate Limiter
    participant L as AsyncLock
    participant P as Pi Agent
    participant AI as Claude/GPT
    participant F as File Detector
    participant S as Storage

    U->>T: Send message
    T->>B: Deliver message
    B->>R: Check rate limit
    alt Rate limited
        R-->>U: Wait message
    else Allowed
        R->>B: OK
        B->>L: Acquire lock
        L->>P: Execute Pi
        P->>S: Load session
        P->>AI: API call
        AI-->>P: Response
        P->>S: Save session
        P->>F: Check files
        F->>S: Scan workspace
        P-->>B: Output + files
        L->>L: Release lock
        B->>T: Send response
        T->>U: Deliver response
        alt Files detected
            B->>T: Send files
            T->>U: Files
        end
    end
```

## Component Interaction

```mermaid
graph LR
    subgraph input["Input Processing"]
        MSG[User Message]
        RATE[Rate Check]
        WS[Get Workspace]
    end

    subgraph execution["Execution"]
        LOCK[Acquire Lock]
        PI[Run Pi]
        STREAM[Stream Activity]
    end

    subgraph output["Output Processing"]
        PARSE[Parse Output]
        FILES[Detect Files]
        FORMAT[Format Markdown]
    end

    subgraph delivery["Delivery"]
        SEND[Send Response]
        ATTACH[Send Files]
    end

    MSG --> RATE
    RATE --> WS
    WS --> LOCK
    LOCK --> PI
    PI --> STREAM
    STREAM -.->|Updates| SEND
    PI --> PARSE
    PARSE --> FILES
    PARSE --> FORMAT
    FORMAT --> SEND
    FILES --> ATTACH

    classDef inputClass fill:#d3f9d8,stroke:#2f9e44
    classDef execClass fill:#ffe3e3,stroke:#c92a2a
    classDef outputClass fill:#c5f6fa,stroke:#0c8599
    classDef deliveryClass fill:#e5dbff,stroke:#5f3dc4

    class MSG,RATE,WS inputClass
    class LOCK,PI,STREAM execClass
    class PARSE,FILES,FORMAT outputClass
    class SEND,ATTACH deliveryClass
```

## To Generate PNG:

### Option 1: Online (Mermaid Live Editor)
1. Visit https://mermaid.live/
2. Copy any diagram above
3. Click "PNG" or "SVG" to download

### Option 2: CLI (mermaid-cli)
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i docs/architecture-simple.md -o docs/architecture.png
```

### Option 3: VS Code Extension
1. Install "Markdown Preview Mermaid Support"
2. Open this file in VS Code
3. Right-click diagram → "Export as PNG"
