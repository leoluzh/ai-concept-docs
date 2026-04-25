# 8 Padrões de RAG Que Você Não Deve Ignorar

> Aprenda sobre 8 arquiteturas RAG para sistemas de IA — de Naive a Agentic e Hybrid — e como cada uma melhora a precisão, recuperação e desempenho em produção.
>
> **Por Damil Shahzad** · 24 de abril de 2026 · Análise

---

Modelos de linguagem de grande escala (LLMs) geram texto fluente. No entanto, eles falham em atender aos requisitos de embasamento, rastreabilidade, atualização e controle de acesso. A Geração Aumentada por Recuperação (RAG — *Retrieval-Augmented Generation*) resolve isso obrigando os modelos a responder com base em evidências externas.

O RAG inicial usava um único pipeline simples. Os sistemas em produção atualmente utilizam múltiplos padrões arquiteturais. Cada padrão tem como alvo um modo de falha diferente. Este artigo explica oito grandes arquiteturas RAG utilizadas em produção hoje.

---

## O que é RAG?

O RAG conecta três sistemas: armazenamento, recuperação e geração. A camada de armazenamento guarda seus documentos, fragmentos (*chunks*) e embeddings. A camada de recuperação encontra evidências relevantes para cada consulta. A camada de geração produz respostas condicionadas ao contexto recuperado. O pipeline vai da consulta, passando pela recuperação de evidências e construção do contexto, até a geração da resposta e retorno das citações. Você obtém embasamento factual, uso de dados atualizados, isolamento de dados privados e suporte a rastreamento de auditoria. O RAG deslocou a engenharia de IA do ajuste de prompts para a engenharia de pipelines de dados.

A camada de armazenamento suporta múltiplos backends, incluindo bancos de dados vetoriais (Pinecone, Weaviate, Milvus), repositórios de documentos (Elasticsearch, OpenSearch) e sistemas híbridos. A camada de recuperação executa modelos de embedding, busca por palavras-chave ou travessia de grafos, dependendo da arquitetura. A camada de geração tipicamente usa um LLM com um template de prompt. As três camadas se comunicam por uma interface bem definida. Você substitui componentes sem reescrever o pipeline inteiro.

---

## 1. Naive RAG

O Naive RAG usa recuperação direta por similaridade vetorial, sem loop de feedback. O nome vem do artigo original sobre RAG. A arquitetura permanece como linha de base para comparação.

### Pipeline

A ingestão de documentos carrega texto bruto de arquivos, bancos de dados ou APIs. O pré-processamento normaliza espaços em branco, remove marcações e segmenta por limites lógicos. A fragmentação de texto divide documentos em segmentos de tamanho fixo ou variável — escolhas comuns: 256 tokens, 512 tokens ou fragmentos baseados em sentenças. A geração de embeddings converte cada fragmento em um vetor usando um modelo pré-treinado. O armazenamento vetorial grava os embeddings em um banco de dados vetorial com metadados (documento de origem, índice do fragmento, timestamp). No momento da consulta, o usuário submete uma pergunta. A incorporação da consulta converte a pergunta em um vetor. A busca vetorial retorna os top-k fragmentos mais próximos por similaridade de cosseno ou distância euclidiana. A injeção de contexto concatena os fragmentos recuperados em um prompt. A geração de resposta passa o prompt ao LLM. O retorno de citações anexa referências de fonte à saída.

### Pontos Fortes

A implementação leva de uma a duas semanas para um engenheiro experiente. O custo de infraestrutura permanece baixo: um modelo de embedding, um banco vetorial, um endpoint de LLM. A abordagem funciona bem para domínios de conhecimento estático. Corpora de FAQ, documentação de produtos e wikis internos se encaixam nesse padrão. A latência fica abaixo de 2 segundos na maioria dos deployments. A ausência de loops de feedback garante comportamento determinístico — a mesma consulta retorna o mesmo conjunto de resultados. A depuração é direta.

### Pontos Fracos

