# Validação de Identidade

## Como a identidade é verificada

A identidade do chamador é um **fator**: a **matrícula (RE)**, digitada por **DTMF** e validada
na base de identidade do cliente através da **API Macro da Cervello**.

```mermaid
flowchart TB
    MAT["Matrícula digitada\n(DTMF — nunca por voz)"] --> API["FastAPI\nverificar_matricula"]
    API --> RL{Rate limit\n(matrícula + ANI)}
    RL -- "excedeu" --> BLK["Bloqueia / transfere"]
    RL -- "ok" --> CV["Cervello — API Macro\nPOST ExecutaMacroFuncao\nHTTPS + Basic auth"]
    CV --> R{Encontrado?}
    R -- "sim" --> OK["Nome + e-mail (campo Login)\n→ monta link SSPR"]
    R -- "não / erro" --> GEN["Mensagem GENÉRICA\n(anti-enumeração — SEC-02)\n→ nova tentativa / transfere"]
```

- A matrícula **nunca** é falada ao agente; entra por teclado e é tratada pelo backend.
- A verificação usa **HTTPS com Basic auth**; a credencial vive só em cofre/ambiente.
- O **e-mail** usado para montar o link do portal Microsoft vem do campo **`Login`** retornado
  pela Cervello.
- Em qualquer falha (não existe, erro técnico, bloqueio), a URA responde de forma **genérica** —
  não é possível distinguir “matrícula inexistente” de “erro”, o que impede enumeração.

---

## Modelo de 1 fator e o mitigante

A URA entrega o **link do portal Microsoft SSPR** por SMS ao celular digitado. O **reset da
senha em si** só se conclui **no portal da Microsoft**, que impõe a **própria verificação**
(fatores do usuário).

!!! info "Receber o link ≠ redefinir a senha"
    Mesmo que alguém com uma matrícula receba o link, **não consegue** concluir o reset sem
    passar pela verificação da Microsoft. Esse é o mitigante central do modelo de 1 fator.
    Recomendação para evolução: 2º fator conferindo os **últimos dígitos do celular cadastrado**
    antes do envio, e não falar o nome completo na confirmação.

---

## Controles associados

| Controle | Implementação |
|---|---|
| Entrada por teclado | Matrícula e celular só por DTMF; a IA não recebe os dígitos |
| Rate limiting | Redis por **matrícula + ANI**; bloqueio após tentativas em janela |
| Anti-enumeração | Mensagem genérica de erro (SEC-02) |
| Credenciais | Basic auth da Cervello em cofre/`.env`; ADC (Google) sem chave em disco |
| Nunca a senha | Apenas o link oficial Microsoft (SEC-01) |
