# Neo — agente RAG local com Ollama

Agente RAG com interface de linha de comandos e página de chat HTTPS. Procura informação na pasta `documentos/` e usa modelos locais via [Ollama](https://ollama.com). A página web suporta fórmulas matemáticas através do MathJax.

## Requisitos

| Componente | Versão testada | Notas |
|---|---|---|
| Linux | Ubuntu | Ambiente onde o agente foi testado |
| Python | 3.12 | Versões anteriores não foram verificadas |
| Ollama | 0.32.14 | Deve estar a correr em `http://127.0.0.1:11434` |
| OpenSSL | — | Necessário para gerar automaticamente o certificado HTTPS |
| RAM e disco | Dependem dos modelos e da janela de contexto | O modelo de chat ocupa cerca de 5,6 GB em disco; `NUM_CTX=16384` aumenta o uso de memória durante a geração |
| Internet | Opcional | Necessária para descarregar modelos e carregar MathJax da CDN; não é necessária para o chat local depois da instalação |

Dependências Python, também listadas em `requirements.txt`:

- `ollama` (testado com 0.6.2)
- `httpx` (testado com 0.28.1)
- `pypdf` (testado com 6.19.0)

O restante usa a biblioteca padrão do Python, incluindo `sqlite3` para a cache dos embeddings.

## Instalação

Instala o Ollama seguindo as instruções oficiais em [ollama.com/download](https://ollama.com/download). Em Linux, o instalador oficial também pode ser usado:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Descarrega os modelos usados pela configuração atual:

```bash
ollama pull nomic-embed-text
ollama pull hf.co/duarteocarmo/AMALIA-9B-0626-DPO-GGUF:Q4_K_M
```

Na pasta do projeto, cria o ambiente virtual e instala as dependências:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Confirma que o serviço Ollama está ativo:

```bash
curl http://127.0.0.1:11434/api/tags
```

## Estrutura

```text
Neo/
├── rag                 # script principal executável
├── README.md
├── requirements.txt
├── documentos/         # documentos da base de conhecimento
├── certificados/       # certificado e chave HTTPS, gerados automaticamente
├── indice.db           # cache SQLite dos trechos e embeddings
└── favicon.ico         # ícone servido pela página web
```

`indice.db` e os certificados são gerados em execução, conforme necessário. Não é preciso criá-los manualmente.

## Utilização

Ativa o ambiente virtual antes de iniciar:

```bash
source .venv/bin/activate
```

Inicia o chat no terminal:

```bash
./rag
```

Ou inicia o servidor HTTPS (por omissão, `https://0.0.0.0:8443`):

```bash
./rag --servidor
./rag --servidor --host 127.0.0.1 --porta 9443
```

O servidor cria um certificado autoassinado. O browser mostrará um aviso de certificado não confiável; isto é esperado para uso local. Para exposição pública, configura um certificado válido ou um proxy HTTPS, como Caddy ou nginx.

Comandos do modo terminal:

- `/reindexar` — atualiza a cache após adicionar, alterar ou remover documentos.
- `/limpar` — limpa o histórico da conversa atual.
- `/raciocinio` — alterna a apresentação do campo de raciocínio, quando suportado pelo modelo.
- `/sair` — termina o programa.

## Documentos suportados

O agente percorre `documentos/` e as suas subpastas. Suporta:

```text
.txt .md .py .json .c .cpp .cc .h .hpp .pdf
```

Os PDFs são extraídos com `pypdf`. Só é extraído texto incorporado no PDF; documentos digitalizados que contenham apenas imagens precisam de OCR, que não está incluído.

## Indexação e cache SQLite

Os trechos e embeddings são guardados em `indice.db`, na mesma pasta do script:

- Na primeira execução, os embeddings têm de ser calculados e a indexação pode demorar vários minutos, dependendo da quantidade de documentos e do hardware.
- Nas execuções seguintes, os embeddings em cache são carregados sem voltar a chamar o modelo para ficheiros inalterados.
- Ficheiros novos ou alterados são reindexados; ficheiros removidos são retirados da cache.
- A cache é invalidada automaticamente se mudar o modelo de embeddings, `CHUNK_SIZE` ou `CHUNK_OVERLAP`.
- Para forçar uma reindexação integral, para o agente e remove `indice.db`; o ficheiro será recriado no arranque seguinte.
- Os embeddings são enviados ao Ollama em lotes pequenos para limitar o uso de memória.

## Conversa e memória

O modelo é sem estado entre pedidos; o programa envia-lhe o histórico recente em cada chamada:

- Guarda no máximo as últimas 8 mensagens (perguntas e respostas), sem repetir os trechos recuperados.
- No modo terminal, o histórico dura até terminares o programa ou usares `/limpar`.
- No browser, cada separador mantém uma conversa independente em memória. Atualizar a página ou abri-la novamente limpa essa conversa.
- Perguntas de seguimento também usam a pergunta anterior na pesquisa da base de conhecimento.

`NUM_CTX=16384` define a janela de contexto enviada ao Ollama. É importante que seja suficientemente grande para o histórico e os trechos recuperados; se for demasiado baixa, o Ollama pode truncar o prompt e o modelo perder parte da conversa. Uma janela maior aumenta o uso de memória.

## Configuração atual

Os parâmetros encontram-se no início do ficheiro `rag`:

| Constante | Valor atual | Função |
|---|---:|---|
| `MODEL` | `hf.co/duarteocarmo/AMALIA-9B-0626-DPO-GGUF:Q4_K_M` | Modelo de chat |
| `EMBEDDING_MODEL` | `nomic-embed-text` | Modelo que cria os embeddings |
| `CHUNK_SIZE` | 1800 | Tamanho alvo dos trechos, em caracteres |
| `CHUNK_OVERLAP` | 240 | Sobreposição entre trechos |
| `TOP_K` | 12 | Máximo de trechos recuperados para contexto |
| `SIMILARIDADE_MINIMA` | 0.15 | Pontuação mínima para incluir um trecho |
| `PESO_EMBEDDINGS` | 0.25 | Peso da similaridade de cosseno na pontuação híbrida |
| `PESO_JACCARD` | 0.75 | Peso da similaridade de Jaccard na pontuação híbrida |
| `TAMANHO_LOTE_EMBED` | 16 | Trechos por pedido de embeddings |
| `MAX_MENSAGENS_HISTORICO` | 8 | Mensagens anteriores mantidas |
| `NUM_CTX` | 16384 | Janela de contexto pedida ao Ollama |
| `TEMPO_LIMITE_OLLAMA` | 240 segundos | Timeout dos pedidos ao Ollama |
| `MOSTRAR_RACIOCINIO` | `False` | Estado inicial da apresentação do raciocínio |
| `SERVIDOR_HOST` / `SERVIDOR_PORTA` | `0.0.0.0` / `8443` | Endereço e porta HTTPS por omissão |

## MathJax

A página web carrega MathJax 3 a partir do jsDelivr e renderiza fórmulas em linha (`$...$` ou `\( ... \)`) e em bloco (`$$...$$` ou `\[ ... \]`). É necessária ligação à Internet para carregar a biblioteca; sem ela, as fórmulas podem aparecer como texto LaTeX.

## Resolução de problemas

- **`ModuleNotFoundError` para `ollama`, `httpx` ou `pypdf`** — ativa o ambiente virtual e executa `pip install -r requirements.txt`.
- **Ollama não responde ou ocorre timeout** — confirma `systemctl status ollama` e testa `curl http://127.0.0.1:11434/api/tags`.
- **`connection reset by peer` durante embeddings** — confirma se o serviço Ollama continua ativo; os embeddings são enviados em lotes para reduzir sobrecarga.
- **O modelo perde mensagens anteriores** — confirma que estás a executar a versão atual de `rag`, reinicia o servidor e mantém `NUM_CTX` suficiente para o prompt.
- **`does not support thinking`** — nem todos os modelos aceitam o parâmetro de raciocínio; o programa desativa-o automaticamente após a resposta de erro.
- **Fórmulas não renderizadas** — MathJax depende da CDN; confirma a ligação à Internet.
- **Avisos `Impossible to decode XFormObject`** — podem ser emitidos pelo `pypdf` ao encontrar objetos gráficos de PDFs; não significam necessariamente que a extração de texto falhou.
- **Aviso de certificado no browser** — o certificado gerado automaticamente é autoassinado; para produção usa um certificado válido.