Nenhum loop de verificação valida as evidências recuperadas. Fragmentos irrelevantes passam quando a similaridade de embedding é enganosa. A qualidade do ranking depende inteiramente da similaridade de embedding. Consultas ambíguas retornam resultados fracos — uma pergunta como "como corrijo o erro" retorna conteúdo genérico de solução de problemas em vez de documentação específica do erro. Consultas multifacetadas sofrem: uma pergunta sobre "preços e integração" recupera fragmentos de apenas uma faceta. O modelo alucina para preencher lacunas quando a recuperação falha.

### Tamanho do Fragmento

A escolha do tamanho do fragmento impacta a qualidade da recuperação. Fragmentos pequenos (128 tokens) fornecem correspondências precisas, mas perdem contexto. Fragmentos grandes (512 tokens) capturam mais contexto, mas diluem a relevância. A sobreposição entre fragmentos (50 tokens) ajuda a preservar contexto nas fronteiras. Teste múltiplos tamanhos (128, 256, 512) em seu conjunto de consultas. Meça o recall em k=5 e k=10.

### Modelos de Embedding

A escolha do modelo de embedding impacta a cobertura semântica. Modelos treinados em textos gerais (OpenAI text-embedding-ada-002, sentence-transformers/all-MiniLM) têm desempenho inferior em corpora específicos de domínio. Use embeddings ajustados ao domínio quando disponíveis. Faça fine-tuning em seu corpus com perda contrastiva. A dimensão do embedding importa: modelos de 384 dimensões são mais rápidos e baratos; modelos de 1536 dimensões capturam distinções mais refinadas.

### Casos de Uso em Produção

- Bots de FAQ com menos de 10.000 perguntas
- Busca em documentação de manuais de produto e referências de API
- Bases de conhecimento internas com conteúdo que muda com pouca frequência
- POCs e demos onde velocidade de implementação supera precisão

---

## 2. Agentic RAG

O Agentic RAG adiciona planejamento, seleção de ferramentas e raciocínio iterativo. O agente divide perguntas complexas em etapas, escolhe ferramentas para cada etapa, as executa e sintetiza uma resposta final. Essa arquitetura lida com fluxos de trabalho que uma única chamada de recuperação não consegue suportar.

### Pipeline

O planejamento de tarefas analisa a consulta do usuário e produz um plano passo a passo. O planejador usa um LLM com exemplos few-shot ou um prompt estruturado. As etapas do plano incluem "recuperar documentos sobre X", "chamar API Y" e "resumir resultados". A seleção de ferramentas mapeia cada etapa para uma ferramenta — busca vetorial, busca por palavras-chave, calculadora, chamadas de API e execução de código. A recuperação em múltiplas etapas executa ferramentas em sequência, onde saídas de etapas anteriores alimentam etapas posteriores. A atualização de memória armazena saídas de ferramentas, conclusões intermediárias e feedback do usuário. A síntese de resposta gera a resposta final a partir do contexto acumulado. O agente volta ao planejamento quando a síntese indica informações faltantes.

### Pontos Fortes

Lida com fluxos de trabalho complexos. Uma consulta como "compare nossos resultados do Q3 com nossos três principais concorrentes e resuma a lacuna" requer múltiplas recuperações, chamadas de API e sumarização. Tarefas de raciocínio de longa duração se tornam viáveis — assistentes de pesquisa consultam papers, patentes e fontes de notícias.

### Pontos Fracos

A latência aumenta com cada etapa de planejamento e execução — uma única consulta frequentemente aciona de 3 a 10 chamadas ao modelo, chegando a 10–30 segundos de latência total. A depuração é difícil: o agente escolhe diferentes ferramentas ou caminhos para consultas similares. O custo de infraestrutura sobe, pois cada etapa consome tokens.

### Orientações de Implementação

- Defina um número máximo de etapas (tipicamente 5 a 10)
- Registre cada chamada de ferramenta e etapa do plano com rastreamento completo
- Defina schemas de ferramentas com descrições claras
- Implemente timeouts por etapa com comportamento de fallback

### Casos de Uso em Produção

- Automação de pesquisa: revisão de literatura, análise de patentes, sumarização de tendências
- Inteligência competitiva: monitoramento de mercado, acompanhamento de concorrentes
- Analytics autônoma: relatórios ad-hoc, exploração de dados, geração de dashboards

