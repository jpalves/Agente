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
- `numpy` (pesquisa vetorizada)

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

Ou inicia o servidor HTTPS (por omissão, abre `https://localhost:8443/` no
browser; `0.0.0.0` é o endereço de escuta e não o endereço a escrever). Noutro
dispositivo da mesma rede, usa o endereço IP deste computador, por exemplo
`https://192.168.50.33:8443/`:

```bash
./rag --servidor
./rag --servidor --host 127.0.0.1 --porta 9443
```

O servidor cria um certificado autoassinado com nomes alternativos para `localhost`, `127.0.0.1` e os endereços IP locais. O browser ainda mostrará um aviso de certificado não confiável; aceita a exceção para o endereço IP antes de usar o chat. Para exposição pública, configura um certificado válido ou um proxy HTTPS, como Caddy ou nginx.
`/api/chat` é apenas a rota POST usada pela página; abri-la diretamente no browser não abre o chat.

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
- No browser, cada separador mantém uma conversa independente em `sessionStorage`; atualizar a página mantém os últimos turnos. O botão **Limpar conversa** apaga o histórico desse separador; fechar o separador termina essa conversa.
- O controlo **Usar histórico** no chat web e o comando `/historico` no terminal ativam ou desativam o envio de mensagens anteriores ao modelo. Desativar não apaga o registo visível; ao reativar, o histórico da conversa volta a ser usado. A preferência do controlo web fica guardada por separador.
- Se o browser bloquear o armazenamento da sessão, as respostas continuam a ser mostradas; só a persistência do histórico fica indisponível. Os erros de rede/timeout são apresentados na conversa e na consola do browser.
- Perguntas de seguimento usam as duas perguntas anteriores na pesquisa, e pedidos explícitos para repetir/resumir a resposta anterior são respondidos a partir do histórico sem acrescentar excertos de documentos.
- A pesquisa ignora palavras comuns, exige correspondência de palavras de conteúdo quando existe, e só recorre a resultados puramente semânticos com `SIMILARIDADE_EMBEDDINGS_MINIMA`. Isto reduz o risco de apresentar documentos sem relação como fontes; baixa esse limiar se consultas por sinónimos deixarem de encontrar material.
- A pesquisa limita a três trechos por ficheiro para evitar que um único PDF ocupe todo o contexto. A resposta da API inclui os caminhos dos ficheiros cujos excertos foram fornecidos ao modelo; o browser mostra-os separadamente da resposta.
- Quando há resultados em pelo menos duas fontes, o chat e o terminal mostram uma estimativa heurística de dispersão da pesquisa. Calcula-se a distribuição da pontuação máxima por fonte e o número equivalente de fontes com peso semelhante; é um sinal de possível ambiguidade, não uma medida calibrada de confiança factual nem uma confirmação de sentidos diferentes.
- A estratégia de resposta combina essa dispersão com os resultados da pesquisa: dispersão baixa prioriza a fonte mais relevante; dispersão moderada/alta pede ao modelo uma síntese comparativa com atribuição das fontes; sem excertos que passem os filtros de relevância, o modelo pode responder com conhecimento geral, identificando que não encontrou apoio no corpus. Entropia alta não faz, por si só, os documentos serem descartados.

`NUM_CTX=16384` define a janela de contexto enviada ao Ollama. É importante que seja suficientemente grande para o histórico e os trechos recuperados; se for demasiado baixa, o Ollama pode truncar o prompt e o modelo perder parte da conversa. Uma janela maior aumenta o uso de memória.

## Configuração atual

Os parâmetros encontram-se no início do ficheiro `rag`:

