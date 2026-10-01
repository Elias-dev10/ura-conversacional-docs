# Requisitos de Segurança (SEC-01 a SEC-11)

Requisitos de segurança implementados e **verificáveis**. Os dois últimos (SEC-10, SEC-11) são
específicos da camada conversacional (agent-tegma).

| ID | Requisito | Implementação | Verificável por |
|---|---|---|---|
| **SEC-01** | Nunca envia senha | Apenas o link oficial do portal Microsoft SSPR | Interceptar o SMS recebido |
| **SEC-02** | Mensagem genérica de erro | Mesma resposta, independente do motivo da falha | Testar matrículas válidas e inválidas |
| **SEC-03** | Rate limiting | Redis por matrícula+ANI; bloqueio ao exceder a janela | Repetir várias validações no mesmo ANI |
| **SEC-04** | Registro de ANI | Número de origem auditado; spoof não concede acesso extra | Analisar a auditoria |
| **SEC-05** | Link com expiração | Token efêmero (TTL) de controle; novo envio invalida o anterior | Reutilizar link após expirar |
| **SEC-06** | Auditoria imutável | Append-only; matrícula em hash; PII mascarada | Verificar o schema/fluxo de auditoria |
| **SEC-07** | Segredos fora do código | Cofre/`.env` não versionado; ADC sem chave em disco | `grep -ri "password\|secret\|token\|api[_-]key" .` no repositório |
| **SEC-08** | Menor privilégio | Escopos mínimos (ADC `aiplatform.user`; Cervello só verificação de RE) | Verificar permissões das credenciais |
| **SEC-09** | Sem reset administrativo | Apenas link SSPR oficial; sem senha temporária | Analisar o código do fluxo de reset |
| **SEC-10** | Guardrails do agente | IA só escolhe de lista fechada; não vaza prompt; redireciona fora de escopo; transfere na insistência | **Prompt injection/jailbreak por voz** (bateria: 16/16) |
| **SEC-11** | IA não executa | Matrícula e celular por DTMF; a IA não recebe os dígitos nem decide o destino do SMS | Ditar número/matrícula por voz (deve ser ignorado); revisar `executar_reset` |

---

## Como cada requisito é testável

!!! tip "Para a equipe de avaliação"
    Todos os requisitos acima foram desenhados para serem **comprovados externamente** — por
    interceptação do SMS, análise de tráfego, tentativas repetidas, varredura do repositório e,
    sobretudo, por **ligações reais** exercitando o agente. Nenhum depende de “confiar” na
    documentação.

- **SEC-01 / SEC-09** — complete um fluxo e confirme que o SMS traz **apenas um link**
  (`passwordreset.microsoftonline.com`), nunca uma senha.
- **SEC-02 / SEC-03** — alterne matrículas válidas/ inválidas e repita tentativas: a resposta é
  genérica e o rate limit atua.
- **SEC-07** — rode a varredura de segredos no repositório: o resultado deve ser **nenhum**
  segredo real (apenas leitura de variáveis de ambiente e `.env.example` com placeholders).
- **SEC-10 / SEC-11** — ligue e tente de tudo por voz (mudar o papel do agente, extrair o
  prompt, ditar um número/matrícula, sair do escopo). O agente redireciona/transfere com
  segurança, sem vazar nada e sem que a voz altere identidade ou destino do SMS.