---

## 3. HyDE RAG

O HyDE (*Hypothetical Document Embeddings*) gera documentos sintéticos para melhorar o matching de recuperação. A ideia: respostas hipotéticas estão mais próximas no espaço de embedding de respostas reais do que consultas brutas. Isso preenche a lacuna de vocabulário entre como os usuários perguntam e como os documentos são escritos.

### Pipeline

O usuário submete uma consulta. A geração de resposta hipotética produz uma ou mais respostas plausíveis usando um LLM — por exemplo, "Como configuro SSL?" pode gerar: "Para configurar SSL, você precisa gerar um certificado, adicionar o caminho do certificado ao arquivo de configuração e reiniciar o servidor." A geração de embedding converte a resposta hipotética em um vetor. A recuperação usa esse vetor em vez do vetor da consulta para buscar no corpus. Os fragmentos recuperados são documentos reais, não hipotéticos. A geração final produz a resposta real a partir das evidências recuperadas, citando fontes reais.

### Variações

- **HyDE único:** gera uma resposta hipotética por consulta
- **Multi-HyDE:** gera de 3 a 5 respostas hipotéticas, embute cada uma, recupera e mescla os resultados — melhora o recall, mas multiplica o custo
- **HyDE com reranking:** adiciona um reranker após a recuperação para filtrar falsos positivos

### Pontos Fortes

A qualidade do recall melhora — benchmarks reportam ganhos de 10–30% sobre recuperação ingênua. A incompatibilidade de vocabulário entre consultas e documentos diminui. Busca técnica se beneficia mais: perguntas de desenvolvedores, mensagens de erro e padrões de uso de API se alinham melhor após o HyDE.

### Pontos Fracos

Inferência extra: o modelo deve gerar uma resposta hipotética antes da recuperação — espere 1,5x a 2x o uso de tokens por consulta. Viés sintético é um risco: documentos gerados às vezes distorcem a recuperação em direção a certos tipos de documento.

### Orientações de Implementação

- Use um modelo rápido e barato para geração hipotética (um modelo de 7B parâmetros frequentemente é suficiente)
- Mantenha respostas hipotéticas concisas (50 a 100 tokens)
- Considere caching: consultas repetidas reutilizam hipotéticos armazenados em cache

---

## 4. Graph RAG

O Graph RAG extrai relacionamentos de entidades em grafos de conhecimento. Documentos se tornam nós e arestas. As consultas percorrem o grafo para montar o contexto. Essa arquitetura se destaca quando os relacionamentos importam tanto quanto o texto bruto.

### Pipeline

A extração de entidades identifica entidades nomeadas em um documento — pessoas, organizações, produtos e conceitos — usando modelos NER, padrões baseados em regras ou parsing com LLM. A vinculação de entidades resolve entidades extraídas para IDs canônicos ("Apple Inc" e "Apple" mapeiam para o mesmo nó). A construção do grafo cria nós para entidades e arestas para relacionamentos. O armazenamento do grafo usa um banco de dados de grafos (Neo4j, Amazon Neptune) ou um grafo em memória. No momento da consulta, a compreensão da consulta identifica entidades mencionadas. A travessia do grafo parte dessas entidades e segue as arestas — estratégias incluem travessia por vizinhança k-hop, busca de caminho e detecção de comunidade. A montagem do contexto extrai texto dos documentos associados aos nós percorridos.

### Pontos Fortes

O raciocínio multi-hop se torna tratável — "Que medicamentos interagem com o medicamento atual do paciente?" requer encadeamento de relacionamentos droga-droga por múltiplos saltos. A explicabilidade é forte: o caminho de raciocínio segue arestas explícitas do grafo. A recuperação com consciência de relacionamentos superficializa conceitos relacionados que a busca vetorial ingênua perderia.

### Pontos Fracos

A construção do grafo é cara — extração e vinculação de entidades requerem modelos treinados ou regras (espere semanas de ajuste para um novo domínio). O design do schema é complexo. Os pipelines de atualização do grafo devem se alinhar com os ciclos de atualização dos dados de origem.

### Orientações de Implementação

