# Padrões Agentivos com Spring AI (Parte 3): Por Que Seu Agente de IA Esquece Tarefas (E Como Resolver)

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 20 de janeiro de 2026

![TodoWriteTool](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260120/agent-todo-write-logo-2.png)

Você já pediu a um agente de IA para executar uma tarefa complexa de múltiplos passos e descobriu que ele pulou uma etapa crítica no meio do caminho? Você não está sozinho.

Pesquisas mostram que os LLMs sofrem com falhas do tipo "perdido no meio" — esquecendo tarefas enterradas em contextos longos. Quando seu agente lida simultaneamente com edições de arquivos, execução de testes e atualizações de documentação, etapas importantes podem desaparecer silenciosamente. Uma solução, inspirada no Claude Code, é tornar o planejamento explícito e observável com a ajuda de uma ferramenta dedicada `TodoWrite`. O resultado: agentes que nunca pulam etapas e fluxos de trabalho que você pode acompanhar em tempo real.

**Esta é a Parte 3 da nossa série Padrões Agentivos com Spring AI.** Já cobrimos [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) para capacidades modulares e [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) para fluxos interativos. Agora exploramos como o `TodoWriteTool` traz gerenciamento estruturado de tarefas para agentes Spring AI.

**Pronto para começar?** Pule para a seção de [Primeiros Passos](#primeiros-passos).

---

## O que é o TodoWriteTool?

O `TodoWriteTool` é uma ferramenta Spring AI que permite aos LLMs criar, acompanhar e atualizar listas de tarefas durante a execução. Inspirado no [TodoWrite do Claude Code](https://platform.claude.com/docs/en/agent-sdk/todo-tracking), ele transforma o planejamento implícito em fluxos de trabalho explícitos e rastreáveis. A implementação completa está disponível no GitHub: [TodoWriteTool.java](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/src/main/java/org/springaicommunity/agent/tools/TodoWriteTool.java)

Quando um agente recebe uma tarefa complexa, como "Adicione um botão de modo escuro na página de configurações e execute os testes", ele usa o `TodoWriteTool` para decompô-la antes da execução:

![Fluxo do TodoWrite](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260120/todo-write-flow.png)

O LLM chama a ferramenta sempre que precisa atualizar o plano — seja criando tarefas iniciais, marcando progresso ou adicionando trabalho recém-descoberto.

A ferramenta aceita uma lista de itens de tarefa, cada um com **id**, **content** (o que precisa ser feito) e **status**. Cada item segue um ciclo de vida simples:

![Estados do TodoWrite](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260120/todo-write-states.png)

A ferramenta impõe uma restrição importante: **apenas uma tarefa pode estar `in_progress` por vez**. Isso força uma execução sequencial e focada, em vez de tentativas dispersas de trabalho paralelo.

Veja como o progresso em tempo real se parece durante a execução:

```
Progresso: 2/4 tarefas concluídas (50%)
[✓] Encontrar os 10 melhores filmes de Tom Hanks
[✓] Agrupar filmes em pares
[→] Imprimir títulos invertidos
[ ] Resumo final
```

### Como o LLM Sabe Quando Usar a Ferramenta

A [descrição da ferramenta](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/8df9b26bdeb98cccfe65c445dea7611605d80e4c/spring-ai-agent-utils/src/main/java/org/springaicommunity/agent/tools/TodoWriteTool.java#L54-L242) instrui o LLM sobre quando o rastreamento de tarefas é apropriado:

> *"Use esta ferramenta quando uma tarefa exigir 3 ou mais etapas ou ações distintas. Ignore quando houver apenas uma tarefa única e simples que possa ser concluída em menos de 3 etapas triviais."*

Esse comportamento auto-regulado significa que o agente decide autonomamente se deve criar uma lista de tarefas com base na complexidade.

💡 **Dica:** Adicionalmente, para melhores resultados, use um prompt de sistema com instruções detalhadas de gerenciamento de tarefas. O [MAIN\_AGENT\_SYSTEM\_PROMPT\_V2](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/src/main/resources/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md#task-management) fornece um exemplo inspirado no Claude Code.

⚠️ **Importante:** O padrão Todo-Write depende do [Chat Memory](https://docs.spring.io/spring-ai/reference/api/chat-memory.html#page-title) para reter as atualizações da lista de tarefas e transmiti-las ao LLM. Além disso, habilitar o [ToolCallAdvisor](https://docs.spring.io/spring-ai/reference/api/advisors-recursive.html#_toolcalladvisor) substitui a chamada de ferramentas integrada do ChatModel e garante que todas as mensagens de ferramentas sejam registradas na memória de chat. Veja a configuração completa de advisors em [Primeiros Passos](#primeiros-passos) abaixo.

---

## Primeiros Passos

### 1. Adicione a Dependência

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.4.0</version>
</dependency>
```

> ℹ️ **Nota:** Requer Spring AI versão `2.0.0-SNAPSHOT` ou `2.0.0-M2` quando lançado.

### 2. Configure Seu Agente

```java
ChatClient chatClient = chatClientBuilder
    .defaultTools(TodoWriteTool.builder().build())
    .defaultAdvisors(
        ToolCallAdvisor.builder().conversationHistoryEnabled(false).build(),
        MessageChatMemoryAdvisor.builder(MessageWindowChatMemory.builder().build()).build())
    .build();

String response = chatClient.prompt()
    .user("Encontre os 10 melhores filmes de Tom Hanks, agrupe-os em pares, " +
          "e imprima cada título ao contrário. Use o TodoWrite para organizar suas tarefas.")
    .call()
    .content();
```

> ⚠️ **Importante:** Definir `conversationHistoryEnabled(false)` desativa o histórico de chamadas de ferramentas integrado em favor do `MessageChatMemoryAdvisor`.

Para um exemplo completo com prompts de sistema e ferramentas adicionais, veja o projeto [todo-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/todo-demo).

### 3. (Opcional) Atualizações de Progresso Orientadas a Eventos

A ferramenta publica eventos que sua aplicação pode usar para atualizar interfaces em tempo real. Por exemplo, defina um `ApplicationEvent` dedicado e um listener de eventos:

```java
public class TodoUpdateEvent extends ApplicationEvent {
    private final List<TodoItem> todos;
    public TodoUpdateEvent(Object source, List<TodoItem> todos) {
        super(source);
        this.todos = todos;
    }
    public List<TodoItem> getTodos() { return todos; }
}

@Component
public class TodoProgressListener {

    @EventListener
    public void onTodoUpdate(TodoUpdateEvent event) {
        int completed = (int) event.getTodos().stream()
            .filter(t -> t.status() == Todos.Status.completed).count();
        int total = event.getTodos().size();

        System.out.printf("\nProgresso: %d/%d tarefas concluídas (%.0f%%)\n",
            completed, total, (completed * 100.0 / total));
    }
}
```

Em seguida, adicione o publicador de eventos no seu `todoEventHandler`:

```java
@Autowired
ApplicationEventPublisher applicationEventPublisher;

ChatClient chatClient = chatClientBuilder
    .defaultTools(TodoWriteTool.builder()
        // Publica eventos de atualização de tarefas
        .todoEventHandler(event ->
            applicationEventPublisher.publishEvent(new TodoUpdateEvent(this, event.todos())))
        .build())
    // ...
    .build();
```

---

## Conclusão

O `TodoWriteTool` traz gerenciamento estruturado de tarefas para agentes Spring AI, transformando o planejamento implícito em fluxos de trabalho explícitos e observáveis. Ao tornar o plano do agente visível e rastreável, você obtém execução mais confiável, melhor experiência do usuário e depuração mais fácil.

**Conclusão principal:** Se o seu agente está pulando etapas em tarefas complexas, adicione o `TodoWriteTool`. A sobrecarga é mínima; o LLM decide quando o rastreamento é necessário com base na complexidade da tarefa.

Combinado com [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) para conhecimento de domínio e [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) para esclarecimento interativo, o `TodoWriteTool` completa a base para construção de agentes de IA confiáveis.

**Próximo:** Na [Parte 4](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents), exploramos a **Orquestração de Subagentes** com `TaskTool`, e na Parte 5, o **Subagent Extension Framework** para orquestração de agentes agnóstica de protocolo.

---

## Recursos

- **Repositório GitHub**: [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils)
- **Documentação do TodoWriteTool**: [TodoWriteTool.md](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/TodoWriteTool.md)
- **Projetos de Exemplo**:
  - [todo-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/todo-demo) — Demonstração focada no TodoWriteTool
  - [code-agent-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/code-agent-demo) — Integração completa do toolkit

### Relacionados

- [Claude Code Todo Tracking](https://platform.claude.com/docs/en/agent-sdk/todo-tracking) — Inspiração original
- [Dynamic Tool Discovery](https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov) — Seleção eficiente de ferramentas
- [Tool Argument Augmentation](https://spring.io/blog/2025/12/23/spring-ai-tool-argument-augmenter-tzolov) — Capturando o raciocínio do LLM

---

## Links da Série

- **Parte 1**: [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) — Capacidades modulares e reutilizáveis
- **Parte 2**: [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) — Fluxos interativos
- **Parte 3**: TodoWriteTool — Planejamento estruturado (este post)
- **Parte 4**: [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) — Arquiteturas de agentes hierárquicas
- **Parte 5**: Subagent Extension Framework (em breve) — Orquestração de agentes agnóstica de protocolo