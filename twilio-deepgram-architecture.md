# Twilio + Deepgram Architecture Diagram

## System Architecture

```mermaid
flowchart TB
    subgraph External["External Systems"]
        Phone["📞 Phone Call<br/>(PSTN)"]
        Caller["👤 Caller"]
    end

    subgraph Twilio["Twilio Platform"]
        TwilioAPI["Twilio Voice API<br/>- Call Routing<br/>- Media Streams<br/>- Call Control"]
        TwiML["TwiML Endpoints<br/>- /inbound_call<br/>- /call-status<br/>- /recording_callback"]
    end

    subgraph AppServer["FastAPI Application Server"]
        direction TB
        
        subgraph WebSocket["WebSocket Handler"]
            WSEndpoint["WS Endpoint<br/>/twilio/{company}/{lang}"]
            
            subgraph Tasks["Concurrent Tasks"]
                TwilioReceiver["twilio_receiver<br/>(Decode Audio)"]
                TwilioSender["twilio_sender<br/>(Encode Audio)"]
                DGSender["dg_sender<br/>(Forward to DG)"]
                DGReceiver["dg_receiver<br/>(Process DG)"]
                SilenceWatch["silence_watchdog<br/>(Monitor Activity)"]
                HangupWatch["hangup_watchdog<br/>(Auto Hangup)"]
            end
        end
        
        subgraph Business["Business Logic"]
            FunctionHandler["Function Handler<br/>- lookup_loan<br/>- make_payment<br/>- transfer_call<br/>- hangup_call<br/>- play_dtmf"]
            CallControl["Call Control<br/>- Hangup<br/>- Transfer<br/>- DTMF"]
            i18n["Internationalization<br/>- English<br/>- Spanish"]
        end
        
        subgraph ServerPool["Server Pool Manager"]
            LoadBalancer["Load Balancer<br/>- Round Robin<br/>- Least Recently Used"]
            HealthMonitor["Health Monitor"]
        end
    end

    subgraph Deepgram["Deepgram Platform"]
        DGAPI["Deepgram Voice AI<br/>- Speech-to-Text<br/>- Text-to-Speech<br/>- Function Calling"]
        DGModels["AI Models<br/>- nova-2<br/>- enhanced<br/>- base"]
    end

    subgraph DataLayer["Data Layer"]
        direction TB
        
        PostgreSQL[(PostgreSQL Database)]
        
        subgraph Tables["Database Tables"]
            CDR["CDR Table<br/>- call_sid<br/>- duration<br/>- status<br/>- direction"]
            ConvoLog["Conversation Log<br/>- transcripts<br/>- timestamps<br/>- events"]
            Functions["Function Definitions<br/>- schemas<br/>- handlers"]
            CompanyConfig["Company Config<br/>- settings<br/>- prompts"]
        end
    end

    subgraph Middleware["External Middleware"]
        RepayAPI["Repay Middleware<br/>- Loan Lookup<br/>- Payment Processing<br/>- ABA Validation"]
        BankAPI["Bank APIs<br/>- Routing Validation"]
    end

    subgraph Logging["Logging & Monitoring"]
        ConvoLogger["Conversation Logger<br/>- Structured Logs<br/>- Event Tracking"]
        CDRLogger["CDR Logger<br/>- Call Metrics<br/>- Duration<br/>- Status"]
        FileLogger["File Logger<br/>- Agent Prompts<br/>- DG Configs<br/>- Timestamps"]
    end

    %% External to Twilio
    Caller -->|Dials| Phone
    Phone -->|PSTN| TwilioAPI

    %% Twilio to App Server
    TwilioAPI -->|HTTP Webhook| TwiML
    TwiML -->|WebSocket Stream| WSEndpoint
    
    %% WebSocket Internal Flow
    WSEndpoint --> TwilioReceiver
    WSEndpoint --> TwilioSender
    WSEndpoint --> DGSender
    WSEndpoint --> DGReceiver
    WSEndpoint --> SilenceWatch
    WSEndpoint --> HangupWatch
    
    TwilioReceiver -->|Audio Queue| DGSender
    DGReceiver -->|Audio Queue| TwilioSender
    
    %% App Server to Deepgram
    DGSender -->|WebSocket Audio| DGAPI
    DGAPI -->|Transcripts & Audio| DGReceiver
    DGAPI --> DGModels
    
    %% Function Calling Flow
    DGReceiver -->|Function Requests| FunctionHandler
    FunctionHandler -->|Function Response| DGAPI
    FunctionHandler --> CallControl
    FunctionHandler --> i18n
    
    %% Call Control
    CallControl -->|REST API| TwilioAPI
    
    %% Server Pool
    TwiML -.->|Route Selection| LoadBalancer
    LoadBalancer -.->|Health Check| HealthMonitor
    
    %% Business Logic to Middleware
    FunctionHandler -->|Loan Data| RepayAPI
    FunctionHandler -->|Validate ABA| BankAPI
    
    %% Database Operations
    FunctionHandler -->|Query/Update| PostgreSQL
    PostgreSQL --> CDR
    PostgreSQL --> ConvoLog
    PostgreSQL --> Functions
    PostgreSQL --> CompanyConfig
    
    %% Logging Flow
    DGReceiver -->|Log Events| ConvoLogger
    WSEndpoint -->|Log Calls| CDRLogger
    FunctionHandler -->|Log Prompts| FileLogger
    
    ConvoLogger -->|Store| ConvoLog
    CDRLogger -->|Store| CDR
    
    %% Styling
    classDef twilioStyle fill:#F22F46,stroke:#333,stroke-width:2px,color:#fff
    classDef deepgramStyle fill:#13EF93,stroke:#333,stroke-width:2px,color:#000
    classDef appStyle fill:#4A90E2,stroke:#333,stroke-width:2px,color:#fff
    classDef dataStyle fill:#F5A623,stroke:#333,stroke-width:2px,color:#000
    classDef logStyle fill:#7B68EE,stroke:#333,stroke-width:2px,color:#fff
    
    class TwilioAPI,TwiML twilioStyle
    class DGAPI,DGModels deepgramStyle
    class WSEndpoint,FunctionHandler,CallControl,LoadBalancer appStyle
    class PostgreSQL,CDR,ConvoLog,Functions,CompanyConfig dataStyle
    class ConvoLogger,CDRLogger,FileLogger logStyle
```