- Comece com um schema mínimo: dois ou três tipos de relacionamento frequentemente bastam
- Use recuperação híbrida: combine travessia de grafo com busca vetorial
- Execute atualizações incrementais: reconstrua o grafo completo apenas quando o schema mudar

### Casos de Uso em Produção

- Suporte à decisão clínica em saúde
- Detecção de fraude
- Pesquisa científica, onde estrutura de relacionamentos importa tanto quanto texto

---

## 5. Corrective RAG

O Corrective RAG adiciona auto-validação e refinamento iterativo. O sistema gera uma resposta, a critica e re-recupera ou re-gera quando a crítica identifica problemas. O loop continua até que a resposta atinja um limiar de qualidade. Essa arquitetura é ideal para domínios de alto risco onde erros são custosos.

### Pipeline

A recuperação inicial busca os top-k fragmentos para a consulta. A geração inicial produz um rascunho de resposta. A crítica avalia o rascunho: a resposta cita as evidências recuperadas? As afirmações são suportadas? Há contradições? O crítico usa um LLM com prompt estruturado ou um classificador treinado. A pontuação produz um score numérico (0 a 1) ou passa/falha. A re-consulta é acionada quando a crítica encontra evidências faltantes ou afirmações não suportadas. A re-geração produz um novo rascunho a partir do contexto expandido. O loop se repete até que o score exceda um limiar ou o máximo de iterações (ex.: 3) seja atingido.

### Pontos Fortes

A precisão factual melhora — benchmarks mostram redução de 15 a 25% na taxa de alucinação. A arquitetura é adequada para aplicações que requerem trilhas de auditoria robustas. Cada resposta vem acompanhada de um log de crítica.

### Pontos Fracos

Maior latência: a maioria das implementações executa de 2 a 4 passagens de geração por consulta — a latência dobra ou triplica. O uso de tokens sobe proporcionalmente. As equipes de engenharia devem projetar funções de pontuação para o estágio de crítica. Um crítico fraco adiciona custo sem benefício. Ajustar o crítico é não trivial.

### Orientações de Implementação

- Comece com uma crítica simples: "Cada afirmação tem uma citação?"
- Use chain-of-thought para o crítico — peça que explique seu raciocínio antes de pontuar
- Defina um número máximo de iterações conservador (três passagens geralmente bastam)
- Registre scores de crítica ao longo do tempo para detectar deriva

### Casos de Uso em Produção

- Analytics financeiro: resumos de resultados, relatórios de risco, verificações de conformidade
- Pesquisa jurídica: recuperação de jurisprudência, análise de contratos, consulta regulatória
- Sistemas de conformidade: verificação de políticas, suporte a auditoria, relatórios regulatórios

---

## 6. Contextual RAG

O Contextual RAG usa estado de conversa e memória de sessão. A recuperação considera turnos anteriores. A geração mantém continuidade. Essa arquitetura suporta diálogos multi-turno onde cada pergunta depende do contexto.

### Pipeline

O armazenamento de sessão mantém um log de mensagens do usuário e respostas do assistente. A sumarização de contexto é executada quando o log excede um limite de tokens — comprimindo turnos antigos em um resumo mais curto. A recuperação com consciência de contexto usa toda a conversa, não apenas a última mensagem — uma consulta como "e quanto ao segundo?" é recuperada com base em "segundo" mais a discussão anterior de uma lista. A geração de resposta recebe o contexto recuperado mais o histórico da conversa. A memória é atualizada com novas informações do turno atual para recuperação futura.

### Pontos Fortes

A consistência multi-turno melhora — perguntas de acompanhamento recebem respostas corretas. "Qual é o preço?" após "Me fale sobre o Produto X" retorna o preço do Produto X. A personalização baseada no histórico do usuário se torna possível.

### Pontos Fracos

A deriva de memória é um risco: contexto obsoleto ou irrelevante se acumula em sessões longas. A contaminação de contexto ocorre quando turnos anteriores enviesam a recuperação de formas indesejadas. A compactação de memória deve ser executada periodicamente — sem compactação, as janelas de contexto transbordam e a relevância se degrada.

### Orientações de Implementação

