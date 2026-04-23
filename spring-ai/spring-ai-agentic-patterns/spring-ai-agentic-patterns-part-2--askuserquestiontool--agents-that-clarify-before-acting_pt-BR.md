# Padrões Agentivos com Spring AI (Parte 2): AskUserQuestionTool — Agentes que Esclarecem Antes de Agir

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 16 de janeiro de 2026 | **Leitura:** 5 min

![AskUserQuestionTool](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260112/ask-user-question-tool-logo.png)

As interações tradicionais com IA seguem um padrão comum: você fornece um prompt, a IA faz suposições e produz uma resposta. Quando essas suposições não correspondem às suas necessidades, você fica iterando por correções. Cada suposição gera retrabalho — desperdiçando tempo e contexto.

E se o seu agente de IA pudesse fazer perguntas de esclarecimento antes de responder?

O **AskUserQuestionTool** resolve isso. Ele permite que o agente de IA faça perguntas de esclarecimento *antes* de responder, coleta requisitos de forma interativa e cria uma especificação alinhada com suas necessidades reais desde o início.

A implementação do Spring AI traz esse padrão interativo para o ecossistema Java, garantindo portabilidade de LLM — defina seus manipuladores de perguntas uma vez e use-os com OpenAI, Anthropic, Google Gemini ou qualquer outro modelo suportado.

**Esta é a Parte 2 da nossa série Padrões Agentivos com Spring AI.** Na [Parte 1](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills), exploramos as Agent Skills — capacidades modulares que estendem a funcionalidade da IA. Agora examinamos o AskUserQuestionTool, que transforma agentes de IA em parceiros colaborativos que coletam requisitos de forma interativa.