## Detailed Data Flow

```mermaid
sequenceDiagram
    participant Caller
    participant Twilio
    participant FastAPI
    participant Deepgram
    participant Database
    participant Middleware

    Caller->>Twilio: Initiates Call
    Twilio->>FastAPI: POST /twilio/inbound_call
    FastAPI->>FastAPI: Generate TwiML with Stream
    FastAPI-->>Twilio: Return TwiML
    
    Twilio->>FastAPI: WebSocket Connect
    FastAPI->>Deepgram: WebSocket Connect
    FastAPI->>Database: Load Company Config
    Database-->>FastAPI: Config & Prompts
    
    Note over FastAPI: Start 6 Concurrent Tasks
    
    loop Audio Streaming
        Twilio->>FastAPI: Media (base64 audio)
        FastAPI->>Deepgram: Forward Audio
        Deepgram->>FastAPI: Transcription
        FastAPI->>Database: Log Conversation
        
        Deepgram->>FastAPI: AI Response Audio
        FastAPI->>Twilio: Media (base64 audio)
    end
    
    alt Function Call Required
        Deepgram->>FastAPI: FunctionCallRequest
        FastAPI->>Middleware: Execute Function (e.g., lookup_loan)
        Middleware-->>FastAPI: Function Result
        FastAPI->>Database: Store Call Data
        FastAPI->>Deepgram: FunctionCallResponse
        Deepgram->>FastAPI: Updated AI Response
    end
    
    alt Transfer Call
        Deepgram->>FastAPI: transfer_call function
        FastAPI->>Deepgram: Inject farewell message
        Deepgram-->>FastAPI: AgentAudioDone
        FastAPI->>Twilio: Transfer Call (REST API)
        Twilio->>Twilio: Transfer to Agent
    end
    
    alt Hangup
        Deepgram->>FastAPI: hangup_call function
        FastAPI->>Deepgram: Inject farewell message
        Deepgram-->>FastAPI: AgentAudioDone
        FastAPI->>Twilio: Hangup Call (REST API)
    end
    
    Twilio->>FastAPI: POST /call-status (completed)
    FastAPI->>Database: Update CDR
    FastAPI->>Database: Generate Call Summary
```

## Component Responsibilities

### 1. Twilio Platform
- **PSTN Connectivity**: Routes phone calls from traditional phone networks
- **Media Streams**: Provides real-time audio streaming via WebSocket
- **Call Control**: REST API for hangup, transfer, DTMF operations
- **Webhooks**: Status updates, recording callbacks

### 2. FastAPI Application Server

#### WebSocket Handler
- **Concurrent Task Management**: Runs 6 async tasks simultaneously
  - `twilio_receiver`: Decodes incoming audio from Twilio
  - `twilio_sender`: Encodes and sends audio back to Twilio
  - `dg_sender`: Forwards audio to Deepgram
  - `dg_receiver`: Processes Deepgram responses
  - `silence_watchdog`: Monitors for silence (25s timeout)
  - `hangup_watchdog`: Auto-hangup after extended silence (60s)