- Defina um orçamento de janela de contexto (tokens para histórico, contexto de recuperação e geração)
- Use uma janela deslizante com resumo: mantenha os últimos N turnos literalmente e resuma o resto
- Armazene correções do usuário explicitamente
- Teste com sessões longas — simule conversas de 20 turnos

### Casos de Uso em Produção

- Assistentes de reunião: sumarização, itens de ação, perguntas de acompanhamento
- Ferramentas de sucesso do cliente: diálogos de suporte, fluxos de onboarding
- Sistemas de conhecimento pessoal: anotações, assistentes de pesquisa, companheiros de aprendizado

---

## 7. Modular RAG

O Modular RAG divide a recuperação em componentes independentes. Cada componente tem uma única responsabilidade. Você substitui, atualiza ou ignora componentes sem reescrever o pipeline. Essa arquitetura suporta necessidades empresariais complexas onde uma solução única falha.

### Pipeline

A reescrita de consulta normaliza e expande a consulta do usuário — correção ortográfica, expansão de consulta e geração multi-consulta (estilo HyDE) ocorrem aqui. A recuperação híbrida executa múltiplas estratégias de busca em paralelo — busca vetorial, busca por palavras-chave e travessia de grafos — cujos resultados alimentam uma etapa de fusão. A filtragem remove resultados irrelevantes por restrições de metadados (intervalo de datas, fonte, controle de acesso). O reranking pontua o conjunto filtrado com um cross-encoder ou ranker aprendido. O roteamento de ferramentas envia consultas para ferramentas especializadas — uma consulta jurídica vai para o corpus jurídico; uma consulta de suporte vai para o corpus de suporte. A síntese de resposta monta a resposta final.

### Pontos Fortes

Cada módulo é atualizado ou substituído de forma independente. A arquitetura suporta fluxos de trabalho empresariais flexíveis — diferentes departamentos precisam de corpora e regras diferentes. Adicionar novas fontes de dados ou estratégias de recuperação é direto.

### Pontos Fracos

A complexidade do sistema aumenta — um pipeline modular completo tem de 6 a 10 componentes, cada um com sua própria configuração, dependências e modos de falha. A observabilidade entre módulos se torna crítica. Um bug na reescrita de consulta corrompeu silenciosamente a recuperação downstream.

### Orientações de Implementação

- Defina interfaces claras entre módulos (formato de entrada/saída padrão)
- Use um framework de pipeline (LangChain, LlamaIndex ou DAG customizado)
- Instrumente cada módulo: registre entradas, saídas e latência
- Versione seu pipeline e faça testes A/B de mudanças em módulos antes do rollout completo
- Comece minimal: um pipeline de 3 módulos (recuperar, reranquear, gerar) frequentemente basta

### Casos de Uso em Produção

- Plataformas de IA empresarial: multi-tenant, multi-corpus, acesso baseado em papel
- Sistemas de pipeline de dados em grande escala: bilhões de documentos, múltiplos backends
- Automação de pesquisa: busca federada, ferramentas especializadas, reprodutibilidade

---

## 8. Hybrid RAG

O Hybrid RAG combina recuperação por palavras-chave e recuperação semântica. A busca por palavras-chave encontra correspondências exatas e lexicais. A busca semântica encontra correspondências conceituais. Juntas, cobrem casos onde qualquer uma delas isolada falha.

### Pipeline

O parsing de consulta extrai palavras-chave e opcionalmente gera uma consulta semântica. A busca por palavras-chave ocorre em um índice invertido (BM25 ou Elasticsearch). A busca semântica ocorre contra um índice vetorial. Ambas retornam listas ranqueadas. A fusão de rankings mescla as duas listas — a Reciprocal Rank Fusion (RRF) é a linha de base comum: `score = sum(1/(k + rank))` entre as listas, com k tipicamente igual a 60. Opcionalmente, o reranking pontua a lista fundida. A geração recebe os principais fragmentos e produz a resposta.

### Palavras-chave vs. Semântica