🚀 **Quer ir direto ao ponto?** Pule para a seção de [Primeiros Passos](#primeiros-passos).

---

## Como o AskUserQuestionTool Funciona

O [AskUserQuestionTool](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/AskUserQuestionTool.md), parte do toolkit [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils), é uma implementação portável do Spring AI baseada na [ferramenta AskUserQuestion do Claude Code](https://platform.claude.com/docs/en/agent-sdk/user-input#question-format), que permite aos agentes de IA fazerem perguntas de múltipla escolha aos usuários durante a execução.

![Fluxo do AskUserQuestionTool](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260112/ask-user-question-tool-flow.png)

A ferramenta segue um fluxo de pergunta e resposta:

1. **A IA gera perguntas** — O agente determina que precisa de informações e constrói perguntas (cada uma contendo texto da pergunta, cabeçalho, 2 a 4 opções e flag de múltipla seleção) e chama a função da ferramenta `askUserQuestion`
2. **O usuário fornece respostas** — Seu manipulador personalizado recebe essas perguntas, as apresenta através da sua interface, coleta as respostas e as retorna à IA
3. **Perguntas adicionais** — Repita os passos 1 e 2 se necessário para coletar mais informações do usuário
4. **A IA continua com contexto** — O agente usa as respostas para fornecer soluções personalizadas

Cada pergunta suporta:

- **Seleção única ou múltipla** — Escolha uma opção ou combine várias
- **Entrada de texto livre** — Os usuários sempre podem fornecer texto personalizado além das opções predefinidas
- **Contexto rico** — Cada opção inclui uma descrição explicando implicações e trocas

💡 **Portabilidade e Agnóstico de Modelo — Sem Dependência de Fornecedor** — Diferente de implementações vinculadas a plataformas LLM específicas, esta implementação do Spring AI funciona com vários provedores de LLM, permitindo trocar de modelo sem reescrever código ou manipuladores de perguntas.

💡 **Relação com o MCP Elicitation** — O AskUserQuestionTool serve como uma abordagem local ao agente para entrada interativa do usuário, conceitualmente semelhante à capacidade de [MCP Elicitation](https://modelcontextprotocol.io/specification/2025-03-26/client/elicitation). Enquanto o MCP Elicitation permite que servidores MCP solicitem entrada estruturada do usuário via schemas JSON, o AskUserQuestionTool fornece o mesmo padrão interativo diretamente dentro do seu agente sem exigir um servidor MCP. O Spring AI também oferece [suporte completo ao MCP Elicitation](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-client.html#_mcpelicitation) via a anotação `@McpElicitation` para cenários orientados pelo servidor.

---

## Exemplo: Assistente de Recomendação de Viagens

Veja como a ferramenta funciona na prática com um caso de uso de recomendação de viagens do [ask-user-question-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/ask-user-question-demo):

```
USUÁRIO: Qual país da UE visitar em seguida?

Interesses: Quais são seus principais interesses ao viajar?
  1. História & Cultura - Museus, sítios históricos, arquitetura
  2. Natureza & Ar Livre - Trilhas, praias, montanhas, parques nacionais
  3. Gastronomia & Bebidas - Experiências culinárias, regiões vinícolas, tours gastronômicos
  4. Cidades & Urbano - Exploração urbana, compras, vida noturna
  (Digite números separados por vírgulas, ou texto personalizado)
1,2

Temporada: Quando você planeja viajar?
  1. Primavera - Março a Maio - Temperaturas amenas, menos turistas
  2. Verão - Junho a Agosto - Tempo quente, alta temporada
  3. Outono - Setembro a Novembro - Temperaturas mais frescas, cores do outono
  4. Inverno - Dezembro a Fevereiro - Mercados de Natal, esportes de inverno
  (Digite um número, ou texto personalizado)
3

Orçamento: Qual é o seu nível de orçamento aproximado para essa viagem?
  1. ...
...

ASSISTENTE: Com base nas suas preferências para uma viagem no outono com interesses
em história, cultura e natureza com orçamento moderado, buscando uma mistura de
pontos turísticos populares e experiências fora do circuito tradicional, aqui estão
excelentes países da UE para considerar:

1. **Portugal** - Bairros históricos de Lisboa, colheitas do Vale do Douro,
   costas do Algarve. Acessível com temperaturas agradáveis no outono.
2. **Grécia** - Sítios antigos, ilhas lindas, ainda quente para nadar
   no outono com menos multidões.
3. **Eslovênia** - Lago Bled, Alpes Julianos, cores espetaculares do outono.
   Mais acessível do que a Áustria ou a Itália vizinhas.
```

O agente coletou requisitos abrangentes por meio de perguntas interativas antes de fornecer recomendações personalizadas — sem suposições, sem necessidade de iteração.

---

## Primeiros Passos

### 1. Adicione a Dependência

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.3.0</version>
</dependency>
```

### 2. Configure Seu Agente

```java
ChatClient chatClient = chatClientBuilder
    .defaultTools(AskUserQuestionTool.builder()
        .questionHandler(this::handleQuestions)
        .build())
    .build();
```

### 3. Implemente Seu `QuestionHandler`

Use os exemplos de console ou web abaixo.

O agente invoca automaticamente a ferramenta quando precisa de esclarecimentos e usa as respostas para fornecer soluções personalizadas.

💡 **Demo:** [ask-user-question-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/ask-user-question-demo)

---

## Exemplos de QuestionHandler

### QuestionHandler Baseado em Console

Uma implementação baseada em console:

```java
private static Map<String, String> handleQuestions(List<Question> questions) {
    Map<String, String> answers = new HashMap<>();
    Scanner scanner = new Scanner(System.in);

    for (Question q : questions) {
        System.out.println("\n" + q.header() + ": " + q.question());

        for (int i = 0; i < q.options().size(); i++) {
            Option opt = q.options().get(i);
            System.out.printf("  %d. %s - %s%n", i + 1, opt.label(), opt.description());
        }

        System.out.println(q.multiSelect()
            ? "  (Digite números separados por vírgulas, ou texto personalizado)"
            : "  (Digite um número, ou texto personalizado)");

        String response = scanner.nextLine().trim();

        // Analisa seleção(ões) numéricas ou usa como texto livre
        try {
            String[] parts = response.split(",");
            List<String> labels = new ArrayList<>();
            for (String part : parts) {
                int index = Integer.parseInt(part.trim()) - 1;
                if (index >= 0 && index < q.options().size()) {
                    labels.add(q.options().get(index).label());
                }
            }
            answers.put(q.question(), labels.isEmpty() ? response : String.join(", ", labels));
        } catch (NumberFormatException e) {
            answers.put(q.question(), response);
        }
    }
    return answers;
}
```

O manipulador exibe as opções, aceita seleções numéricas (como "1,2") ou texto livre (como "Um orçamento moderado") e retorna as respostas ao agente.

### QuestionHandler Baseado em Web

Para aplicações web, use `CompletableFuture` para conectar interações assíncronas de UI com a API síncrona do `QuestionHandler`. Envie perguntas ao seu frontend via WebSocket/SSE e bloqueie em `future.get()`. Complete o future quando o usuário enviar respostas via um endpoint REST.

---

## Conclusão

O **AskUserQuestionTool** transforma agentes de IA de respondentes baseados em suposições em parceiros colaborativos que coletam requisitos antes de agir, entregando respostas alinhadas com suas necessidades logo na primeira tentativa.

---

## Próximos na Série

- [**TodoWriteTool**](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/) — Acompanhe fluxos de trabalho de múltiplas etapas de forma transparente
- [**Orquestração de Subagentes**](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) — Arquiteturas multi-agente hierárquicas
- [**Integração A2A**](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration) — Construindo agentes interoperáveis com o protocolo Agent2Agent
- **Subagent Extension Framework** (em breve) — Orquestração de agentes agnóstica de protocolo

---

## Recursos

- **Repositório GitHub**: [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils)
- **Documentação do AskUserQuestionTool**: [AskUserQuestionTool](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/AskUserQuestionTool.md)
- **Documentação do Spring AI**: [docs.spring.io/spring-ai](https://docs.spring.io/spring-ai/reference/)
- **Demo**: [ask-user-question-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/ask-user-question-demo) — Questionamento interativo baseado em console (este post)
- **Claude Agent SDK**: [Documentação de Entrada do Usuário](https://platform.claude.com/docs/en/agent-sdk/user-input#question-format)

---

## Links da Série

- **Parte 1**: [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) — Capacidades modulares e reutilizáveis
- **Parte 2**: AskUserQuestionTool (este post) — Fluxos interativos
- **Parte 3**: [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/) — Planejamento estruturado
- **Parte 4**: [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) — Arquiteturas de agentes hierárquicas
- **Parte 5**: [Integração A2A](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration) — Construindo agentes interoperáveis com o protocolo Agent2Agent
- **Em breve**: Subagent Extension Framework — Orquestração de agentes agnóstica de protocolo

---

## Blogs Relacionados do Spring AI

- [Dynamic Tool Discovery](https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov) — Alcance 34-64% de economia de tokens
- [Tool Argument Augmentation](https://spring.io/blog/2025/12/23/spring-ai-tool-argument-augmenter-tzolov) — Capture o raciocínio do LLM durante a execução de ferramentas