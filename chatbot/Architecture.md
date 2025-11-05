```mermaid
flowchart TD

%% =========================
%% TOP-LEVEL LAYOUT
%% =========================
    U[User] -->|browser| ST[Streamlit UI<br/>app.py]

%% =========================
%% FRONTEND (STREAMLIT)
%% =========================
    subgraph FE[Frontend (Streamlit)]
    direction TB
        ST -->|apply_styling()| CSS[Custom UI / CSS<br/>frontend/styling.py + background.gif]

        %% Chat + Agent
        ST --> CHI[st.chat_input()<br/>messages state]
        ST -->|init| PVAR[Session State<br/>tools + prompt]
        PVAR -->|prompt()| PR[System Prompt<br/>prompt.py]
        PVAR -->|pdf_tool()| TPDF[Tool Handle: pdf_search<br/>tools.py]
        PVAR -->|web_tool()| TWEB[Tool Handle: web_data_search<br/>tools.py]
        PVAR -->|google_tool()| TGOOG[Tool Handle: google_search<br/>tools.py]

        %% File + URL intake (sidebar)
        ST -->|upload PDF| UPDF[st.file_uploader]
        ST -->|enter URL| UURL[st.text_input('URL')]

        %% Outbound calls from Streamlit
        UPDF -->|POST /add_pdf (file)| BE_PDF[/FastAPI: /add_pdf/ (PDF)/]
        UURL -->|POST /scrape_webdata (url)| BE_SCRAPE[/FastAPI: /scrape_webdata/ (Web)/]

        %% Agent creation and run
        CHI -->|invoke()| AGENT[LangChain AgentExecutor<br/>create_tool_calling_agent()]
        AGENT --> LLM[ChatGoogleGenerativeAI<br/>gemini-2.5-flash]

        %% Agent tool calls
        AGENT -- pdf_search --> TPDF
        AGENT -- web_data_search --> TWEB
        AGENT -- google_search --> TGOOG

        %% Tool wrappers make HTTP calls back to API
        TPDF -->|POST /search_query_in_pdf| BE_QPDF[/FastAPI: /search_query_in_pdf/]
        TWEB -->|POST /search_query_in_web| BE_QWEB[/FastAPI: /search_query_in_web/]

        %% Live Google (snippets/metadata only)
        TGOOG --> GAPI[(Google Search APIs)]
    end

%% =========================
%% BACKEND (FASTAPI + ROUTERS)
%% =========================
    subgraph BE[Backend API (FastAPI) : api.py]
    direction TB

        subgraph ROUTES[Routers : api_endpoints/]
        direction TB
            RPDF[pdf_api.py<br/>APIRouter]:::router
            RWEB[web_api.py<br/>APIRouter]:::router
        end

        %% Middleware & instrumentation
        CORS[CORS Middleware]:::infra
        PROM[Prometheus Instrumentator<br/>/metrics]:::infra
        LOG[api.log<br/>logging.basicConfig(...)]:::infra

        %% Pydantic models
        PDM[pydantic_models.py<br/>QueryRequest, WebDataRequest]:::model

        %% Wiring
        BE_PDF ==> RPDF
        BE_QPDF ==> RPDF
        BE_SCRAPE ==> RWEB
        BE_QWEB ==> RWEB

        CORS --- RPDF
        CORS --- RWEB
        PROM --- RPDF
        PROM --- RWEB
        LOG --- RPDF
        LOG --- RWEB
        RPDF --- PDM
        RWEB --- PDM
    end

%% =========================
%% DATA INGEST / RETRIEVAL LAYER
%% =========================
    subgraph DL[Data / Retrieval Layer]
    direction TB

        %% Shared Chroma + Embeddings
        CHC[(ChromaDB Client)]:::store
        EBM[HuggingFaceEmbeddings<br/>all-MiniLM-L6-v2]:::embed

        %% PDF flow
        subgraph PDFPATH[PDF Ingestion]
        direction TB
            UPATH[(uploads/ folder)]:::fs
            PDC[pdf_data_collector.py]:::svc
            PLOAD[PyPDFLoader(mode='page')]:::lib
            PPROC[process_pdf_data()<br/>chunk + embed + add]:::logic
            PCOLL[(Chroma Collection:<br/>pdf_data_collection)]:::store
        end

        %% Web flow
        subgraph WEBPATH[Web Ingestion]
        direction TB
            WDC[web_data_collector.py]:::svc
            WLOAD[WebBaseLoader(url)]:::lib
            WSPLIT[RecursiveCharacterTextSplitter<br/>chunk_size=800, overlap=100]:::lib
            WPROC[process_web_data()<br/>chunk + embed + add]:::logic
            WCOLL[(Chroma Collection:<br/>web_data_collection)]:::store
        end

        %% Search usage paths
        QPDF[Chroma.similarity_search(k=5..10)<br/>(via pdf_api.py)]:::logic
        QWEB[Chroma.similarity_search(k=10)<br/>(via web_api.py)]:::logic
    end

%% =========================
%% FLOWS / EDGES
%% =========================

    %% PDF upload -> ingest
    RPDF ---|/add_pdf writes file| UPATH
    UPATH -->|PDC.add_pdf_data(path)| PDC
    PDC --> PLOAD --> PPROC
    PPROC -->|embed with EBM| EBM
    PPROC -->|add to| PCOLL
    CHC --- PCOLL
    CHC --- WCOLL

    %% Web scrape -> background ingest
    RWEB -->|BackgroundTasks.add_task(add_web_data)| WDC
    WDC --> WLOAD --> WSPLIT --> WPROC
    WPROC -->|embed with EBM| EBM
    WPROC -->|add to| WCOLL

    %% Search endpoints -> retrieval
    RPDF -->|/search_query_in_pdf| QPDF --> PCOLL
    RWEB -->|/search_query_in_web| QWEB --> WCOLL

    %% Tool wrappers (LangChain) -> API
    BE_QPDF --> RPDF
    BE_QWEB --> RWEB

%% =========================
%% WRAPPERS (LangChain Tools)
%% =========================
    subgraph LW[LangChain Tool Wrappers]
    direction TB
        PWRAP[PDFSearchAPIWrapper<br/>pdf_wrapper.py]:::wrap
        WWRAP[WebSearchAPIWrapper<br/>web_wrapper.py]:::wrap
        PWRUN[PDFSearchRun (BaseTool)]:::wrap
        WWRUN[WebSearchRun (BaseTool)]:::wrap
    end

    %% Connections: tools -> wrappers -> API
    TPDF --> PWRUN --> PWRAP -->|HTTP POST| BE_QPDF
    TWEB --> WWRUN --> WWRAP -->|HTTP POST| BE_QWEB

%% =========================
%% MISC / ENV
%% =========================
    subgraph ENV[Config & Dependencies]
    direction TB
        REQ[requirements.txt]:::infra
        GEM[Env var: GEMINI_API_KEY<br/>(validated in app.py)]:::infra
        GSN[langchain-google-community / google api client]:::lib
        LCH[LangChain core/community/chroma]:::lib
        BSOUP[beautifulsoup4, lxml]:::lib
        TORCH[torch, transformers, sentence-transformers]:::lib
    end

    REQ --- FE
    REQ --- BE
    REQ --- DL
    GEM --- LLM
    GSN --- TGOOG
    LCH --- AGENT
    BSOUP --- WLOAD
    TORCH --- EBM

%% =========================
%% RESPONSES / RENDER
%% =========================
    QPDF -->|JSON: content + metadata(page_number)| RPDF
    QWEB -->|JSON: content + metadata(source_url, chunk)| RWEB
    RPDF -->|JSONResponse| TPDF
    RWEB -->|JSONResponse| TWEB
    TPDF --> AGENT
    TWEB --> AGENT
    TGOOG --> AGENT
    AGENT -->|final text answer| ST
    ST -->|render| U

%% =========================
%% STYLES
%% =========================
classDef router fill:#0b7285,stroke:#033,stroke-width:1,color:#fff;
classDef infra fill:#495057,stroke:#222,stroke-width:1,color:#fff;
classDef model fill:#845ef7,stroke:#402,stroke-width:1,color:#fff;
classDef store fill:#2b8a3e,stroke:#062,stroke-width:1,color:#fff;
classDef svc fill:#5c7cfa,stroke:#224,stroke-width:1,color:#fff;
classDef lib fill:#adb5bd,stroke:#444,stroke-width:1,color:#111;
classDef logic fill:#fcc419,stroke:#b08900,stroke-width:1,color:#111;
classDef wrap fill:#e8590c,stroke:#843,stroke-width:1,color:#fff;
classDef fs fill:#51cf66,stroke:#093,stroke-width:1,color:#111;
```