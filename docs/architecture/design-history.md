# Histórico de design — diagramas MCP Audio

Log das iterações (perguntas, respostas, correções de premissa e versões) usadas para construir os diagramas de arquitetura. Este arquivo **não** substitui os diagramas; aponta para os artefatos finais.

## Artefatos atuais

| Artefato | Tipo | Arquivo |
|----------|------|---------|
| C4 Containers | Estrutural | [c4-containers-mcp-audio.md](c4-containers-mcp-audio.md) |
| Sequência `search_audio` | Comportamental | [sequence-search-audio.md](sequence-search-audio.md) |

---

## Linha do tempo / versões

Sessão de construção: **2026-09-26** (ordem lógica das iterações).

| Versão | Artefato | Mudança |
|--------|----------|---------|
| C4-0 | Roteiro | Lacunas e perguntas de esclarecimento (antes de qualquer Mermaid). |
| C4-1 | Roteiro + C4 | Respostas do stakeholder; roteiro Tabela 2 fechado; primeira entrega `C4Container` com descrições longas + lacunas residuais fechadas. |
| C4-1a | C4 (abortado) | Tentativa de “corrigir” renderização no Cursor via `flowchart`; limpeza/revert a pedido — voltar ao `C4Container` original. |
| C4-1b | Premissa | Stakeholder: `C4Container` **é** visível no GitHub; premissa contrária descartada. |
| C4-2 | C4 | Mesmo `C4Container`, caixas e `Rel()` encurtados; nota apontando detalhes à tabela; removido `UpdateRelStyle`. |
| SEQ-0 | Roteiro | Pedido de sequência; conflito template pedido/pagamento vs domínio áudio; só lacunas/perguntas. |
| SEQ-1 | Premissas | Respostas: MCP Audio, só `search_audio`, timeout Qdrant, Host+MCP+Qdrant, `tool_call_id`, retry do agente, filtros autor/texto/áudio/tag/top-k. |
| SEQ-1b | Notação | PlantUML descartado; notação = **Mermaid** `sequenceDiagram`. |
| SEQ-2 | Sequência | Entrega completa: lacunas fechadas, roteiro, Mermaid, checklist de PR. |
| HIST-1 | Este log | Documentação do histórico de versões e iterações. |

---

## A) C4 Containers — iterações

### A.1 Lacunas e perguntas (C4-0)

Antes do diagrama, o arquiteto pediu esclarecimentos sobre:

- Motor vetorial e locus dos áudios
- Ingestão (quem gera ASR / IPA / embeddings)
- Semântica de busca e filtros
- Backup (escopo, frequência, integridade, dono)
- Imutabilidade de áudio (só tools MCP vs ACL de disco)
- Superfície MCP (tools)
- Efeito de update de texto/áudio no embedding

### A.2 Respostas do stakeholder (entrada de C4-1)

Decisões que fecharam o roteiro estrutural:

- **Qdrant** local; áudios copiados para `db/`; DB guarda **path**
- Embeddings: **HuBERT** (áudio) + modelo de texto (transcrição); fonemas **IPA** no metadado
- Busca por embedding de texto e/ou áudio; filtro por **autor** (e depois tags no desenho de search)
- Tools canônicas: `add_audio`, `search_audio`, `update_transcription`, `update_phonemes`, `backup_now`
- Nunca excluir/modificar áudio **via tools MCP**; alterar transcrição/fonema ok; reindex quando texto (ou áudio) muda
- Backup: tudo (áudios + metadados/Qdrant), semanal + `backup_now`, integridade SHA-256, container **Backup Job** no sistema MCP

### A.3 v1 — diagrama denso (C4-1)

Primeira versão em Mermaid `C4Container` com descrições longas dentro das caixas (lista de tools, payloads, restrições) e tabela de notas + decisões residuais (destino `backup/YYYY-MM-DD/`, RRF, named vectors `audio`/`text`, adapters plugáveis).

### A.4 Desvio e revert (C4-1a)

Erro de render no preview do Cursor levou a experimentar `flowchart` e tooling de validação. O stakeholder pediu **limpar a bagunça** e restaurar o arquivo **como estava** (`C4Container`), para seguir noutro editor.

### A.5 Premissa GitHub (C4-1b)

Pedido de melhor legibilidade gerou plano inicial convertendo para `flowchart` “por causa do GitHub”. O stakeholder corrigiu: **`C4Container` é visível no GitHub**. O plano foi atualizado: **manter C4Container**, só encurtar texto.

### A.6 v2 — legibilidade (C4-2) — estado atual

- Mantido `C4Container`
- 4º parâmetro dos containers e rótulos `Rel()` curtos
- Removido `UpdateRelStyle`
- Nota: detalhes na tabela “Notas do desenho”
- Semântica inalterada (Host, MCP Server, Processing, Qdrant, Audio Storage, Backup Job)

Ver artefato: [c4-containers-mcp-audio.md](c4-containers-mcp-audio.md).

---

## B) Sequência `search_audio` — iterações

### B.1 Pedido e conflito de template (SEQ-0)

Pedido de diagrama **comportamental** (sequência) com descrição em linguagem natural sobre listar áudios / alterar transcrição ou fonema. O enunciado também trazia escopo residual de **pedido/pagamento** e “timeout do provedor” + idempotência em retries.

Arquiteto **não** gerou o diagrama ainda: listou lacunas e perguntou o domínio real.

### B.2 Respostas (SEQ-1)

| Pergunta | Resposta |
|----------|----------|
| Domínio | MCP Audio (pagamento foi erro de template) |
| Cenário | Único: `search_audio` |
| Timeout | Qdrant |
| Participantes | MCP Server + Qdrant (+ Agente/Host no retry) |
| Idempotência | `tool_call_id` — dedupe de **embed** e de **search** |
| Retry | Agente tenta de novo com o mesmo id |
| Filtros | autor, texto, áudio, tag, top-k |

### B.3 Notação (SEQ-1b)

Plano inicial citava PlantUML; stakeholder: **vai ser Mermaid**, não PlantUML.

### B.4 Entrega (SEQ-2) — estado atual

Arquivo com, nesta ordem:

1. Lacunas fechadas  
2. Roteiro revisado (sucesso + timeout Qdrant + retry)  
3. Mermaid `sequenceDiagram` com notas de idempotência  
4. Checklist de PR (ordem, falhas parciais, cache, observabilidade mínima)

Ver artefato: [sequence-search-audio.md](sequence-search-audio.md).

---

## Decisões permanentes (até nova revisão)

- Nenhuma tool MCP **exclui ou sobrescreve** arquivo de áudio.
- Qdrant: named vectors `audio` + `text`; busca `both` com **RRF**; payload com path, autor, transcrição, IPA, tags.
- Áudio em `db/`; DB/Qdrant guardam path, não o binário na resposta de search.
- Backup Job: espelho completo + `MANIFEST.sha256`; semanal + `backup_now`.
- Update de transcrição ⇒ reindex do embedding de texto; áudio não mutável via agente.
- Em `search_audio`, idempotência por **`tool_call_id`**: evita double-embed e double-search após sucesso (incl. timeout ambíguo); timeout **não** marca sucesso no store.

---

## Como evoluir este log

Ao mudar um diagrama, acrescente uma linha na tabela de versões e um parágrafo curto na seção A ou B, com o **porquê** da mudança (não só o diff).
