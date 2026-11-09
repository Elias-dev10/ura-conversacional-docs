# Fluxo da Ligação (voz)

## Visão de alto nível

```mermaid
sequenceDiagram
    autonumber
    participant C as Chamador
    participant FS as FreeSWITCH
    participant MB as agent-tegma (STT/LLM/TTS)
    participant API as FastAPI (backend)
    participant CV as Cervello (RE)
    participant TW as Twilio (SMS)

    C->>FS: Liga para o DID (PSTN → SIP)
    FS->>MB: Áudio em tempo real (WebSocket L16, loopback)
    MB->>C: Saudação (TTS) — "Como posso ajudar?"
    C->>MB: Fala o pedido (reset de senha)
    MB->>MB: STT + LLM — decide seguir o fluxo (interno/terceirizado)
    MB->>MB: LLM escolhe executar_reset (daqui em diante, código determinístico)
    MB->>C: "Digite a matrícula" (coleta DTMF via ESL)
    C-->>MB: Matrícula (teclado)
    MB->>API: verificar_matricula (matrícula, ANI, call_uuid)
    API->>CV: Verifica RE (HTTPS, Basic auth)
    CV-->>API: Nome + e-mail (campo "Login")
    MB->>C: "Digite o celular" + confirmação (DTMF)
    C-->>MB: Celular (teclado) → confirma
    MB->>API: enviar_sms (mesmo call_uuid)
    API->>TW: Envia link SSPR
    TW-->>C: SMS com o link oficial Microsoft
    MB->>C: "Link enviado. Precisa de mais algo?" (TTS)
```

---

## Etapas e controles

| # | Etapa | Controle de segurança |
|---|---|---|
| 1 | Entrada da chamada | Firewall + ACL SIP: só as faixas da operadora chegam ao FreeSWITCH |
| 2 | Mídia para o agente | WebSocket **em loopback**; nenhuma porta de mídia exposta |
| 3 | Conversa (STT/LLM/TTS) | A IA só recebe **fala**; escolhe ações de uma **lista fechada**; guardrails anti-jailbreak |
| 4 | Coleta de matrícula | **DTMF**, não voz; validada no backend; rate limit por matrícula+ANI |
| 5 | Verificação de identidade | Cervello via **HTTPS + Basic auth**; mensagem genérica em erro (anti-enumeração) |
| 6 | Coleta/confirmação do celular | **DTMF**; celular BR de 11 dígitos; confirmação dos últimos dígitos; nunca aceita número por voz |
| 7 | Envio do link | **Só o link** do portal Microsoft SSPR (SEC-01), nunca a senha; só para a matrícula **verificada na mesma ligação**; **1 SMS por ligação** + limite por matrícula/ANI; registrado na auditoria |
| 8 | Encerramento/fim | Resposta de sucesso **determinística** no servidor (não deixada à IA) |

---

## Saídas alternativas (contingência)

```mermaid
flowchart TB
    START([Em atendimento]) --> Q{Situação}
    Q -- "Terceirizado" --> T[Transfere ao plantão]
    Q -- "Matrícula não encontrada / bloqueio / falha" --> T
    Q -- "Insiste fora do escopo (3x)" --> T
    Q -- "Cancelar / concluído" --> H[Encerra educadamente]
    T --> BR["bridge via tronco Datora →\nAnalista (11) 2091-6450"]
```

A transferência é feita por **bridge** pelo tronco SIP de saída: o chamador permanece na mesma
ligação ouvindo o toque até o atendente atender. Limites anti-abuso (teto de turnos e de
duração da ligação) também levam, no limite, à transferência segura.
