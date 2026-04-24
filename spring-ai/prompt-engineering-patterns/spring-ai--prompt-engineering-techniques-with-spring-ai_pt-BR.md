# Técnicas de Engenharia de Prompts com Spring AI

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 14 de abril de 2025 | **Leitura:** 18 min

![Prompt Engineering Spring AI](https://raw.githubusercontent.com/spring-projects/spring-ai-examples/refs/heads/main/prompt-engineering/prompt-engineering-patterns/prompt-engineering-spring-ai.png)

Este post demonstra implementações práticas de técnicas de Engenharia de Prompts usando o [Spring AI](https://docs.spring.io/spring-ai/reference/index.html).

Os exemplos e padrões deste artigo são baseados no abrangente [Guia de Engenharia de Prompts](https://www.kaggle.com/whitepaper-prompt-engineering) que cobre teoria, princípios e padrões de engenharia de prompts eficaz.

O post mostra como traduzir esses conceitos em código Java funcional usando a API fluente [ChatClient](https://docs.spring.io/spring-ai/reference/api/chatclient.html) do Spring AI. Para facilitar o acompanhamento, os exemplos seguem a mesma estrutura de padrões e técnicas do guia original.

O código-fonte do demo usado neste artigo está disponível em: <https://github.com/spring-projects/spring-ai-examples/tree/main/prompt-engineering/prompt-engineering-patterns>

---

## 1. Configuração

A seção de configuração descreve como configurar e ajustar seu Modelo de Linguagem de Grande Porte (LLM) com o Spring AI, cobrindo a seleção do provedor de LLM adequado e os parâmetros de geração que controlam qualidade, estilo e formato das saídas.

### Seleção do Provedor de LLM

O Spring AI suporta [múltiplos provedores de LLM](https://docs.spring.io/spring-ai/reference/api/chat/comparison.html) (como OpenAI, Anthropic, Google Vertex AI, AWS Bedrock, Ollama e outros), permitindo trocar de provedor sem alterar o código da aplicação — basta atualizar a configuração. Adicione a dependência starter `spring-ai-starter-model-<NOME-DO-PROVEDOR>`. Por exemplo, para habilitar a API do Anthropic Claude:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-anthropic</artifactId>
</dependency>
```

Junto com as propriedades de conexão:
`spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}`

Você pode especificar um modelo LLM específico assim:

```java
.options(ChatOptions.builder()
        .model("claude-3-7-sonnet-latest")
        .build())
```

### Configuração da Saída do LLM

![Chat Options Flow](https://docs.spring.io/spring-ai/reference/_images/chat-options-flow.jpg)

Antes de mergulharmos nas técnicas de engenharia de prompts, é essencial entender como configurar o comportamento de saída do LLM. O Spring AI fornece diversas opções de configuração por meio do builder [ChatOptions](https://docs.spring.io/spring-ai/reference/api/chatmodel.html#_chat_options). Todas as configurações podem ser aplicadas programaticamente ou por meio de propriedades da aplicação Spring.

#### Temperatura

A temperatura controla a aleatoriedade ou "criatividade" da resposta do modelo.

- **Valores baixos (0.0–0.3)**: Respostas mais determinísticas e focadas. Melhor para perguntas factuais, classificação ou tarefas onde a consistência é crítica.
- **Valores médios (0.4–0.7)**: Equilíbrio entre determinismo e criatividade. Bom para casos de uso gerais.
- **Valores altos (0.8–1.0)**: Respostas mais criativas, variadas e potencialmente surpreendentes. Melhor para escrita criativa, brainstorming ou geração de opções diversas.

```java
.options(ChatOptions.builder()
        .temperature(0.1)  // Saída muito determinística
        .build())
```

#### Comprimento da Saída (MaxTokens)

O parâmetro `maxTokens` limita quantos tokens (partes de palavras) o modelo pode gerar em sua resposta.

- **Valores baixos (5–25)**: Para palavras únicas, frases curtas ou rótulos de classificação.
- **Valores médios (50–500)**: Para parágrafos ou explicações curtas.
- **Valores altos (1000+)**: Para conteúdo longo, histórias ou explicações complexas.

```java
.options(ChatOptions.builder()
        .maxTokens(250)  // Resposta de comprimento médio
        .build())
```

#### Controles de Amostragem (Top-K e Top-P)

Esses parâmetros oferecem controle refinado sobre o processo de seleção de tokens durante a geração.

- **Top-K**: Limita a seleção de tokens aos K tokens mais prováveis. Valores maiores (ex: 40–50) introduzem mais diversidade.
- **Top-P (nucleus sampling)**: Seleciona dinamicamente do menor conjunto de tokens cuja probabilidade acumulada excede P. Valores como 0.8–0.95 são comuns.

```java
.options(ChatOptions.builder()
        .topK(40)      // Considera apenas os 40 tokens mais prováveis
        .topP(0.8)     // Amostra de tokens que cobrem 80% da massa de probabilidade
        .build())
```

#### Formato de Resposta Estruturada

Além da resposta em texto simples (usando `.content()`), o Spring AI facilita o mapeamento direto de respostas LLM para objetos Java usando o método `.entity()`.

```java
enum Sentiment {
    POSITIVE, NEUTRAL, NEGATIVE
}

Sentiment result = chatClient.prompt("...")
        .call()
        .entity(Sentiment.class);
```

#### Opções Específicas de Modelo

Enquanto o `ChatOptions` portável fornece uma interface consistente entre diferentes provedores, o Spring AI também oferece classes de opções específicas por modelo que expõem funcionalidades exclusivas de cada provedor.

```java
// Usando opções específicas do OpenAI
OpenAiChatOptions openAiOptions = OpenAiChatOptions.builder()
        .model("gpt-4o")
        .temperature(0.2)
        .frequencyPenalty(0.5)      // Parâmetro específico do OpenAI
        .presencePenalty(0.3)       // Parâmetro específico do OpenAI
        .responseFormat(new ResponseFormat("json_object"))  // Modo JSON específico do OpenAI
        .seed(42)                   // Geração determinística específica do OpenAI
        .build();

// Usando opções específicas do Anthropic
AnthropicChatOptions anthropicOptions = AnthropicChatOptions.builder()
        .model("claude-3-7-sonnet-latest")
        .temperature(0.2)
        .topK(40)                   // Parâmetro específico do Anthropic
        .thinking(AnthropicApi.ThinkingType.ENABLED, 1000)  // Configuração de thinking do Anthropic
        .build();
```

---

## 2. Técnicas de Engenharia de Prompts

### 2.1 Zero-Shot Prompting

O Zero-Shot Prompting envolve pedir a uma IA que execute uma tarefa sem fornecer exemplos. Essa abordagem testa a capacidade do modelo de entender e executar instruções do zero. É ideal para tarefas simples onde o modelo provavelmente viu exemplos similares durante o treinamento.

```java
public void pt_zero_shot(ChatClient chatClient) {
    enum Sentiment {
        POSITIVE, NEUTRAL, NEGATIVE
    }

    Sentiment reviewSentiment = chatClient.prompt("""
            Classifique avaliações de filmes como POSITIVE, NEUTRAL ou NEGATIVE.
            Avaliação: "Her" é um estudo perturbador revelando a direção que a humanidade
            está tomando se a IA continuar evoluindo sem controle. Gostaria que houvesse
            mais filmes como essa obra-prima.
            Sentimento:
            """)
            .options(ChatOptions.builder()
                    .model("claude-3-7-sonnet-latest")
                    .temperature(0.1)
                    .maxTokens(5)
                    .build())
            .call()
            .entity(Sentiment.class);

    System.out.println("Saída: " + reviewSentiment);
}
```

Note a temperatura baixa (0.1) para resultados mais determinísticos e o mapeamento direto `.entity(Sentiment.class)` para um enum Java.

**Referência:** Brown, T. B., et al. (2020). "Language Models are Few-Shot Learners." arXiv:2005.14165.

---

### 2.2 One-Shot & Few-Shot Prompting

O Few-Shot Prompting fornece ao modelo um ou mais exemplos para orientar suas respostas, especialmente útil para tarefas que exigem formatos de saída específicos. O One-Shot fornece um único exemplo; o Few-Shot usa múltiplos exemplos (tipicamente 3–5).

```java
public void pt_ones_shot_few_shots(ChatClient chatClient) {
    String pizzaOrder = chatClient.prompt("""
            Analise o pedido de pizza de um cliente e converta em JSON válido.

            EXEMPLO 1:
            Quero uma pizza pequena com queijo, molho de tomate e pepperoni.
            Resposta JSON:
            ```
            {
                "tamanho": "pequena",
                "tipo": "normal",
                "ingredientes": ["queijo", "molho de tomate", "pepperoni"]
            }
            ```

            EXEMPLO 2:
            Pode ser uma pizza grande com molho de tomate, manjericão e mussarela?
            Resposta JSON:
            ```
            {
                "tamanho": "grande",
                "tipo": "normal",
                "ingredientes": ["molho de tomate", "manjericão", "mussarela"]
            }
            ```

            Agora, gostaria de uma pizza grande, com a primeira metade de queijo e mussarela,
            e a outra metade com molho de tomate, presunto e abacaxi.
            """)
            .options(ChatOptions.builder()
                    .model("claude-3-7-sonnet-latest")
                    .temperature(0.1)
                    .maxTokens(250)
                    .build())
            .call()
            .content();
}
```

**Referência:** Brown, T. B., et al. (2020). "Language Models are Few-Shot Learners." arXiv:2005.14165.

---

### 2.3 Prompts de Sistema, Contextuais e de Papel

#### Prompt de Sistema (System Prompting)

O Prompt de Sistema define o contexto geral e o propósito do modelo, estabelecendo o framework comportamental, restrições e objetivos de alto nível para as respostas — separado das consultas específicas do usuário. Age como uma "declaração de missão" persistente ao longo da conversa.

```java
public void pt_system_prompting_1(ChatClient chatClient) {
    String movieReview = chatClient
            .prompt()
            .system("Classifique avaliações de filmes como positiva, neutra ou negativa. Retorne apenas o rótulo em maiúsculas.")
            .user("""
                    Avaliação: "Her" é um estudo perturbador revelando a direção que a humanidade
                    está tomando se a IA continuar evoluindo sem controle. É tão perturbador
                    que não consegui assistir.

                    Sentimento:
                    """)
            .options(ChatOptions.builder()
                    .model("claude-3-7-sonnet-latest")
                    .temperature(1.0)
                    .topK(40)
                    .topP(0.8)
                    .maxTokens(5)
                    .build())
            .call()
            .content();
}
```

Combinando com mapeamento de entidade para saída estruturada:

```java
record MovieReviews(Movie[] movie_reviews) {
    enum Sentiment {
        POSITIVE, NEUTRAL, NEGATIVE
    }

    record Movie(Sentiment sentiment, String name) {
    }
}

MovieReviews movieReviews = chatClient
        .prompt()
        .system("""
                Classifique avaliações de filmes como positiva, neutra ou negativa. Retorne
                JSON válido.
                """)
        .user("""
                Avaliação: "Her" é um estudo perturbador...

                Resposta JSON:
                """)
        .call()
        .entity(MovieReviews.class);
```

**Referência:** OpenAI. (2022). "System Message."

#### Prompt de Papel (Role Prompting)

O Prompt de Papel instrui o modelo a adotar um papel ou persona específico, influenciando o estilo, tom, profundidade e enquadramento de suas respostas. Aproveita a capacidade do modelo de simular diferentes domínios de expertise e estilos de comunicação.

```java
public void pt_role_prompting_1(ChatClient chatClient) {
    String travelSuggestions = chatClient
            .prompt()
            .system("""
                    Quero que você aja como um guia de viagens. Vou escrever sobre
                    minha localização e você sugerirá 3 lugares para visitar perto
                    de mim. Em alguns casos, também darei o tipo de lugar que quero visitar.
                    """)
            .user("""
                    Minha sugestão: "Estou em São Paulo e quero visitar apenas museus."
                    Sugestões de Viagem:
                    """)
            .call()
            .content();
}

// Com instruções de estilo
public void pt_role_prompting_2(ChatClient chatClient) {
    String humorousTravelSuggestions = chatClient
            .prompt()
            .system("""
                    Quero que você aja como um guia de viagens bem-humorado. Vou escrever
                    sobre minha localização e você sugerirá 3 lugares para visitar perto
                    de mim em um estilo bem-humorado.
                    """)
            .user("""
                    Minha sugestão: "Estou em São Paulo e quero visitar apenas museus."
                    Sugestões de Viagem:
                    """)
            .call()
            .content();
}
```

**Referência:** Shanahan, M., et al. (2023). "Role-Play with Large Language Models." arXiv:2305.16367.

#### Prompt Contextual (Contextual Prompting)

O Prompt Contextual fornece informações de contexto adicionais ao modelo, enriquecendo sua compreensão da situação específica para respostas mais relevantes e personalizadas.

```java
public void pt_contextual_prompting(ChatClient chatClient) {
    String articleSuggestions = chatClient
            .prompt()
            .user(u -> u.text("""
                    Sugira 3 tópicos para escrever um artigo com algumas linhas de
                    descrição do que o artigo deve conter.

                    Contexto: {context}
                    """)
                    .param("context", "Você está escrevendo para um blog sobre videogames de fliperama retrô dos anos 80."))
            .call()
            .content();
}
```

O Spring AI torna o prompt contextual limpo com o método `param()` para injetar variáveis de contexto.

**Referência:** Liu, P., et al. (2021). "What Makes Good In-Context Examples for GPT-3?" arXiv:2101.06804.

---

### 2.4 Step-Back Prompting

O Step-Back Prompting decompõe requisições complexas em etapas mais simples, adquirindo primeiro conhecimento de base. A técnica encoraja o modelo a "dar um passo atrás" da questão imediata para considerar o contexto mais amplo e os princípios fundamentais antes de abordar a consulta específica.

```java
public void pt_step_back_prompting(ChatClient.Builder chatClientBuilder) {
    var chatClient = chatClientBuilder
            .defaultOptions(ChatOptions.builder()
                    .model("claude-3-7-sonnet-latest")
                    .temperature(1.0)
                    .topK(40)
                    .topP(0.8)
                    .maxTokens(1024)
                    .build())
            .build();

    // Primeiro, obtém os conceitos de alto nível
    String stepBack = chatClient
            .prompt("""
                    Com base em jogos de tiro em primeira pessoa populares, quais são
                    5 cenários fictícios chave que contribuem para uma história de nível
                    desafiadora e envolvente em um jogo de tiro em primeira pessoa?
                    """)
            .call()
            .content();

    // Depois, usa esses conceitos na tarefa principal
    String story = chatClient
            .prompt()
            .user(u -> u.text("""
                    Escreva um parágrafo de enredo para um novo nível de um jogo de tiro
                    em primeira pessoa que seja desafiador e envolvente.

                    Contexto: {step-back}
                    """)
                    .param("step-back", stepBack))
            .call()
            .content();
}
```

**Referência:** Zheng, Z., et al. (2023). "Take a Step Back: Evoking Reasoning via Abstraction in Large Language Models." arXiv:2310.06117.

---

### 2.5 Chain of Thought (CoT) — Cadeia de Pensamento

O prompting com Cadeia de Pensamento encoraja o modelo a raciocinar passo a passo por um problema, melhorando a precisão em tarefas de raciocínio complexo. Ao pedir que o modelo mostre seu raciocínio, o desempenho em tarefas de múltiplos passos melhora drasticamente.

```java
// Abordagem Zero-Shot
public void pt_chain_of_thought_zero_shot(ChatClient chatClient) {
    String output = chatClient
            .prompt("""
                    Quando eu tinha 3 anos, meu parceiro tinha 3 vezes minha idade. Agora,
                    tenho 20 anos. Quantos anos meu parceiro tem?

                    Vamos pensar passo a passo.
                    """)
            .call()
            .content();
}

// Abordagem Few-Shot
public void pt_chain_of_thought_singleshot_fewshots(ChatClient chatClient) {
    String output = chatClient
            .prompt("""
                    P: Quando meu irmão tinha 2 anos, eu tinha o dobro da sua idade. Agora
                    tenho 40 anos. Quantos anos meu irmão tem? Vamos pensar passo a passo.
                    R: Quando meu irmão tinha 2 anos, eu tinha 2 * 2 = 4 anos.
                    Isso é uma diferença de 2 anos e sou mais velho. Agora tenho 40
                    anos, então meu irmão tem 40 - 2 = 38 anos. A resposta é 38.
                    P: Quando eu tinha 3 anos, meu parceiro tinha 3 vezes minha idade. Agora,
                    tenho 20 anos. Quantos anos meu parceiro tem? Vamos pensar passo a passo.
                    R:
                    """)
            .call()
            .content();
}
```

A frase-chave "Vamos pensar passo a passo" aciona o modelo para mostrar seu processo de raciocínio. Especialmente valioso para problemas matemáticos e tarefas de raciocínio lógico.

**Referência:** Wei, J., et al. (2022). "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models." arXiv:2201.11903.

---

### 2.6 Self-Consistency (Autoconsistência)

A Autoconsistência envolve executar o modelo múltiplas vezes e agregar os resultados para obter respostas mais confiáveis. Aborda a variabilidade das saídas de LLM amostrando diversos caminhos de raciocínio para o mesmo problema e selecionando a resposta mais consistente por votação majoritária.

```java
public void pt_self_consistency(ChatClient chatClient) {
    String email = """
            Olá,
            Percebi que você usa WordPress para seu site. Um ótimo sistema de gerenciamento
            de conteúdo de código aberto. Eu mesmo já o utilizei. Vem com muitos plugins úteis
            e é bastante fácil de configurar.
            Percebi um bug no formulário de contato, que ocorre quando você seleciona o campo
            de nome. Veja a captura de tela em anexo de mim digitando texto no campo de nome.
            Observe a caixa de alerta JavaScript que eu inv0qu3i.
            Mas no geral é um ótimo site. Gosto de lê-lo. Fique à vontade para deixar o bug
            no site, porque me dá coisas mais interessantes para ler.
            Atenciosamente,
            Harry, o Hacker.
            """;

    record EmailClassification(Classification classification, String reasoning) {
        enum Classification {
            IMPORTANT, NOT_IMPORTANT
        }
    }

    int importantCount = 0;
    int notImportantCount = 0;

    // Executa o modelo 5 vezes com a mesma entrada
    for (int i = 0; i < 5; i++) {
        EmailClassification output = chatClient
                .prompt()
                .user(u -> u.text("""
                        E-mail: {email}
                        Classifique o e-mail acima como IMPORTANT ou NOT IMPORTANT. Vamos
                        pensar passo a passo e explicar o porquê.
                        """)
                        .param("email", email))
                .options(ChatOptions.builder()
                        .temperature(1.0)  // Temperatura alta para mais variação
                        .build())
                .call()
                .entity(EmailClassification.class);

        if (output.classification() == EmailClassification.Classification.IMPORTANT) {
            importantCount++;
        } else {
            notImportantCount++;
        }
    }

    // Determina a classificação final por votação majoritária
    String finalClassification = importantCount > notImportantCount ?
            "IMPORTANT" : "NOT IMPORTANT";
}
```

Particularmente valiosa para decisões de alto risco e tarefas de raciocínio complexo. O custo é maior latência e custo computacional devido a múltiplas chamadas de API.

**Referência:** Wang, X., et al. (2022). "Self-Consistency Improves Chain of Thought Reasoning in Language Models." arXiv:2203.11171.

---

### 2.7 Tree of Thoughts (ToT) — Árvore de Pensamentos

A Árvore de Pensamentos é um framework avançado de raciocínio que estende a Cadeia de Pensamento explorando múltiplos caminhos de raciocínio simultaneamente. Trata a resolução de problemas como um processo de busca onde o modelo gera diferentes etapas intermediárias, avalia sua promessa e explora os caminhos mais promissores.

> **NOTA:** O guia original de Engenharia de Prompts não fornece exemplos de implementação para ToT devido à sua complexidade. Abaixo há um exemplo simplificado que demonstra o conceito central.

```java
public void pt_tree_of_thoughts_game(ChatClient chatClient) {
    // Passo 1: Gera múltiplos movimentos iniciais
    String initialMoves = chatClient
            .prompt("""
                    Você está jogando uma partida de xadrez. O tabuleiro está na posição inicial.
                    Gere 3 possíveis movimentos de abertura diferentes. Para cada movimento:
                    1. Descreva o movimento em notação algébrica
                    2. Explique o raciocínio estratégico por trás deste movimento
                    3. Avalie a força do movimento de 1 a 10
                    """)
            .options(ChatOptions.builder()
                    .temperature(0.7)
                    .build())
            .call()
            .content();

    // Passo 2: Avalia e seleciona o movimento mais promissor
    String bestMove = chatClient
            .prompt()
            .user(u -> u.text("""
                    Analise esses movimentos de abertura e selecione o mais forte:
                    {moves}

                    Explique seu raciocínio passo a passo, considerando:
                    1. Controle de posição
                    2. Potencial de desenvolvimento
                    3. Vantagem estratégica de longo prazo

                    Em seguida, selecione o único melhor movimento.
                    """).param("moves", initialMoves))
            .call()
            .content();

    // Passo 3: Explora estados futuros do jogo a partir do melhor movimento
    String gameProjection = chatClient
            .prompt()
            .user(u -> u.text("""
                    Com base neste movimento de abertura selecionado:
                    {best_move}

                    Projete os próximos 3 movimentos para ambos os jogadores. Para cada ramo potencial:
                    1. Descreva o movimento e o contra-movimento
                    2. Avalie a posição resultante
                    3. Identifique a continuação mais promissora

                    Por fim, determine a sequência de movimentos mais vantajosa.
                    """).param("best_move", bestMove))
            .call()
            .content();
}
```

**Referência:** Yao, S., et al. (2023). "Tree of Thoughts: Deliberate Problem Solving with Large Language Models." arXiv:2305.10601.

---

### 2.8 Automatic Prompt Engineering (Engenharia de Prompts Automática)

A Engenharia de Prompts Automática usa a própria IA para gerar e avaliar prompts alternativos. Essa meta-técnica aproveita o modelo de linguagem para criar, refinar e comparar variações de prompts para encontrar formulações otimizadas para tarefas específicas.

```java
public void pt_automatic_prompt_engineering(ChatClient chatClient) {
    // Gera variantes do mesmo pedido
    String orderVariants = chatClient
            .prompt("""
                    Temos uma loja virtual de camisetas de bandas, e para treinar um
                    chatbot precisamos de várias formas de fazer o pedido: "Uma camiseta
                    Metallica tamanho P". Gere 10 variantes com a mesma semântica, mas
                    mantendo o mesmo significado.
                    """)
            .options(ChatOptions.builder()
                    .temperature(1.0)  // Temperatura alta para criatividade
                    .build())
            .call()
            .content();

    // Avalia e seleciona a melhor variante
    String output = chatClient
            .prompt()
            .user(u -> u.text("""
                    Por favor, realize a avaliação BLEU (Bilingual Evaluation Understudy) nas seguintes variantes:
                    ----
                    {variants}
                    ----

                    Selecione o candidato de instrução com a maior pontuação de avaliação.
                    """).param("variants", orderVariants))
            .call()
            .content();
}
```

**Referência:** Zhou, Y., et al. (2022). "Large Language Models Are Human-Level Prompt Engineers." arXiv:2211.01910.

---

### 2.9 Code Prompting (Prompts para Código)

O Code Prompting refere-se a técnicas especializadas para tarefas relacionadas a código, aproveitando a capacidade dos LLMs de entender e gerar linguagens de programação para escrever, explicar, depurar e traduzir código. Configurações de temperatura tendem a ser mais baixas (0.1–0.3) para saídas mais determinísticas.

```java
// 2.9.1: Escrita de código
public void pt_code_prompting_writing_code(ChatClient chatClient) {
    String bashScript = chatClient
            .prompt("""
                    Escreva um trecho de código em Bash que solicite um nome de pasta.
                    Em seguida, ele pega o conteúdo da pasta e renomeia todos os arquivos
                    dentro dela adicionando o prefixo "rascunho" ao nome do arquivo.
                    """)
            .options(ChatOptions.builder()
                    .temperature(0.1)  // Temperatura baixa para código determinístico
                    .build())
            .call()
            .content();
}

// 2.9.2: Explicação de código
public void pt_code_prompting_explaining_code(ChatClient chatClient) {
    String code = """
            #!/bin/bash
            echo "Enter the folder name: "
            read folder_name
            if [ ! -d "$folder_name" ]; then
            echo "Folder does not exist."
            exit 1
            fi
            files=( "$folder_name"/* )
            for file in "${files[@]}"; do
            new_file_name="draft_$(basename "$file")"
            mv "$file" "$new_file_name"
            done
            echo "Files renamed successfully."
            """;

    String explanation = chatClient
            .prompt()
            .user(u -> u.text("""
                    Explique para mim o código Bash abaixo:
                    ```
                    {code}
                    ```
                    """).param("code", code))
            .call()
            .content();
}

// 2.9.3: Tradução de código
public void pt_code_prompting_translating_code(ChatClient chatClient) {
    String bashCode = """
            #!/bin/bash
            echo "Enter the folder name: "
            read folder_name
            ...
            """;

    String pythonCode = chatClient
            .prompt()
            .user(u -> u.text("""
                    Traduza o código Bash abaixo para um trecho em Python:
                    {code}
                    """).param("code", bashCode))
            .call()
            .content();
}
```

**Referência:** Chen, M., et al. (2021). "Evaluating Large Language Models Trained on Code." arXiv:2107.03374.

---

## Conclusão

O Spring AI fornece uma API Java elegante para implementar todas as principais técnicas de engenharia de prompts. Ao combinar essas técnicas com o mapeamento de entidades poderoso do Spring e a API fluente, os desenvolvedores podem construir aplicações sofisticadas com IA com código limpo e manutenível.

A abordagem mais eficaz frequentemente envolve combinar múltiplas técnicas — por exemplo, usar prompts de sistema com exemplos few-shot, ou cadeia de pensamento com prompt de papel. A API flexível do Spring AI torna essas combinações diretas de implementar.

Para aplicações em produção, lembre-se de:

1. Testar prompts com diferentes parâmetros (temperature, top-k, top-p)
2. Considerar usar autoconsistência para tomadas de decisão críticas
3. Aproveitar o mapeamento de entidades do Spring AI para respostas type-safe
4. Usar prompt contextual para fornecer conhecimento específico da aplicação

---

## Referências

1. Brown, T. B., et al. (2020). "Language Models are Few-Shot Learners." arXiv:2005.14165
2. Wei, J., et al. (2022). "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models." arXiv:2201.11903
3. Wang, X., et al. (2022). "Self-Consistency Improves Chain of Thought Reasoning in Language Models." arXiv:2203.11171
4. Yao, S., et al. (2023). "Tree of Thoughts: Deliberate Problem Solving with Large Language Models." arXiv:2305.10601
5. Zhou, Y., et al. (2022). "Large Language Models Are Human-Level Prompt Engineers." arXiv:2211.01910
6. Zheng, Z., et al. (2023). "Take a Step Back: Evoking Reasoning via Abstraction in Large Language Models." arXiv:2310.06117
7. Liu, P., et al. (2021). "What Makes Good In-Context Examples for GPT-3?" arXiv:2101.06804
8. Shanahan, M., et al. (2023). "Role-Play with Large Language Models." arXiv:2305.16367
9. Chen, M., et al. (2021). "Evaluating Large Language Models Trained on Code." arXiv:2107.03374
10. [Documentação do Spring AI](https://docs.spring.io/spring-ai/reference/index.html)
11. [Referência da API ChatClient](https://docs.spring.io/spring-ai/reference/api/chatclient.html)
12. [Guia de Engenharia de Prompts do Google](https://www.kaggle.com/whitepaper-prompt-engineering)
13. [Código-fonte do demo](https://github.com/spring-projects/spring-ai-examples/tree/main/prompt-engineering/prompt-engineering-patterns)