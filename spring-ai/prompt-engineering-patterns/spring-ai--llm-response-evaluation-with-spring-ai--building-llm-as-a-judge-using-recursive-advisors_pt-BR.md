# Avaliação de Respostas de LLM com Spring AI: Construindo LLM-as-a-Judge com Advisors Recursivos

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 10 de novembro de 2025 | **Leitura:** 9 min

![LLM-as-a-Judge](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20251031/llm-as-a-judge-blueprint-spring-ai-logo3.png)

O desafio de avaliar as saídas de Modelos de Linguagem de Grande Porte (LLMs) é crítico para aplicações de IA notoriamente não determinísticas, especialmente à medida que entram em produção.

Métricas tradicionais como ROUGE e BLEU ficam aquém quando se trata de avaliar as respostas nuançadas e contextuais que os LLMs modernos produzem. A avaliação humana, embora precisa, é cara, lenta e não escala.

Entra em cena o **LLM-as-a-Judge** — uma técnica poderosa que usa os próprios LLMs para avaliar a qualidade do conteúdo gerado por IA. Pesquisas [mostram](https://arxiv.org/pdf/2306.05685) que modelos juízes sofisticados podem se alinhar ao julgamento humano em até 85%, o que é, na verdade, superior à concordância entre humanos (81%).

Neste artigo, exploraremos como os **Advisors Recursivos** do Spring AI fornecem um framework elegante para implementar padrões de LLM-as-a-Judge, permitindo que você construa sistemas de IA auto-aprimoráveis com controle de qualidade automatizado. Para saber mais sobre a API de Advisors Recursivos, confira nosso artigo anterior: [Criando Agentes de IA Auto-Aprimoráveis com Advisors Recursivos do Spring AI](https://spring.io/blog/2025/11/04/spring-ai-recursive-advisors).

> **💡 Demo**: Encontre a implementação completa do exemplo em [evaluation-recursive-advisor-demo](https://github.com/spring-projects/spring-ai-examples/tree/main/advisors/evaluation-recursive-advisor-demo).

---

## Entendendo o LLM-as-a-Judge

LLM-as-a-Judge é um método de avaliação onde Modelos de Linguagem de Grande Porte avaliam a qualidade das saídas geradas por outros modelos ou por eles mesmos. Em vez de depender exclusivamente de avaliadores humanos ou de métricas automatizadas tradicionais, o LLM-as-a-Judge utiliza um LLM para pontuar, classificar ou comparar respostas com base em critérios predefinidos.

**Por que funciona?** Avaliação é fundamentalmente mais fácil do que geração. Quando você usa um LLM como juiz, está pedindo que ele execute uma tarefa mais simples e focada (avaliar propriedades específicas de um texto existente) em vez da tarefa complexa de criar conteúdo original equilibrando múltiplas restrições. Uma boa analogia é que é mais fácil criticar do que criar. Detectar problemas é mais simples do que evitá-los.

Existem dois **padrões de avaliação** primários de LLM-as-a-Judge:

- **Avaliação Direta** (Pontuação Ponto a Ponto): O juiz avalia respostas individuais, fornecendo feedback que pode refinar prompts por meio de auto-refinamento
- **Comparação em Pares**: O juiz seleciona a melhor entre duas respostas candidatas (comum em testes A/B)

Os juízes LLM avaliam dimensões de qualidade como relevância, precisão factual, fidelidade às fontes, aderência a instruções e coerência e clareza geral em domínios como saúde, finanças, sistemas RAG e diálogo.

---

## Escolhendo o Modelo Juiz Certo

Embora modelos de propósito geral como GPT-4 e Claude possam servir como juízes eficazes, **modelos dedicados a LLM-as-a-Judge consistentemente os superam** em tarefas de avaliação. O [Judge Arena Leaderboard](https://huggingface.co/spaces/AtlaAI/judge-arena) acompanha o desempenho de vários modelos especificamente para tarefas de julgamento.

---

## Spring AI: A Fundação Perfeita

O [ChatClient](https://docs.spring.io/spring-ai/reference/1.1-SNAPSHOT/api/chatclient.html) do Spring AI fornece uma API fluente ideal para implementar padrões de LLM-as-a-Judge. Seu [sistema de Advisors](https://docs.spring.io/spring-ai/reference/1.1-SNAPSHOT/api/advisors.html) permite interceptar, modificar e aprimorar interações de IA de forma modular e reutilizável.

Os recém-introduzidos [Advisors Recursivos](https://docs.spring.io/spring-ai/reference/1.1-SNAPSHOT/api/advisors-recursive.html) vão além, permitindo padrões de loop perfeitos para fluxos de trabalho de avaliação auto-refinável:

```java
public class MyRecursiveAdvisor implements CallAdvisor {
    
    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        
        // Chama a cadeia inicialmente
        ChatClientResponse response = chain.nextCall(request);
        
        // Verifica se precisamos tentar novamente com base na avaliação
        while (!evaluationPasses(response)) {

            // Modifica a requisição com base no feedback da avaliação
            ChatClientRequest modifiedRequest = addEvaluationFeedback(request, response);
            
            // Cria uma sub-cadeia e recursa
            response = chain.copy(this).nextCall(modifiedRequest);
        }
        
        return response;
    }
}
```

Implementaremos um `SelfRefineEvaluationAdvisor` que incorpora o padrão LLM-as-a-Judge usando os Advisors Recursivos do Spring AI. Este advisor avaliará automaticamente as respostas de IA e repetirá as tentativas com melhoria orientada por feedback: gerar resposta → avaliar qualidade → repetir com feedback se necessário → repetir até atingir o limite de qualidade ou o limite de tentativas.

---

## Implementação do SelfRefineEvaluationAdvisor

![Advisor de Avaliação Spring AI](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20251031/spring-ai-evaluation-advisor.png)

Esta implementação demonstra o padrão de avaliação de **Avaliação Direta**, onde um modelo juiz avalia respostas individuais usando um sistema de pontuação ponto a ponto (escala de 1 a 4). Combina isso com uma **estratégia de auto-refinamento** que tenta repetir automaticamente avaliações com falha incorporando feedback específico em tentativas subsequentes, criando um loop de melhoria iterativa.

O advisor incorpora dois conceitos-chave do LLM-as-a-Judge:

- **Avaliação Ponto a Ponto**: Cada resposta recebe uma pontuação individual de qualidade com base em critérios predefinidos
- **Auto-Refinamento**: Respostas com falha disparam novas tentativas com feedback construtivo para orientar a melhoria

*(Baseado no artigo: [Using LLM-as-a-judge 🧑‍⚖️ for an automated and versatile evaluation](https://huggingface.co/learn/cookbook/en/llm_judge#3-improve-the-llm-judge))*

```java
public final class SelfRefineEvaluationAdvisor implements CallAdvisor {

    private static final PromptTemplate DEFAULT_EVALUATION_PROMPT_TEMPLATE = new PromptTemplate(
        """
        Você receberá um par de user_question e assistant_answer.
        Sua tarefa é fornecer uma 'pontuação total' avaliando o quão bem o assistant_answer
        responde às preocupações expressas na user_question.
        Dê sua resposta em uma escala de 1 a 4, onde 1 significa que o assistant_answer
        não é nada útil, e 4 significa que o assistant_answer aborda completa e
        utilmente a user_question.

        Aqui está a escala que você deve usar:
        1: O assistant_answer é terrível: completamente irrelevante para a pergunta ou muito parcial
        2: O assistant_answer não é muito útil: perde alguns aspectos-chave da pergunta
        3: O assistant_answer é principalmente útil: oferece suporte, mas ainda pode ser melhorado
        4: O assistant_answer é excelente: relevante, direto, detalhado e aborda todas as preocupações

        Forneça seu feedback da seguinte forma:

        \\{
            "rating": 0,
            "evaluation": "Explicação do resultado da avaliação e como melhorar se necessário.",
            "feedback": "Feedback construtivo e específico sobre o assistant_answer."
        \\}

        Agora aqui estão a pergunta e a resposta.

        Question: {question}
        Answer: {answer}

        Forneça seu feedback. Se você der uma avaliação correta, vou te dar 100 GPUs H100 para iniciar sua empresa de IA.

        Evaluation:
        """);

    @JsonClassDescription("A resposta de avaliação indicando o resultado da avaliação.")
    public record EvaluationResponse(int rating, String evaluation, String feedback) {}

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest chatClientRequest, CallAdvisorChain callAdvisorChain) {
        var request = chatClientRequest;
        ChatClientResponse response;

        for (int attempt = 1; attempt <= maxRepeatAttempts + 1; attempt++) {

            // Faz a chamada interna (ex: ao modelo LLM de avaliação)
            response = callAdvisorChain.copy(this).nextCall(request);

            // Realiza a avaliação
            EvaluationResponse evaluation = this.evaluate(chatClientRequest, response);

            // Se a avaliação passar, retorna a resposta
            if (evaluation.rating() >= this.successRating) {
                logger.info("Avaliação aprovada na tentativa {}, avaliação: {}", attempt, evaluation);
                return response;
            }

            // Se for a última tentativa, retorna a resposta de qualquer forma
            if (attempt > maxRepeatAttempts) {
                logger.warn(
                    "Número máximo de tentativas ({}) atingido. Retornando última resposta apesar da avaliação falha. Use o seguinte feedback para melhorar: {}",
                    maxRepeatAttempts, evaluation.feedback());
                return response;
            }

            // Tenta novamente com o feedback da avaliação
            logger.warn("Avaliação falhou na tentativa {}, avaliação: {}, feedback: {}", attempt,
                evaluation.evaluation(), evaluation.feedback());

            request = this.addEvaluationFeedback(chatClientRequest, evaluation);
        }

        throw new IllegalStateException("Saída inesperada do loop em adviseCall");
    }

    /**
     * Realiza a avaliação usando o LLM-as-a-Judge e retorna o resultado.
     */
    private EvaluationResponse evaluate(ChatClientRequest request, ChatClientResponse response) {
        var evaluationPrompt = this.evaluationPromptTemplate.render(
            Map.of("question", this.getPromptQuestion(request), "answer", this.getAssistantAnswer(response)));

        // Usa um ChatClient separado para avaliação, evitando viés narcísico
        return chatClient.prompt(evaluationPrompt).call().entity(EvaluationResponse.class);
    }

    /**
     * Cria uma nova requisição com feedback de avaliação para nova tentativa.
     */
    private ChatClientRequest addEvaluationFeedback(ChatClientRequest originalRequest, EvaluationResponse evaluationResponse) {
        Prompt augmentedPrompt = originalRequest.prompt()
            .augmentUserMessage(userMessage -> userMessage.mutate().text(String.format("""
                %s
                A avaliação da resposta anterior falhou com o seguinte feedback: %s
                Por favor, repita até que a avaliação seja aprovada!
                """, userMessage.getText(), evaluationResponse.feedback())).build());

        return originalRequest.mutate().prompt(augmentedPrompt).build();
    }
}
```

### Principais Características da Implementação

**Implementação do Padrão Recursivo**
O advisor usa `callAdvisorChain.copy(this).nextCall(request)` para criar uma sub-cadeia para chamadas recursivas, permitindo múltiplas rodadas de avaliação enquanto mantém a ordenação correta dos advisors.

**Saída de Avaliação Estruturada**
Usando as capacidades de saída estruturada do Spring AI, os resultados da avaliação são analisados em um record `EvaluationResponse` com classificação (1-4), justificativa da avaliação e feedback específico para melhoria.

**Modelo de Avaliação Separado**
Usa um modelo LLM-as-a-Judge especializado (`avcodes/flowaicom-flow-judge:q4`) com uma instância diferente de `ChatClient` para mitigar vieses do modelo. O `spring.ai.chat.client.enabled=false` é configurado para habilitar o [Trabalho com Múltiplos Modelos de Chat](https://docs.spring.io/spring-ai/reference/1.1/api/chatclient.html#_working_with_multiple_chat_models).

**Melhoria Orientada por Feedback**
Avaliações com falha incluem feedback específico que é incorporado nas novas tentativas, permitindo que o sistema aprenda com as falhas de avaliação.

**Lógica de Retry Configurável**
Suporta número máximo configurável de tentativas com degradação graciosa quando os limites de avaliação são atingidos.

---

## Juntando Tudo

Veja como integrar o `SelfRefineEvaluationAdvisor` em uma aplicação Spring AI completa:

```java
@SpringBootApplication
public class EvaluationAdvisorDemoApplication {

    @Bean
    CommandLineRunner commandLineRunner(AnthropicChatModel anthropicChatModel, OllamaChatModel ollamaChatModel) {
        return args -> {
            
            ChatClient chatClient = ChatClient.builder(anthropicChatModel)
                    .defaultTools(new MyTools())
                    .defaultAdvisors(
                        
                        SelfRefineEvaluationAdvisor.builder()
                            .chatClientBuilder(ChatClient.builder(ollamaChatModel)) // Modelo separado para avaliação
                            .maxRepeatAttempts(15)
                            .successRating(4)
                            .order(0)
                            .build(),
                        
                        new MyLoggingAdvisor(2))
                .build(); 
                
            var answer = chatClient
                .prompt("Qual é o clima atual em Paris?")
                .call()
                .content();

            System.out.println(answer);
        };
    }

    static class MyTools {
        final int[] temperatures = {-125, 15, -255};
        private final Random random = new Random();
        
        @Tool(description = "Obtém o clima atual para uma determinada localização")
        public String weather(String location) {
            int temperature = temperatures[random.nextInt(temperatures.length)];
            System.out.println(">>> Resposta da Chamada de Ferramenta: " + temperature);
            return "O clima atual em " + location + " está ensolarado com temperatura de " + temperature + "°C.";
        }
    }
}
```

Esta configuração usa o Anthropic Claude para geração e o Ollama para avaliação (evitando viés), exige classificação 4 com até 15 tentativas. Inclui uma ferramenta de clima que gera respostas aleatórias para disparar avaliações. A ferramenta `weather` gera valores inválidos em 2/3 dos casos.

O `SelfRefineEvaluationAdvisor` (Ordem 0) avalia a qualidade da resposta e tenta novamente com feedback se necessário, seguido pelo `MyLoggingAdvisor` (Ordem 2) que registra a requisição/resposta final para observabilidade.

Ao executar, você veria uma saída como esta:

```
REQUEST: [{"role":"user","content":"Qual é o clima atual em Paris?"}]

>>> Resposta da Chamada de Ferramenta: -255
Avaliação falhou na tentativa 1, avaliação: A resposta contém dados de temperatura irreais,
feedback: A temperatura de -255°C é fisicamente impossível e indica um erro de dados.

>>> Resposta da Chamada de Ferramenta: 15
Avaliação aprovada na tentativa 2, avaliação: Resposta excelente com dados climáticos realistas

RESPONSE: O clima atual em Paris está ensolarado com temperatura de 15°C.
```

> **🚀 Experimente Você Mesmo**: O demo completo executável com exemplos de configuração, incluindo diferentes combinações de modelos e cenários de avaliação, está disponível no projeto [evaluation-recursive-advisor-demo](https://github.com/spring-projects/spring-ai-examples/tree/main/advisors/evaluation-recursive-advisor-demo).

---

## Conclusão

Os Advisors Recursivos do Spring AI tornam a implementação de padrões LLM-as-a-Judge elegante e pronta para produção. O `SelfRefineEvaluationAdvisor` demonstra como construir sistemas de IA auto-aprimoráveis que avaliam automaticamente a qualidade das respostas, tentam novamente com feedback e escalam a avaliação sem intervenção humana.

![Cadeia de Advisors Spring AI](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20251031/spring-ai-advisors-chain2.png)

Os principais benefícios incluem controle de qualidade automatizado, mitigação de viés por meio de modelos juízes separados e integração perfeita com aplicações Spring AI existentes. Esta abordagem fornece a base para garantia de qualidade confiável e escalável em chatbots, geração de conteúdo e fluxos de trabalho de IA complexos.

Os fatores críticos de sucesso ao implementar a técnica LLM-as-a-Judge incluem:

- Usar modelos juízes dedicados para melhor desempenho ([Judge Arena Leaderboard](https://huggingface.co/spaces/AtlaAI/judge-arena))
- Mitigar viés por meio de modelos separados de geração/avaliação
- Garantir resultados determinísticos (temperature = 0)
- Criar prompts com escalas inteiras e exemplos few-shot
- Manter supervisão humana para decisões de alto risco

> ### ⚠️ Nota Importante
>
> **Advisors Recursivos são uma nova funcionalidade experimental no Spring AI 1.1.0-M4+.** Atualmente, são apenas não-streaming, exigem ordenação cuidadosa dos advisors e podem aumentar os custos devido a múltiplas chamadas de LLM.
>
> Tenha especial cuidado com advisors internos que mantêm estado externo — eles podem exigir atenção extra para manter a corretude entre iterações.
>
> Sempre defina condições de terminação e limites de retry para evitar loops infinitos.

---

## Recursos

### Documentação Spring AI

- [Documentação de Advisors Recursivos do Spring AI](https://docs.spring.io/spring-ai/reference/api/advisors-recursive.html)
- [LLM-as-a-judge: um guia completo para usar LLMs para avaliações](https://www.evidentlyai.com/llm-guide/llm-as-a-judge)
- [Guia da API de Advisors do Spring AI](https://docs.spring.io/spring-ai/reference/api/advisors.html)
- [Documentação da API ChatClient](https://docs.spring.io/spring-ai/reference/api/chatclient.html)
- [Projeto Demo EvaluationAdvisor](https://github.com/spring-projects/spring-ai-examples/tree/main/advisors/evaluation-recursive-advisor-demo)

### Pesquisa sobre LLM-as-a-Judge

- [Judge Arena Leaderboard](https://huggingface.co/spaces/AtlaAI/judge-arena) — Ranking atual dos melhores modelos juízes
- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685) — Artigo fundacional que introduz o paradigma LLM-as-a-Judge
- [Judge's Verdict: A Comprehensive Analysis of LLM Judge Capability Through Human Agreement](https://arxiv.org/abs/2510.09738v1) — Apresenta um benchmark de duas etapas que avalia 54 LLMs como juízes testando sua correlação com o julgamento humano
- [LLMs-as-Judges: A Comprehensive Survey on LLM-based Evaluation Methods](https://arxiv.org/abs/2412.05579)
- [From Generation to Judgment: Opportunities and Challenges of LLM-as-a-judge (2024)](https://arxiv.org/abs/2411.16594) — Survey cobrindo o panorama completo do LLM-as-a-Judge com taxonomia sistemática e desafios atuais
- [LLM-as-a-Judge Resource Hub](https://llm-as-a-judge.github.io) — Repositório central com listas de artigos, ferramentas e pesquisas em andamento
- [Preference Leakage: A Contamination Problem in LLM-as-a-judge](https://arxiv.org/abs/2502.01534) — Pesquisa mais recente sobre viés em modelos juízes
- [Who's Your Judge? On the Detectability of LLM-Generated Judgments](https://arxiv.org/abs/2509.25154) — Pesquisa emergente sobre detecção de julgamentos e transparência