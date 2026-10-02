# Figure 10.6

## Caption

End-to-end request flow on Cloud Run. Environment variables wire Service A and Service B to their respective Cloud Run and Vertex AI RAG endpoints at deployment time, ensuring portable, reproducible infrastructure.

## Mermaid

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TD
    ENV["<div style='min-width: 640px;'><b>Deployment Configuration Source</b><br/>Environment Variables (<code>.env</code>) or <code>gcloud run deploy</code> Flags</div>"]

    subgraph SVA ["Service A Environment (policy-retrieval)"]
        direction TB
        A1["<div style='min-width: 270px;'><b>RAG_CORPUS</b><br/>Vertex AI corpus resource name</div>"]
        A2["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_PROJECT</b><br/>GCP project identifier</div>"]
        A3["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_LOCATION</b><br/>GCP deployment region</div>"]
        A1 --- A2 --- A3
    end

    subgraph SVB ["Service B Environment (policy-agent)"]
        direction TB
        B1["<div style='min-width: 270px;'><b>RETRIEVAL_SERVICE_URL</b><br/>Service A Cloud Run endpoint</div>"]
        B2["<div style='min-width: 270px;'><b>AGENT_MODEL</b><br/>Gemini model identifier</div>"]
        B3["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_PROJECT</b><br/>GCP project identifier</div>"]
        B4["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_LOCATION</b><br/>GCP deployment region</div>"]
        B1 --- B2 --- B3 --- B4
    end

    ENV ==> SVA
    ENV ==> SVB

    classDef env fill:#F3F4F6,stroke:#4B5563,color:#000000,stroke-width:1.5px
    classDef sva fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef svb fill:#EFF6FF,stroke:#2563EB,color:#000000,stroke-width:1.5px
    classDef subg_a fill:#FFFFFF,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef subg_b fill:#FFFFFF,stroke:#2563EB,color:#000000,stroke-width:1.5px

    class ENV env
    class SVA subg_a
    class SVB subg_b
    class A1,A2,A3 sva
    class B1,B2,B3,B4 svb
```