| Constante | Valor atual | Função |
|---|---:|---|
| `MODEL` | `hf.co/duarteocarmo/AMALIA-9B-0626-DPO-GGUF:Q4_K_M` | Modelo de chat |
| `EMBEDDING_MODEL` | `nomic-embed-text` | Modelo que cria os embeddings |
| `CHUNK_SIZE` | 1800 | Tamanho alvo dos trechos, em caracteres |
| `CHUNK_OVERLAP` | 240 | Sobreposição entre trechos |
| `TOP_K` | 8 | Máximo de trechos recuperados para contexto |
| `MODO_RESPOSTA` | `automatico` | `automatico`, `priorizar_documentos` ou `priorizar_llm` |
| `MAX_TRECHOS_POR_FONTE` | 3 | Máximo de trechos selecionados por ficheiro |
| `SIMILARIDADE_MINIMA` | 0.25 | Pontuação híbrida mínima para incluir um trecho |
| `SIMILARIDADE_EMBEDDINGS_MINIMA` | 0.68 | Semelhança mínima quando não há correspondência lexical |
| `PESO_EMBEDDINGS` | 0.75 | Peso da similaridade de cosseno na pontuação híbrida |
| `PESO_JACCARD` | 0.25 | Peso da cobertura lexical ponderada por IDF |
| `TAMANHO_LOTE_EMBED` | 16 | Trechos por pedido de embeddings |
| `MAX_MENSAGENS_HISTORICO` | 8 | Mensagens anteriores mantidas |
| `NUM_CTX` | 16384 | Janela de contexto pedida ao Ollama |
| `TEMPO_LIMITE_OLLAMA` | 240 segundos | Timeout dos pedidos ao Ollama |
| `MOSTRAR_RACIOCINIO` | `False` | Estado inicial da apresentação do raciocínio |
| `SERVIDOR_HOST` / `SERVIDOR_PORTA` | `0.0.0.0` / `8443` | Endereço e porta HTTPS por omissão |

## MathJax

A página web carrega MathJax 3 a partir do jsDelivr e renderiza fórmulas em linha (`$...$` ou `\( ... \)`) e em bloco (`$$...$$` ou `\[ ... \]`). É necessária ligação à Internet para carregar a biblioteca; sem ela, as fórmulas podem aparecer como texto LaTeX.

## Resolução de problemas

- **`ModuleNotFoundError` para `ollama`, `httpx`, `pypdf` ou `numpy`** — ativa o ambiente virtual e executa `pip install -r requirements.txt`.
- **Ollama não responde ou ocorre timeout** — confirma `systemctl status ollama` e testa `curl http://127.0.0.1:11434/api/tags`.
- **`connection reset by peer` durante embeddings** — confirma se o serviço Ollama continua ativo; os embeddings são enviados em lotes para reduzir sobrecarga.
- **O modelo perde mensagens anteriores** — confirma que estás a executar a versão atual de `rag`, reinicia o servidor e mantém `NUM_CTX` suficiente para o prompt.
- **`does not support thinking`** — nem todos os modelos aceitam o parâmetro de raciocínio; o programa desativa-o automaticamente após a resposta de erro.
- **Fórmulas não renderizadas** — MathJax depende da CDN; confirma a ligação à Internet.
- **Avisos `Impossible to decode XFormObject`** — podem ser emitidos pelo `pypdf` ao encontrar objetos gráficos de PDFs; não significam necessariamente que a extração de texto falhou.
- **Aviso de certificado no browser** — o certificado gerado automaticamente é autoassinado; para produção usa um certificado válido.

## Desempenho

- A pesquisa usa **numpy**: os embeddings ficam numa matriz normalizada (cosseno = um produto matriz-vetor) e a cobertura lexical usa um índice invertido de palavras. Com ~58 mil trechos, `recuperar` passou de ~11 s para ~0,1 s por pergunta.
- `KEEP_ALIVE = "30m"` mantém os modelos carregados no Ollama entre perguntas.
- O tempo restante de cada resposta é o LLM; para o reduzir, diminua `TOP_K` ou `CHUNK_SIZE` (menos contexto no prompt). Alterar `CHUNK_SIZE` obriga a reindexar tudo.

## Relevância das respostas

- Os excertos recuperados vão na mensagem do utilizador, imediatamente antes da pergunta, com instruções para prevalecerem sobre o conhecimento do modelo (`temperature` 0,2).
- A componente lexical é uma cobertura ponderada por IDF (palavras raras pesam mais; as que existem em mais de `FREQUENCIA_MAXIMA_PALAVRA` dos trechos são ignoradas), em vez de Jaccard puro, que ficava diluído em trechos longos.
- Se o modelo continuar a ignorar os documentos, aumenta `SIMILARIDADE_MINIMA` (ex.: 0.45) para só passarem trechos realmente relevantes.
- `FATOR_PESO_PDF` e `FATOR_PESO_TEXTO` permitem ajustar separadamente o peso do tipo de ficheiro na pontuação final. `1.0` mantém a pontuação; valores acima de `1.0` favorecem esse tipo e abaixo de `1.0` penalizam-no. Por exemplo, `FATOR_PESO_PDF = 0.8` e `FATOR_PESO_TEXTO = 1.2` favorece ficheiros de texto. O ajuste não exige reindexar, mas requer reiniciar o programa/servidor.
