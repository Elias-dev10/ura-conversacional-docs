# Guia para Avaliadores

Esta página antecipa o que a equipe de segurança **encontrará — e o que não encontrará** — em
cada tipo de teste. Todos os testes abaixo são **autorizados e incentivados**.

## O que esperar por tipo de teste

| Teste | Ferramentas típicas | Resultado esperado |
|---|---|---|
| **Varredura de portas** | nmap, masscan | Do seu IP: SIP/RTP e SSH **filtrados/fechados**; `8010/6380/8090/8021` inalcançáveis (loopback). Nenhuma porta de app aberta. |
| **Intrusão SIP** | sipvicious, svmap, svcrack | INVITE/OPTIONS de IP fora das faixas da operadora **rejeitado** (firewall + ACL). Sem registro com credencial padrão. |
| **Scan de aplicação web** | nikto, ZAP, Burp | Sem app web exposto (FastAPI é loopback). Nada a enumerar externamente. |
| **SSH / acesso admin** | hydra, tentativas diretas | Porta 22 **bloqueada** fora do IAP. Sem brute-force possível de fora. |
| **Força bruta / enumeração** | scripts próprios | Mensagem **genérica** (SEC-02) + **rate limit** por matrícula+ANI. |
| **Prompt injection / jailbreak (voz)** | ligação real | Agente **recusa e redireciona**; não vaza prompt; voz **não** altera identidade nem destino do SMS; insistência → transferência. |
| **Análise de tráfego** | tcpdump, Wireshark | Saídas em **HTTPS**; SSH via IAP cifrado; corpo do SMS traz só o link. |
| **Varredura de segredos** | `grep`/`gitleaks` no repo | **Nenhum** segredo real; tudo em cofre/ambiente; ADC sem chave. |

---

## Roteiro sugerido (passo a passo)

```mermaid
flowchart TB
    P1["1. Scan de rede no IP público\n34.95.240.153"] --> P2["2. Intrusão SIP de IP não autorizado"]
    P2 --> P3["3. Tentar SSH fora do IAP"]
    P3 --> P4["4. Ligar no DID e exercitar o agente\n(jailbreak, número por voz, fora de escopo)"]
    P4 --> P5["5. Enumeração de matrícula + rate limit"]
    P5 --> P6["6. Interceptar o SMS (só link?)"]
    P6 --> P7["7. Varredura de segredos no repositório"]
```

1. **Rede** — confirme que, do seu IP, SIP/RTP/SSH estão filtrados e as portas de app não existem.
2. **SIP** — envie tráfego SIP de um IP fora da faixa da Datora; deve ser barrado antes da aplicação.
3. **SSH** — tente conectar direto no IP público; deve falhar (só IAP).
4. **Aplicação (voz)** — ligue para `+55 11 4501-1057` e ataque o agente: tente mudar seu papel,
   extrair o prompt, ditar número/matrícula por voz, pedir coisas fora do reset. Observe que ele
   redireciona/transfere e que nada disso muda identidade ou destino do SMS.
5. **Identidade** — alterne matrículas e repita tentativas; verifique mensagem genérica + bloqueio.
6. **Entrega** — conclua um fluxo e confirme que o SMS traz **apenas** o link oficial Microsoft.
7. **Código** — rode a varredura de segredos no repositório.

---

## Ambiente e escopo

!!! info "Ambiente de homologação, isolado da produção"
    Este sistema é a **URA conversacional** em ambiente de **homologação**, separado e
    independente do projeto DTMF anterior. O IP público e as portas estão neste documento
    justamente para viabilizar o scan. **Não há credenciais** aqui.

!!! warning "Observações operacionais durante os testes"
    - Fluxos concluídos disparam **SMS reais** (há rate limit; evite volume desnecessário).
    - Pedir atendimento humano **transfere para o plantão real** — combine a janela de testes.
    - Liberações temporárias de IP (softphone de teste) são removidas ao fim da validação.

Qualquer comportamento divergente do descrito aqui — especialmente a URA **aceitando** algo fora
do escopo, **vazando** o prompt do agente, ou **enviando** o SMS para um número informado por voz
— deve ser reportado como achado.