#### Business Logic
- **Function Handler**: Executes business functions called by AI
- **Call Control**: Manages call operations (hangup, transfer, DTMF)
- **i18n Support**: Multi-language prompts and responses

#### Server Pool Manager
- **Load Balancing**: Distributes calls across stream servers
- **Health Monitoring**: Tracks server availability and performance

### 3. Deepgram Platform
- **Speech Recognition**: Real-time transcription of caller speech
- **Voice AI**: Generates contextual responses
- **Function Calling**: Requests business logic execution
- **Text-to-Speech**: Converts AI responses to natural speech

### 4. PostgreSQL Database

#### Tables
- **CDR (Call Detail Records)**: Call metadata, duration, status
- **Conversation Log**: Full transcripts and event history
- **Function Definitions**: Business function schemas
- **Company Config**: Per-company settings and prompts

### 5. External Middleware
- **Repay Middleware**: Loan lookups, payment processing
- **Bank APIs**: ABA routing number validation

### 6. Logging System
- **Conversation Logger**: Structured event logging
- **CDR Logger**: Call metrics and analytics
- **File Logger**: Audit trail for prompts and configurations

## Key Features

### Real-Time Processing
- Bidirectional audio streaming with minimal latency
- Concurrent task execution for optimal performance
- Async/await pattern for non-blocking operations

### Intelligent Call Handling
- Silence detection with proactive prompts
- Automatic hangup after extended inactivity
- Context-aware AI responses

### Business Integration
- Dynamic function calling based on conversation
- Secure middleware integration
- Database-backed configuration

### Monitoring & Analytics
- Comprehensive conversation logging
- Call detail records for analytics
- Audit trails for compliance

### Scalability
- Server pool management for load distribution
- Stateless design for horizontal scaling
- Health monitoring and failover

## Audio Flow Details

### Twilio → Deepgram
1. Twilio sends mulaw audio (8kHz, base64-encoded)
2. FastAPI decodes and buffers audio chunks
3. Audio forwarded to Deepgram via WebSocket
4. Deepgram transcribes in real-time

### Deepgram → Twilio
1. Deepgram generates AI response
2. Deepgram synthesizes speech (mulaw format)
3. FastAPI receives binary audio
4. Audio base64-encoded and sent to Twilio
5. Twilio plays audio to caller

## Security Considerations

- **API Authentication**: Deepgram API keys, Twilio auth tokens
- **Request Validation**: Twilio webhook signature verification
- **Sensitive Data**: SSN, loan numbers handled securely
- **Environment Variables**: Credentials stored in env vars
- **Secure WebSockets**: WSS protocol for encrypted communication

## Deployment Architecture

```mermaid
graph TB
    subgraph Production["Production Environment"]
        LB[Load Balancer<br/>Traefik]
        
        subgraph Containers["Docker Containers"]
            App1[FastAPI Instance 1]
            App2[FastAPI Instance 2]
            App3[FastAPI Instance 3]
        end
        
        DB[(PostgreSQL<br/>External)]
    end
    
    Internet[Internet] -->|HTTPS/WSS| LB
    LB -->|Route| App1
    LB -->|Route| App2
    LB -->|Route| App3
    
    App1 -->|Query| DB
    App2 -->|Query| DB
    App3 -->|Query| DB
    
    classDef containerStyle fill:#4A90E2,stroke:#333,stroke-width:2px,color:#fff
    classDef dbStyle fill:#F5A623,stroke:#333,stroke-width:2px,color:#000
    
    class App1,App2,App3 containerStyle
    class DB dbStyle
```

## Technology Stack Summary

| Component | Technology |
|-----------|------------|
| **Backend Framework** | FastAPI (Python) |
| **Telephony** | Twilio Voice API |
| **AI/Speech** | Deepgram Voice AI |
| **Database** | PostgreSQL |
| **WebSocket** | Native Python websockets |
| **Load Balancer** | Traefik |
| **Deployment** | Docker + Docker Compose |
| **Middleware** | Repay API |
| **i18n** | Custom JSON-based system |

## Performance Characteristics

- **Latency**: 100-500ms total (network + processing)
- **Audio Quality**: 8kHz telephony (Twilio) → Deepgram processing
- **Concurrent Calls**: Scales horizontally with server pool
- **Silence Detection**: 25 seconds before prompt
- **Auto Hangup**: 60 seconds of total silence
- **Audio Chunk Size**: ~20ms per chunk

## Related Documentation

- [Twilio + Deepgram Stack Overview](twilio-deepgram-stack.md)
- Technical Design Document (sts-twilio/docs/technical_design.md)
- WebSocket Flow Documentation (sts-twilio/docs/websocket_endpoint_flow.md)
- API Reference (sts-twilio/docs/API_REFERENCE.md)
