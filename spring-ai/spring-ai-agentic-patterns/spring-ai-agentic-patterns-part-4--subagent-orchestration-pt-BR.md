# Padrões Agentivos com Spring AI (Parte 4): Orquestração de Subagentes

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 27 de janeiro de 2026 | **Leitura:** 5 min

![Subagentes](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260127/subagents.png)

Em vez de um agente generalista fazendo tudo, delegue para agentes especializados. Isso mantém as janelas de contexto focadas — evitando a poluição que degrada o desempenho.

A [Task tool](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/TaskTools.md), parte do toolkit [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils), é uma implementação Spring AI **portável e agnóstica de modelo**, inspirada nos [subagentes do Claude Code](https://platform.claude.com/docs/en/agent-sdk/subagents). Ela permite arquiteturas de agentes hierárquicas onde subagentes especializados lidam com tarefas focadas em **janelas de contexto dedicadas**, retornando apenas os resultados essenciais ao agente pai. Além do formato baseado em Markdown do Claude, a arquitetura é extensível — suportando [A2A](https://google.github.io/A2A/) e outros protocolos agentivos para orquestração de agentes heterogêneos (mais informações serão fornecidas em um post de acompanhamento).

**Esta é a Parte 4 da nossa série Padrões Agentivos com Spring AI.** Já cobrimos [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills), [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) e [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/). Agora exploramos subagentes hierárquicos.

**Pronto para começar?** Pule para [Primeiros Passos](#primeiros-passos).

---

## Como Funciona

O agente principal delega tarefas a subagentes especializados por meio da Task tool, com cada subagente operando em sua própria janela de contexto isolada. A arquitetura de subagentes é composta por três componentes principais:

**1. Agente Principal (Orquestrador)**
O agente primário que interage com os usuários. Seu LLM tem acesso à ferramenta `Task` e conhece os subagentes disponíveis por meio do **Registro de Agentes** — um catálogo de nomes e descrições de subagentes populado na inicialização. O agente principal decide automaticamente quando delegar com base no campo `description` de cada subagente.

**2. Arquivos de Configuração de Agentes**
Os subagentes são definidos como arquivos Markdown (ex: `agent-x.md`, `agent-y.md`) em uma pasta `agents/`. Cada arquivo especifica o nome do subagente, descrição, ferramentas permitidas, modelo preferido e prompt de sistema. Essas configurações populam tanto o Registro de Agentes quanto a Task tool na inicialização.

**3. Subagentes**
Instâncias de agentes separadas que executam em janelas de contexto isoladas. Cada subagente pode usar um **LLM diferente** (LLM-X, LLM-Y, LLM-Z) com seu próprio prompt de sistema, ferramentas e skills — permitindo roteamento multi-modelo baseado na complexidade da tarefa.

O diagrama abaixo ilustra o fluxo de execução:

![Arquitetura de subagentes](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260127/sub-agents-architecture.png)

1. **Carregamento:** Na inicialização, a Task tool carrega as referências de subagentes configuradas, resolve seus nomes e descrições, e popula o registro de agentes.
2. **Usuário** envia uma pergunta complexa ao agente principal.
3. **O LLM do agente principal** avalia a solicitação e verifica os subagentes disponíveis no registro.
4. **O LLM decide** delegar invocando a ferramenta `Task` com o nome do subagente e a descrição da tarefa.
5. **A Task tool** instancia o subagente apropriado com base na configuração do agente.
6. **O subagente** trabalha de forma autônoma em sua janela de contexto dedicada.
7. **Os resultados** fluem de volta ao agente principal (apenas as descobertas essenciais, não as etapas intermediárias).
8. **O agente principal** sintetiza e retorna a resposta final ao usuário.

Cada subagente opera com:

- **Janela de contexto dedicada** — Isolada da conversa principal, evitando poluição
- **Prompt de sistema customizado** — Expertise personalizada para domínios específicos
- **Acesso a ferramentas configurável** — Restrito apenas às capacidades necessárias
- **Roteamento multi-modelo** — Direcione tarefas simples a modelos mais baratos, análises complexas a modelos mais capazes
- **Execução paralela** — Inicie múltiplos subagentes de forma concorrente
- **Tarefas em segundo plano** — Operações de longa duração executam de forma assíncrona

### Subagentes Integrados

O Spring AI Agent Utils fornece quatro subagentes integrados, registrados automaticamente quando o `TaskTool` é configurado:

| Subagente | Propósito | Ferramentas |
|---|---|---|
| **Explore** | Exploração rápida e somente leitura de código-fonte — encontrar arquivos, pesquisar código, analisar conteúdos | Read, Grep, Glob |
| **General-Purpose** | Pesquisa e execução de múltiplos passos com acesso completo de leitura/escrita | Todas as ferramentas |
| **Plan** | Arquiteto de software para projetar estratégias de implementação e identificar trade-offs | Somente leitura + busca |
| **Bash** | Especialista em execução de comandos para operações git, builds e tarefas de terminal | Apenas Bash |

Consulte a [documentação de referência](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/TaskTools.md#built-in-sub-agents) para capacidades detalhadas. Múltiplos subagentes podem ser executados de forma concorrente — por exemplo, executando `style-checker`, `security-scanner` e `test-coverage` simultaneamente durante uma revisão de código.

---

## Primeiros Passos

### 1. Adicione a Dependência

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.4.2</version>
</dependency>
```

### 2. Configure Seu Agente

```java
import org.springaicommunity.agent.tools.task.TaskToolCallbackProvider;

@Configuration
public class AgentConfig {

    @Bean
    CommandLineRunner demo(ChatClient.Builder chatClientBuilder) {
        return args -> {
            // Configurar Task tools
            var taskTools = TaskToolCallbackProvider.builder()
                .chatClientBuilder("default", chatClientBuilder)
                .subagentReferences(
                    ClaudeSubagentReferences.fromRootDirectory("src/main/resources/agents"))
                .build();

            // Construir o chat client principal com as Task tools
            ChatClient chatClient = chatClientBuilder
                .defaultToolCallbacks(taskTools)
                .build();

            // Uso natural — o agente delegará para subagentes automaticamente
            String response = chatClient
                .prompt("Explore o módulo de autenticação e explique como ele funciona")
                .call()
                .content();
        };
    }
}
```

O agente principal reconhece automaticamente quando delegar para subagentes com base nos campos `description` de cada um.

### 3. Roteamento Multi-Modelo (Opcional)

Direcione subagentes para diferentes modelos com base na complexidade da tarefa:

```java
var taskTools = TaskToolCallbackProvider.builder()
    .chatClientBuilder("default", sonnetBuilder)   // Modelo padrão
    .chatClientBuilder("haiku", haikuBuilder)      // Rápido e econômico
    .chatClientBuilder("opus", opusBuilder)        // Análises complexas
    .build();
```

Os subagentes especificam seu modelo preferido em sua definição, e a Task tool faz o roteamento correspondente.

---

## Criando Subagentes Personalizados

Subagentes personalizados são arquivos Markdown com frontmatter YAML, normalmente armazenados em `.claude/agents/`:

```
projeto-raiz/
├── .claude/
│   └── agents/
│       ├── revisor-de-codigo.md
│       └── executor-de-testes.md
```

### Formato do Arquivo de Subagente

```markdown
---
name: revisor-de-codigo
description: Revisor de código especialista. Use proativamente após escrever código.
tools: Read, Grep, Glob
disallowedTools: Edit, Write
model: sonnet
---

Você é um revisor de código sênior com expertise em qualidade de software.

**Quando Invocado:**
1. Execute `git diff` para ver as mudanças recentes
2. Foque a análise nos arquivos modificados
3. Verifique o contexto do código ao redor

**Checklist de Revisão:**
- Clareza e legibilidade do código
- Convenções de nomenclatura adequadas
- Tratamento de erros
- Vulnerabilidades de segurança

**Saída:** Feedback claro e acionável com referências de arquivo.
```

### Campos de Configuração

| Campo | Obrigatório | Descrição |
|---|---|---|
| `name` | Sim | Identificador único (letras minúsculas com hífens) |
| `description` | Sim | Descrição em linguagem natural de quando usar este subagente |
| `tools` | Não | Nomes das ferramentas permitidas (herda todas se omitido) |
| `disallowedTools` | Não | Ferramentas a negar explicitamente |
| `model` | Não | Preferência de modelo: `haiku`, `sonnet`, `opus` |

Consulte a [documentação de referência](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/TaskTools.md#creating-custom-sub-agents) para campos adicionais como `skills` e `permissionMode`.

> **Importante:** Subagentes não podem instanciar seus próprios subagentes. Não inclua `Task` na lista de `tools` de um subagente.

### Carregando Subagentes Personalizados

```java
import org.springaicommunity.agent.tools.task.subagent.claude.ClaudeSubagentReferences;

var taskTools = TaskToolCallbackProvider.builder()
    .chatClientBuilder("default", chatClientBuilder)
    .subagentReferences(
        ClaudeSubagentReferences.fromRootDirectory("src/main/resources/agents")
    )
    .build();
```

---

## Execução em Segundo Plano

Subagentes de longa duração podem executar de forma assíncrona. O agente principal continua trabalhando enquanto os subagentes em segundo plano são executados. Use o `TaskOutputTool` para recuperar os resultados quando necessário. Para armazenamento persistente de tarefas entre instâncias, consulte a [documentação do TaskRepository](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/TaskTools.md#background-task-management).

---

## Conclusão

A Task tool traz arquiteturas de subagentes hierárquicas para o Spring AI, possibilitando isolamento de contexto, instruções especializadas e roteamento multi-modelo eficiente. Ao delegar tarefas complexas para subagentes focados, seu agente principal permanece enxuto e responsivo.

**Próximo:** Na [Parte 5](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration), exploramos a **Integração A2A** — construindo agentes interoperáveis com o protocolo Agent2Agent. Em um post de acompanhamento, abordaremos o **Subagent Extension Framework** — uma abstração agnóstica de protocolo para integrar agentes remotos via A2A, MCP ou protocolos personalizados.

---

## Recursos

- **Repositório GitHub**: [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils)
- **Documentação do TaskTools**: [TaskTools.md](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/TaskTools.md)
- **Projeto de Exemplo**: [subagent-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/subagent-demo)

### Relacionados

- [Claude Code Subagents](https://platform.claude.com/docs/en/agent-sdk/subagents) — Inspiração original

---

## Links da Série

- **Parte 1**: [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) — Capacidades modulares e reutilizáveis
- **Parte 2**: [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) — Fluxos interativos
- **Parte 3**: [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/) — Planejamento estruturado
- **Parte 4**: Orquestração de Subagentes (este post) — Arquiteturas de agentes hierárquicas
- **Parte 5**: [Integração A2A](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration) — Construindo agentes interoperáveis com o protocolo Agent2Agent
- **Em breve**: Subagent Extension Framework — Orquestração de agentes agnóstica de protocolo

### Blogs Relacionados do Spring AI

- [Dynamic Tool Discovery](https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov) — Alcance 34-64% de economia de tokens
- [Tool Argument Augmentation](https://spring.io/blog/2025/12/23/spring-ai-tool-argument-augmenter-tzolov) — Capture o raciocínio do LLM durante a execução de ferramentas