A busca por palavras-chave se destaca em correspondências exatas — IDs de produto, códigos de erro, nomes próprios. A busca semântica se destaca em paráfrases e consultas conceituais — "Como corrijo problemas de conexão?" corresponde a "solução de problemas de conectividade de rede." O híbrido cobre ambos: uma consulta sobre "receita Q3" obtém hits de palavras-chave em "Q3" e "receita" mais hits semânticos em relatórios de resultados e resumos financeiros.

### Pontos Fortes

Precisão vem da correspondência por palavras-chave. Recall vem da busca semântica. Dados estruturados e não estruturados funcionam bem juntos. Casos de uso em produção incluem busca jurídica, auditorias de conformidade e plataformas de busca empresarial.

### Pontos Fracos

O ajuste do ranking é complexo — modelos de fusão de rankings requerem otimização contínua. O RRF assume contribuição igual, mas seus dados frequentemente precisam de pesos diferentes. Modelos de fusão aprendida frequentemente superam o RRF, mas precisam de dados de treinamento (pares consulta-documento rotulados).

### Orientações de Implementação

- Comece com RRF (sem necessidade de treinamento) e ajuste k (tipicamente 40–80) em um pequeno conjunto de validação
- Adicione filtros de metadados (fonte, data, acesso)
- Considere roteamento por tipo de consulta: consultas curtas (1 a 3 palavras) geralmente precisam de mais peso de palavras-chave; consultas longas e conceituais precisam de mais peso semântico
- Implemente ambos os caminhos em paralelo para manter a latência baixa

### Casos de Uso em Produção

- Busca jurídica: jurisprudência, contratos, regulamentações
- Auditoria de conformidade: consulta de políticas, verificação regulatória
- Busca empresarial: intranet, gestão de documentos, base de conhecimento

---

## Comparação Entre Arquiteturas

| Arquitetura | Complexidade | Precisão | Custo | Latência | Tempo de Impl. | Melhor Para |
|---|---|---|---|---|---|---|
| **Naive RAG** | Baixa | Média | Baixo | Baixa | Dias | Corpora estáticos e estreitos |
| **Agentic RAG** | Alta | Alta | Alto | Alta | Semanas/meses | Fluxos complexos e empresas |
| **Modular RAG** | Alta | Alta | Alto | Alta | Semanas/meses | Workflows empresariais |
| **Corrective RAG** | Alta | Muito alta | Alto | Alta | Semanas | Domínios de alto risco |
| **HyDE RAG** | Média | Média-alta | Médio | Média | 1–2 semanas | Busca técnica |
| **Contextual RAG** | Média | Média-alta | Médio | Média | 1–2 semanas | Diálogo multi-turno |
| **Hybrid RAG** | Média | Média-alta | Médio | Média | 1–2 semanas | Precisão e recall mistos |
| **Graph RAG** | Alta | Alta | Alto | Média | Semanas | Raciocínio relacional |

Escolha uma arquitetura com base no modo de falha:

- **Naive RAG** resolve simplicidade
- **Agentic RAG** resolve autonomia
- **HyDE RAG** resolve incompatibilidade de vocabulário
- **Graph RAG** resolve raciocínio relacional
- **Corrective RAG** resolve verificação
- **Contextual RAG** resolve memória
- **Modular RAG** resolve composição de workflows empresariais
- **Hybrid RAG** resolve o equilíbrio entre precisão e cobertura semântica

**Recursos:**
- Site do projeto de implementação RAG: https://www.neurondb.ai
- Código-fonte da implementação RAG: https://github.com/neurondb/neurondb

---

## Conclusão

RAG não é mais uma arquitetura única. Cada padrão resolve um problema específico. O sucesso em produção depende do design do pipeline, da qualidade dos dados e da disciplina de avaliação. Os sistemas mais robustos combinam múltiplos padrões RAG em uma única plataforma orquestrada — um sistema pode usar recuperação Hybrid, verificação Corrective e memória Contextual. Os sistemas RAG do futuro parecerão menos com pipelines de busca e mais com sistemas operacionais de dados distribuídos.

---

*Artigo original publicado em [DZone](https://dzone.com/articles/dont-ignore-these-rag-patterns) por Damil Shahzad (24 de abril de 2026). Tradução para o português brasileiro.*