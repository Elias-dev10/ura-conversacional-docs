# URA Conversacional (agent-tegma) — Documentação de Segurança

Site MkDocs (Material + Mermaid) com a topologia, a arquitetura e as camadas de
segurança da **URA conversacional por voz** de reset de senha, no mesmo padrão
entregue antes para a URA DTMF. Destinado à equipe de segurança do cliente para
validar o layout, as camadas de defesa e conduzir testes autorizados.

> Não contém credenciais. IP público e portas são incluídos propositalmente para o scan.

## Pré-visualizar localmente

```bash
python -m venv .venv && . .venv/Scripts/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
mkdocs serve        # http://127.0.0.1:8000
```

## Publicar no GitHub Pages

O mesmo fluxo do site anterior (`ura-service-desk-docs`). Em um repositório próprio
(ex.: `ura-conversacional-docs`) da conta `Elias-dev10`:

```bash
pip install -r requirements.txt
mkdocs gh-deploy --force      # publica em https://<conta>.github.io/<repo>/
```

`gh-deploy` constrói o site e envia para o branch `gh-pages`, que o GitHub Pages serve.
