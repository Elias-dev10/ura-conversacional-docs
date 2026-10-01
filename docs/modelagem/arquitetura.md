# Arquitetura e Topologia

## Diagrama de Arquitetura Completo

```mermaid
flowchart TB
    subgraph EXT[" Externo "]
        CALLER["Chamador\nPSTN / Celular"]
        DATORA["Datora SIP\nentrada + saída\n187.60.60.161:5060\nDID +55 11 4501-1057"]
        GOOGLE["Google Cloud APIs\nSpeech-to-Text · Text-to-Speech\nVertex AI (Gemini) — ADC"]
        CERVELLO["Cervello — API Macro\nverificação de RE\nBasic auth (HTTPS)"]
        TWILIO["Twilio\nSMS"]
        MS["Portal SSPR\nMicrosoft\npasswordreset.microsoftonline.com"]
        ATEND["Analista de Plantão\n(11) 2091-6450"]
    end

    subgraph GCP[" GCP — projeto erika-conversacional-homol "]
        FW{{"Firewall VPC erika-conv-net\n(apenas CIDRs autorizados)"}}
        IAP["IAP Google\nSSH admin\n35.235.240.0/20"]

        subgraph VM[" VM erika-conv-vm — 34.95.240.153 (interno 10.60.0.2) "]
            FS["FreeSWITCH\nmod_audio_stream + mod_sofia\nSIP 5060/5080 + RTP"]
            MB["mediabridge — agent-tegma\nWebSocket :8090 (áudio L16)\nSTT · LLM · TTS · ESL\n127.0.0.1 apenas"]
            API["FastAPI — executor das ações\n127.0.0.1:8010\nloopback apenas"]
            REDIS[("Redis 7\n127.0.0.1:6380\nrate-limit + sessão")]
            PG[("PostgreSQL 16\nauditoria append-only")]
        end

        SM["Secret Manager\n(cofre de credenciais)"]
        LOG["Cloud Logging\n+ Monitoring"]
    end

    ADMIN["Operador Admin"] -- "SSH via IAP\n(autenticação Google 2FA)" --> IAP --> FW

    CALLER --> DATORA
    DATORA -- "SIP — só CIDR Datora\n187.60.60.0/24" --> FW --> FS

    FS -- "áudio em tempo real\nWebSocket L16 (loopback)" --> MB
    MB -- "playback TTS (ESL)" --> FS
    MB -- "HTTPS (ADC)\nfala ⇄ STT/TTS/LLM" --> GOOGLE

    MB -- "HTTP loopback\nexecutar_reset (DTMF)" --> API
    API --> REDIS
    API --> PG
    API -- "HTTPS — Basic auth\nverifica RE" --> CERVELLO
    API -- "HTTPS" --> TWILIO -- "SMS link SSPR" --> CALLER
    TWILIO -. "link redireciona para" .-> MS

    FS -- "bridge B-leg\nSIP Datora saída" --> DATORA --> ATEND

    SM -- "secrets em runtime" --> API
    SM -- "secrets em runtime" --> MB
    VM --> LOG
```

!!! note "A regra de ouro desta arquitetura"
    A IA (**agent-tegma**) recebe **apenas a fala** do chamador e escolhe ações de uma
    **lista fechada**. A **matrícula e o celular** chegam por **DTMF (teclado)** e são
    validados pelo FastAPI — a IA **nunca** vê os dígitos nem decide identidade ou destino do
    SMS. Logo, não há caminho para *prompt injection* redirecionar o reset.

---

## Topologia de rede

```mermaid
flowchart LR
    subgraph INTERNET["Internet"]
        direction TB
        CALLER2["Chamador\nIP variável"]
        DATORA2["Datora\n187.60.60.0/24"]
        SCAN["Scanner externo\nIP qualquer"]
    end

    subgraph GCP_FW["Firewall GCP — erika-conv-net"]
        direction TB
        R1["ALLOW: tcp/udp 5060,5080\norigem: Datora"]
        R2["ALLOW: udp 16384-32768\norigem: Datora (RTP)"]
        R3["ALLOW: tcp 22\norigem: 35.235.240.0/20 (IAP)"]
        R4["DENY: todo o resto\n(default-allow-ssh/rdp removidas)"]
    end

    subgraph VM2["VM erika-conv-vm 34.95.240.153"]
        direction TB
        FS2["FreeSWITCH\n0.0.0.0:5060/5080"]
        MB2["mediabridge (agent-tegma)\n127.0.0.1:8090 (WS)"]
        API2["FastAPI\n127.0.0.1:8010"]
        REDIS2["Redis\n127.0.0.1:6380"]
        ESL2["ESL\n127.0.0.1:8021"]
    end

    DATORA2 --> R1 --> FS2
    CALLER2 --> R2 --> FS2
    SCAN -- "qualquer porta TCP" --> R4
    R4 -- "BLOQUEADO" --> VM2
```

---

## Componentes internos

| Componente | Função | Escuta em | Acessível externamente? |
|---|---|---|---|
| **FreeSWITCH** | Telefonia — atende SIP, mídia RTP, `mod_audio_stream`, bridge | `0.0.0.0:5060/5080` (SIP) | Apenas da faixa Datora |
| **mediabridge (agent-tegma)** | Ponte de mídia + agente: STT, LLM, TTS, ESL; WebSocket de áudio | `127.0.0.1:8090` (WS) · ESL `127.0.0.1:8021` | **Não** |
| **FastAPI** | Executor das ações — valida RE, rate limit, envia o link | `127.0.0.1:8010` | **Não** |
| **Redis** | Rate limiting + estado efêmero | `127.0.0.1:6380` | **Não** |
| **PostgreSQL** | Auditoria imutável | interno | **Não** |

---

## Separação de responsabilidades

O design segue a **separação estrita** — reforçada pela regra de ouro da camada de IA:

- **FreeSWITCH** atende a ligação e transporta a mídia — **não toma decisões**.
- **mediabridge / agent-tegma** conduz a conversa (STT→LLM→TTS) e **propõe** ações de uma lista
  fechada — **não executa** nem recebe os dígitos sensíveis.
- **FastAPI** toma e executa as decisões (valida RE, coleta DTMF, envia o link) — **nunca exposto**.
- **Redis** guarda estado temporário com TTL — **sem PII persistida**.
- **PostgreSQL** registra auditoria imutável — **matrícula apenas em hash**.

```mermaid
flowchart LR
    FS3["FreeSWITCH\nTransporte de mídia"] <-- "WS L16 (loopback)" --> MB3["agent-tegma\nConversa / propõe"]
    MB3 -- "HTTP loopback\ncontrato de ações" --> API3["FastAPI\nValida / executa"]
    API3 --> R3["Redis\nEstado efêmero"]
    API3 --> PG3["PostgreSQL\nAuditoria imutável"]
```

Mesmo que o FreeSWITCH **ou** o agente de IA sejam manipulados, o atacante não alcança a
lógica de negócio, o banco ou as credenciais — tudo fica no loopback interno, e identidade e
destino do SMS só se resolvem por DTMF validado no FastAPI.
