# Neo – agente RAG local com Ollama

Chatbot de linha de comandos e servidor HTTPS (página de chat com MathJax) que responde com base nos documentos da pasta `documentos/`, usando modelos locais via [Ollama](https://ollama.com).

## Requisitos

| Componente | Versão testada | Notas |
|---|---|---|
| Linux | – | testado em Ubuntu (Python 3.12) |
| Python | 3.12 | 3.10+ deve funcionar (usa `X \| None` e `from __future__ import annotations`) |
| Ollama | 0.32.x | serviço a correr em `http://127.0.0.1:11434` |
| OpenSSL | qualquer | só para gerar o certificado do servidor HTTPS |
| Espaço em disco | ~6 GB | modelo de chat (~5,6 GB) + embeddings (~274 MB) |
| RAM | ≥ 8 GB | o modelo de 9B quantizado (Q4_K_M) usa ~6 GB |
| Internet | opcional | só para descarregar modelos e para o MathJax (CDN) na página web |

Dependências Python (`requirements.txt`): `ollama`, `httpx`, `pypdf`. O resto usa só a biblioteca padrão (`sqlite3`, `http.server`, `ssl`, …).

## Instalação

```bash
# 1. Ollama (https://ollama.com/download)
curl -fsSL https://ollama.com/install.sh | sh

# 2. Modelos
ollama pull nomic-embed-text
ollama pull hf.co/duarteocarmo/AMALIA-9B-0626-DPO-GGUF:Q4_K_M

# 3. Ambiente virtual e dependências Python
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Confirma que o Ollama responde: `curl http://127.0.0.1:11434/api/tags`.

## Estrutura

```
Neo/
├── rag                 # script principal (executável)
├── requirements.txt
├── documentos/         # base de conhecimento (subpastas permitidas)
├── certificados/       # servidor.crt / servidor.key (gerados automaticamente)
└── indice.db           # cache SQLite dos embeddings (gerada automaticamente)
```

## Utilização

```bash
source .venv/bin/activate

./rag                                   # chat no terminal
./rag --servidor                        # servidor HTTPS em https://0.0.0.0:8443
./rag --servidor --host 127.0.0.1 --porta 9443
```

No servidor, o browser avisa que o certificado é auto-assinado (esperado em desenvolvimento): aceita a exceção.

Comandos no chat de terminal: `/reindexar`, `/raciocinio`, `/sair`.

### Documentos suportados

`.txt .md .py .json .c .cpp .cc .h .hpp .pdf`, lidos recursivamente. PDFs são lidos com `pypdf` (só texto; PDFs digitalizados precisam de OCR, não incluído).

### Cache de embeddings

- O 1.º arranque calcula os embeddings (lento: ~14 min para ~18 mil trechos) e guarda-os em `indice.db`.
- Os arranques seguintes demoram segundos; só ficheiros novos ou alterados são reindexados.
- A cache é refeita automaticamente se mudares `EMBEDDING_MODEL`, `CHUNK_SIZE` ou `CHUNK_OVERLAP`. Para forçar, apaga `indice.db`.

## Configuração (topo do ficheiro `rag`)

| Constante | Valor | Descrição |
|---|---|---|
| `MODEL` | AMALIA-9B Q4_K_M | modelo de chat |
| `EMBEDDING_MODEL` | `nomic-embed-text` | modelo de embeddings |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | 900 / 120 | tamanho e sobreposição dos trechos (caracteres) |
| `TOP_K` | 8 | nº máximo de trechos no contexto |
| `SIMILARIDADE_MINIMA` | 0.25 | pontuação mínima para um trecho entrar no contexto |
| `PESO_EMBEDDINGS` / `PESO_JACCARD` | 0.7 / 0.3 | pesos da pontuação híbrida (cosseno + Jaccard) |
| `TAMANHO_LOTE_EMBED` | 16 | trechos por pedido de embeddings |
| `TEMPO_LIMITE_OLLAMA` | 60 s | timeout dos pedidos ao Ollama |
| `MOSTRAR_RACIOCINIO` | `False` | mostra o campo `thinking` (só em modelos que o suportem) |
| `SERVIDOR_HOST` / `SERVIDOR_PORTA` | `0.0.0.0` / `8443` | endereço do servidor HTTPS |

Com bases grandes e diversas, `TOP_K=6` e `SIMILARIDADE_MINIMA=0.40` reduzem a mistura de fontes.

## Resolução de problemas

- **`ModuleNotFoundError: pypdf/ollama/httpx`** – ativa o venv (`source .venv/bin/activate`) e corre `pip install -r requirements.txt`.
- **`connection reset by peer` / timeouts** – verifica `systemctl status ollama`; os embeddings já são enviados em lotes pequenos para evitar sobrecarga.
- **`does not support thinking`** – o modelo não suporta `think`; é desativado automaticamente. Modelos que suportam: `qwen3`, `deepseek-r1`.
- **Fórmulas sem renderizar** – o MathJax vem de uma CDN; sem internet aparece o LaTeX em texto.
- **Avisos `Impossible to decode XFormObject`** – vêm do `pypdf` com PDFs com imagens; são inofensivos.
- **Aviso de certificado no browser** – certificado auto-assinado; para produção usa um certificado válido ou um proxy (nginx/Caddy).
