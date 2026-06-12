# TP2-IMLE
# Retail Vision Intelligence System
**LIACD — Trabalho #2**  
António Gomes

Sistema de inspeção visual contínua de prateleiras de supermercado com memória histórica, motor de regras em linguagem natural, e interface conversacional CLI.

---

## Estrutura do Projeto

```
TP2-IMLE/
├── .env                        # GEMINI_API_KEY (não incluído no repositório)
├── .env.example
├── requirements.txt
├── evaluate.py                 # Harness de avaliação
├── evaluation_report.json      # Resultados da avaliação
├── data/
│   ├── ground_truth/           # 21 imagens + ground_truth.json
│   ├── images/                 # Dataset de imagens
│   ├── inspections/            # INS_*.json + report_*.md + rules_log_*.json
│   └── rules/                  # RULE_*.json + rules_default.txt
├── prompts/
│   ├── shelf_inspector_zero_shot.txt
│   ├── shelf_inspector_chain_of_thought.txt
│   └── shelf_inspector_few_shot.txt
├── src/
│   ├── shelf_inspector.py      # Componente 1
│   ├── rule_engine.py          # Componente 2
│   ├── rag_memory.py           # Componente 3
│   ├── report_generator.py     # Componente 4
│   └── interface.py            # Componente 5
├── cache/                      # Cache MD5 das chamadas à API
└── vectorstore/                # ChromaDB persistente
```

---

## Requisitos

- Python 3.13
- Chave de API do Gemini (`gemini-2.5-flash-lite`)

```bash
pip install -r requirements.txt
```

O ficheiro `requirements.txt` inclui: `google-generativeai`, `chromadb`, `sentence-transformers`, `python-dotenv`, `Pillow`.

Configurar a chave de API:
```
# .env
GEMINI_API_KEY=a_tua_chave_aqui
```

---

## Modelo

O enunciado referia o Gemini 1.5 Flash, descontinuado pela Google em 2026. Este projeto usa o `gemini-2.5-flash-lite`, o substituto oficial. O modelo é configurável via variável de ambiente `MODEL` no `.env`.

---

## Componentes

### 1. Shelf Inspector (`src/shelf_inspector.py`)

Analisa imagens de prateleiras com o Gemini e devolve um inspection record JSON.

```bash
# Analisar uma imagem
python src/shelf_inspector.py --image data/ground_truth/012.jpg

# Analisar um directório
python src/shelf_inspector.py --images-dir data/images --save
```

**Gestão de API:**
- Cache local por hash MD5 do ficheiro — imagens já analisadas não consomem quota
- Rate limiter: janela deslizante 15 req/min com backoff exponencial em erro 429
- Fallback gracioso: quota esgotada → modo cache apenas

**Output:**
```json
{
  "inspection_id": "INS_YYYYMMDD_HHMMSS_NNN",
  "timestamp": "2026-06-11T10:16:03Z",
  "zone_id": "Z_S5",
  "overall_status": "ok|warning|critical",
  "issues": [...],
  "shelf_fill_rate": 0.95,
  "products_detected": [...],
  "model_reasoning": "raciocínio passo a passo"
}
```

---

### 2. Rule Engine (`src/rule_engine.py`)

Converte regras em linguagem natural para JSON executável. Deteta ambiguidades e pede clarificação.

```bash
# Carregar regras de ficheiro
python src/rule_engine.py --load-rules data/rules/rules_default.txt

# Adicionar uma regra
python src/rule_engine.py --add "Na zona Z_S5, se houver produtos danificados, alerta critical."

# Listar regras
python src/rule_engine.py --list
```

O ficheiro `data/rules/rules_default.txt` contém as regras pré-definidas (uma por linha, `#` = comentário). As linhas comentadas são intencionalmente ambíguas — para demonstrar o sistema de clarificação na defesa.

---

### 3. RAG Memory (`src/rag_memory.py`)

Indexa inspection records numa vector store (ChromaDB) e responde a queries em linguagem natural.

```bash
# Indexar todas as inspeções
python src/rag_memory.py --index

# Fazer uma query
python src/rag_memory.py --query "Que zonas tiveram problemas de prateleira vazia?"
```

**Stack:**
- Embeddings: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (local, offline após download ~471MB)
- Vector store: ChromaDB persistente em `vectorstore/`
- Estratégia de chunking: híbrida (summary da inspeção + chunks por issue)

Na primeira execução, o modelo de embeddings é descarregado automaticamente (~471MB). As execuções seguintes são rápidas (modelo em cache).

---

### 4. Report Generator (`src/report_generator.py`)

Gera um relatório Markdown com 5 secções obrigatórias: Sumário Executivo, Problemas por Zona, Regras Disparadas, Contexto Histórico (RAG), Recomendações.

```bash
python src/report_generator.py
```

O relatório é guardado em `data/inspections/report_YYYY-MM-DD_HH-MM-SS.md`.

---

### 5. Interface Conversacional (`src/interface.py`)

CLI interactiva com 4 modos de operação.

```bash
python src/interface.py
```

**Comandos disponíveis:**

```
# Inspeção
> inspect Z_S5 --image foto.jpg
> inspect all --images-dir ./fotos/

# Regras
> add rule "Avisa-me quando o fill rate cair abaixo de 50%."
> list rules
> delete rule RULE_003
> test rule RULE_001 --image foto.jpg
> load rules data/rules/rules_default.txt

# Histórico
> history "Que zonas tiveram problemas esta semana?"
> compare Z_S1 Z_S3 --period "last 7 days"

# Relatório
> report --session today
> report --zone Z_S5 --period "last 14 days"

# Outros
> help
> status
> exit
```

---

## Avaliação

```bash
python evaluate.py --images-dir data/ground_truth --output evaluation_report.json
```

O professor fornece `test_images/` no momento da defesa com 10 imagens não vistas.

**Resultados obtidos (21 imagens de ground truth):**

| Componente | Métrica | Valor |
|---|---|---|
| Análise Visual | JSON Parse Rate | 100% |
| | Issue Detection Rate | 61,9% |
| | False Positive Rate | 26,6% |
| | Severity Accuracy | 85,7% |
| | Hallucination Rate | 0,0% |
| Rule Engine | Rule Parse Rate | 100% |
| | Rule Correctness | 100% |
| | Ambiguity Detection | 100% |
| LLM-as-Judge | Inspection Quality | 88,0% |
| | Report Quality | 70,0% |

---

## Fluxo Completo

```bash
# 1. Carregar as regras
python src/rule_engine.py --load-rules data/rules/rules_default.txt

# 2. Analisar imagens e guardar inspeções
python src/shelf_inspector.py --images-dir data/images --save

# 3. Indexar as inspeções no RAG
python src/rag_memory.py --index

# 4. Gerar relatório
python src/report_generator.py

# 5. Interface conversacional
python src/interface.py
```

---

## Zonas da Loja

| Zona | Descrição |
|---|---|
| Z_S1 | Frescos e laticínios |
| Z_S2 | Padaria e pastelaria |
| Z_S3 | Talho e charcutaria |
| Z_S4 | Higiene e limpeza |
| Z_S5 | Bebidas e conservas |
| Z_S6 | Vinhos e destilados |
| Z_S7 | Congelados |
| Z_UNKNOWN | Zona não identificada |
