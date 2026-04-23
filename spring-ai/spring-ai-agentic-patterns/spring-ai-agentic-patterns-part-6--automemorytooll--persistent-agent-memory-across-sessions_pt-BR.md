# Padrões Agentivos com Spring AI (Parte 6): AutoMemoryTools — Memória Persistente de Agentes Entre Sessões

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 07 de abril de 2026 | **Leitura:** 9 min

> *Memória de Longo Prazo Baseada em Arquivos para Agentes Spring AI*

Agentes são tão úteis quanto aquilo que conseguem lembrar. O [Chat Memory](https://docs.spring.io/spring-ai/reference/2.0-SNAPSHOT/api/chat-memory.html#page-title) do Spring AI armazena toda a conversa e pode persistir as informações entre reinicializações, mas guarda *tudo* — e quando a janela de contexto enche, as mensagens mais antigas são removidas. A futura Session API adicionará sumarização recursiva para suavizar isso, mas fatos precisos ainda se perdem quando detalhes são comprimidos.

`AutoMemoryTools` e `AutoMemoryToolsAdvisor`, parte do toolkit [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils), oferecem aos seus agentes uma **memória de longo prazo durável, baseada em arquivos**, que persiste entre sessões. O design é inspirado no [sistema de auto-memória do Claude Code](https://code.claude.com/docs/en/memory#auto-memory) e na [especificação da Memory Tool da API do Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) — portado para o Spring AI para funcionar com qualquer provedor de LLM.

---

## Memória de Longo Prazo vs. Histórico de Conversa

`ChatMemory` e `AutoMemoryTools` são complementares — um agente bem configurado usa os dois. O `ChatMemory` mantém a janela completa da conversa: cada turno, automaticamente, limitado por uma janela deslizante. O `AutoMemoryTools` é a camada curada: o modelo escreve apenas o que *vale a pena guardar para sempre* — uma preferência do usuário, uma decisão de projeto, uma correção de comportamento — em um arquivo Markdown tipado que sobrevive indefinidamente.

Use `ChatMemory` para a tarefa atual; use `AutoMemoryTools` para fatos que ainda devem estar disponíveis na semana que vem.

---

> **Esta é a Parte 6 da nossa série Padrões Agentivos com Spring AI.** Já cobrimos [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills), [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool), [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/), [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) e [Integração A2A](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration). Agora adicionamos memória que sobrevive à sessão.

🚀 **Quer ir direto ao ponto?** Pule para a seção de [Início Rápido](#início-rápido).

---

## Como Funciona

O agente gerencia sua própria memória por meio de seis ferramentas especializadas no `AutoMemoryTools`, todas restritas a um diretório de memórias isolado. O diagrama abaixo mostra o fluxo completo de requisição:

![Fluxo de execução do AutoMemoryTools](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260407/spring-ai-auto-memory-flow.png)

**① Requisição do usuário** — a requisição, combinada com o prompt de sistema de memória, flui pela pilha de advisors do Spring AI (`ToolCallAdvisor` + `ChatMemoryAdvisor`) até o LLM. O LLM decide se deve carregar, criar ou atualizar memórias antes de responder.

**② Chamada de ferramenta** — o LLM invoca o `AutoMemoryTools`. As ferramentas leem e escrevem arquivos Markdown tipados no diretório de memória configurado — `MEMORY.md` como índice, mais arquivos individuais por tópico como `user_profile.md` ou `project_history.md`.

**③ Recuperação/atualização subsequente** — o LLM pode emitir chamadas adicionais para carregar arquivos específicos ou atualizar os existentes; por exemplo, carregar um arquivo após encontrar seu ponteiro no `MEMORY.md`, ou mesclar entradas durante a consolidação.

**④ Resposta final** — uma vez concluídas todas as operações de memória, o LLM produz sua resposta, que flui de volta pela pilha de advisors ao usuário.

### Prompt de Sistema de Memória

O prompt de sistema de memória conduz esse comportamento. Dois variantes são incluídos no jar:

- `AUTO_MEMORY_TOOLS_SYSTEM_PROMPT.md` — usado com as [Opções A](#opção-a-automemorytoolsadvisor-sem-boilerplate) e [B](#opção-b-configuração-manual-automemorytools-diretamente); `AutoMemoryTools` dedicado e isolado
- `AUTO_MEMORY_FILESYSTEM_TOOLS_SYSTEM_PROMPT.md` — usado com a [Opção C](#opção-c-filesystemtools--shelltools); operações genéricas `Read`/`Write`/`Edit` via `FileSystemTools`

Ambos os prompts codificam o mesmo modelo de memória, diferindo apenas nas operações que o modelo é instruído a chamar. Eles instruem o modelo a ler `MEMORY.md` no início de cada sessão, salvar via fluxo de dois passos (`MemoryCreate` → `MemoryInsert`), aplicar os quatro tipos de memória, ignorar conteúdo efêmero, verificar fatos recuperados antes de agir sobre eles, e manter o índice sincronizado ao deletar ou renomear arquivos.

### MEMORY.md — O Arquivo Índice

`MEMORY.md` é o índice sempre carregado. É uma lista simples de ponteiros de uma linha para todos os arquivos de memória:

```
- [Perfil do Usuário](user_profile.md) — Alice, engenheira backend, prefere respostas curtas
- [Testes de Feedback](feedback_testing.md) — sempre usar banco de dados real em testes de integração
- [Reescrita Auth do Projeto](project_auth.md) — motivado por conformidade legal, não dívida técnica
```

O modelo lê esse índice no início de cada sessão e, em seguida, carrega seletivamente os arquivos que parecem relevantes — mantendo a janela de contexto enxuta mesmo com o crescimento da memória.

### Formato dos Arquivos de Memória

Cada memória é um arquivo Markdown com frontmatter YAML:

```
---
name: perfil do usuário
description: Alice — engenheira backend, prefere respostas curtas
type: user
---

Engenheira backend chamada Alice.
Prefere respostas concisas e diretas sem resumos finais.
```

### Tipos de Memória

Nem tudo vale a pena guardar. O modelo de memória define quatro tipos, cada um com orientações claras sobre o que salvar e quando — para que o agente acumule informação útil, não ruído:

- **`user`** — cargo, objetivos, experiência, estilo de comunicação
- **`feedback`** — correções e abordagens confirmadas ("pare de resumir", "sim, estava certo")
- **`project`** — decisões e prazos que não estão no código ou no git (alvos de migração, datas de congelamento)
- **`reference`** — ponteiros para sistemas externos (boards do Linear, dashboards do Grafana, canais do Slack)

### Operações de Memória

O `AutoMemoryTools` expõe seis operações com nomes específicos e isoladas. A Opção C alcança o mesmo resultado por meio das operações genéricas de `FileSystemTools` e `ShellTools`.

#### AutoMemoryTools

As Opções A e B usam o `AutoMemoryTools`, que implementa a [especificação da Memory Tool da API do Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) de forma direta, com operações nomeadas por propósito e isoladas:

| Ferramenta | Propósito |
|---|---|
| `MemoryView` | Lê um arquivo com números de linha, ou lista um diretório dois níveis abaixo |
| `MemoryCreate` | Cria um novo arquivo de memória (Passo 1 do salvamento em dois passos) |
| `MemoryStrReplace` | Substitui uma string exata e única em um arquivo existente |
| `MemoryInsert` | Insere texto após um número de linha específico — uso principal: anexar ao `MEMORY.md` |
| `MemoryDelete` | Deleta um arquivo ou diretório recursivamente |
| `MemoryRename` | Renomeia ou move um arquivo; atualiza o link do `MEMORY.md` separadamente |

#### FileSystemTools & ShellTools

A Opção C usa as ferramentas genéricas `FileSystemTools` e `ShellTools`, mapeando para as mesmas operações:

| Operação | Equivalente a |
|---|---|
| `Read` | `MemoryView` |
| `Write` | `MemoryCreate` |
| `Edit` | `MemoryStrReplace` / `MemoryInsert` |
| `Bash` (ex: `rm`, `mv`) | `MemoryDelete` / `MemoryRename` |

As Opções A e B isolam todos os caminhos; a Opção C não tem isolamento — o agente tem acesso total ao sistema de arquivos.

---

## Abordagens de Integração

Existem três formas de adicionar memória de longo prazo a um agente Spring AI, variando de zero boilerplate até totalmente manual. Escolha com base em quanto controle você precisa sobre o prompt de sistema, as preocupações de segurança e se seu agente já usa ferramentas de sistema de arquivos de uso geral.

### Opção A: AutoMemoryToolsAdvisor (sem boilerplate)

Adicione um único advisor ao builder do seu `ChatClient`:

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        // Memória de longo prazo — fatos que sobrevivem entre sessões
        AutoMemoryToolsAdvisor.builder()
            .memoriesRootDirectory("/home/user/.agent/memories")
            .build(),

        // Histórico de conversa — janela completa de mensagens para esta sessão
        MessageChatMemoryAdvisor.builder(
            MessageWindowChatMemory.builder().maxMessages(100).build())
            .build(),

        // Chamada de ferramentas
        ToolCallAdvisor.builder().disableInternalConversationHistory().build())
    .build();
```

Em cada requisição, o advisor injeta automaticamente `AUTO_MEMORY_TOOLS_SYSTEM_PROMPT.md` na mensagem do sistema, registra todas as seis `AutoMemoryTools` (desduplicando registros existentes) e, opcionalmente, adiciona um lembrete de consolidação se o `memoryConsolidationTrigger` disparar.

![AutoMemoryTools Advisor](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260407/spring-ai-auto-memory-tools-advisor.png)

O `AutoMemoryToolsAdvisor` é executado primeiro na cadeia, enriquecendo o contexto da requisição com o prompt de sistema de memória e as seis definições de ferramentas antes de passar para o `ToolCallAdvisor` e `ChatMemoryAdvisor`. Quando o contexto enriquecido chega ao LLM, ele já contém o histórico da conversa, as definições de ferramentas e as instruções de memória — tudo que o modelo precisa para decidir o que carregar, salvar ou atualizar.

### Opção B: Configuração Manual (AutoMemoryTools diretamente)

Configure o `AutoMemoryTools` no seu `ChatClient` junto com o prompt de sistema complementar:

```java
@Value("classpath:/prompt/AUTO_MEMORY_TOOLS_SYSTEM_PROMPT.md")
Resource memorySystemPrompt;

@Value("${agent.memory.dir}")
String memoryDir;

ChatClient chatClient = chatClientBuilder
    .defaultSystem(p -> p
        .text(memorySystemPrompt)
        .param("MEMORIES_ROOT_DIERCTORY", memoryDir))
    .defaultTools(
        AutoMemoryTools.builder().memoriesDir(memoryDir).build(),
        TodoWriteTool.builder().build())
    .defaultAdvisors(ToolCallAdvisor.builder().build())
    .build();
```

Use essa abordagem quando quiser controle total sobre a estrutura do prompt de sistema — por exemplo, combinando memória com um prompt de sistema principal personalizado.

### Opção C: FileSystemTools + ShellTools

Se o agente já tem `FileSystemTools` e `ShellTools` para outras tarefas, você pode implementar o mesmo padrão de memória sem adicionar `AutoMemoryTools`. O agente usa as mesmas operações `Read`, `Write` e `Edit` que usaria para qualquer trabalho com arquivos — a memória é apenas mais um diretório.

```java
@Value("classpath:/prompt/AUTO_MEMORY_FILESYSTEM_TOOLS_SYSTEM_PROMPT.md")
Resource memorySystemPrompt;

@Value("${agent.memory.dir}")
String memoryDir;

ChatClient chatClient = chatClientBuilder
    .defaultSystem(p -> p
        .text(memorySystemPrompt)
        .param("MEMORIES_ROOT_DIERCTORY", memoryDir))   // diz ao agente onde escrever
    .defaultTools(
        ShellTools.builder().build(),         // Bash — mkdir, ls, etc.
        FileSystemTools.builder().build())    // Read, Write, Edit — operações de arquivo de memória
    .defaultAdvisors(ToolCallAdvisor.builder().build())
    .build();
```

As mesmas convenções de memória se aplicam — arquivos tipados, índice `MEMORY.md`, salvamento em dois passos — mas **não há isolamento**: o agente tem acesso total ao sistema de arquivos e permanece no diretório configurado apenas por convenção.

|  | `AutoMemoryTools` (Opções A e B) | `FileSystemTools` (Opção C) |
|---|---|---|
| **Modelo de caminhos** | Caminhos relativos, isolados à raiz de memórias | Caminhos absolutos, acesso total ao sistema de arquivos |
| **Segurança** | Proteção contra traversal integrada | Sem isolamento — o agente segue o prompt por convenção |
| **Nomes das ferramentas** | Nomes por propósito (`MemoryCreate`, `MemoryView`, …) | Genéricos (`Write`, `Read`, `Edit`) |
| **Ideal para** | Agentes somente de memória, isolamento necessário | Agentes que já usam ferramentas de sistema de arquivos para outras tarefas |

Essa abordagem segue diretamente o [padrão de auto-memória do Claude Code](https://code.claude.com/docs/en/memory), onde as mesmas ferramentas de arquivo servem tanto para edição de código quanto para gerenciamento de memória.

> ⚠️ **Nota de segurança:** Com `FileSystemTools`, o agente pode ler e escrever em qualquer lugar do sistema de arquivos. Use essa abordagem apenas em ambientes controlados e confiáveis.

---

## Mantendo a Memória Limpa

Com o tempo, um armazenamento de memória acumula entradas redundantes, sobrepostas ou desatualizadas. Pedir periodicamente ao agente que consolide — mesclando duplicatas, removendo fatos desatualizados, refinando descrições — mantém o armazenamento enxuto e o índice `MEMORY.md` legível.

**Opções B e C (explícito):** basta perguntar:

```
USUÁRIO> Por favor, consolide sua memória — mescle duplicatas e remova qualquer coisa desatualizada.
```

**Opção A (gatilho automático):** o `AutoMemoryToolsAdvisor` aceita um predicado `memoryConsolidationTrigger`. Quando retorna `true`, um `<system-reminder>` é injetado na mensagem de sistema da próxima requisição, instruindo o modelo a consolidar sem que o usuário precise pedir. O demo usa um gatilho de dupla condição — tempo decorrido ou o usuário diz "tchau":

```java
Instant lastInteraction = Instant.now();

AutoMemoryToolsAdvisor.builder()
    .memoriesRootDirectory(memoryDir)
    .memoryConsolidationTrigger((request, instant) -> {
        var previous = lastInteraction;
        lastInteraction = Instant.now();

        // Consolidar quando passam mais de 60 segundos entre turnos
        if (instant.isAfter(previous.plusSeconds(60))) {
            return true;
        }
        // Também consolidar quando o usuário diz tchau
        var msg = request.prompt().getLastUserOrToolResponseMessage().getText();
        return msg != null && msg.toLowerCase().contains("bye");
    })
    .build()
```

Outras estratégias úteis:

```java
// Probabilístico: ~5% das requisições
.memoryConsolidationTrigger((req, now) -> Math.random() < 0.05)

// Contagem de turnos: a cada 50 chamadas
AtomicInteger counter = new AtomicInteger();
.memoryConsolidationTrigger((req, now) -> counter.incrementAndGet() % 50 == 0)
```

---

## Início Rápido

### 1. Adicione a Dependência

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.7.0</version>
</dependency>
```

> **Nota:** Para a versão estável mais recente, verifique a [página de releases do GitHub](https://github.com/spring-ai-community/spring-ai-agent-utils/releases). Requer Spring AI `2.0.0-M4+`.

### 2. Configure o Diretório de Memória

```properties
# application.properties
agent.memory.dir=${user.home}/.spring-ai-agent/my-app/memory
```

### 3. Configure a Integração

```java
@Value("${agent.memory.dir}")
String memoryDir;

ChatClient chatClient = chatClientBuilder
    .defaultAdvisors(
        // Memória de longo prazo — fatos que sobrevivem entre sessões
        AutoMemoryToolsAdvisor.builder()
            .memoriesRootDirectory(memoryDir)
            .build(),

        // Histórico de conversa — janela completa de mensagens para esta sessão
        MessageChatMemoryAdvisor.builder(MessageWindowChatMemory.builder().maxMessages(100).build())
            .build(),

        // Chamada de ferramentas
        ToolCallAdvisor.builder().disableInternalConversationHistory().build())
    .build();
```

### 4. Veja em Ação

Primeira sessão:

```
USUÁRIO> Meu nome é Alice. Sou engenheira backend e prefiro respostas curtas.
USUÁRIO> Lembre-se: estamos migrando do PostgreSQL para o CockroachDB neste trimestre.
```

Segunda sessão (novo processo JVM):

```
USUÁRIO> O que você sabe sobre mim?
ASSISTENTE> Você é Alice, engenheira backend. Você prefere respostas curtas.
            Você está migrando do PostgreSQL para o CockroachDB neste trimestre.
```

---

## Projetos de Exemplo

Três exemplos executáveis estão disponíveis no repositório:

**[memory-tools-advisor-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/memory/memory-tools-advisor-demo)** (Opção A) — `AutoMemoryToolsAdvisor` + `ToolCallAdvisor` + `MessageChatMemoryAdvisor` + advisor de log personalizado, com gatilho de consolidação de dupla condição.

**[memory-tools-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/memory/memory-tools-demo)** (Opção B) — `AutoMemoryTools` configurado manualmente junto com `TodoWriteTool`, demonstrando composição explícita do prompt de sistema.

**[memory-filesystem-tools-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/memory/memory-filesystem-tools-demo)** (Opção C) — mesmas convenções de memória implementadas com `FileSystemTools` + `ShellTools`, sem ferramentas de memória dedicadas.

Todos os três suportam Anthropic Claude, Google Gemini e OpenAI — descomente o provedor desejado no `pom.xml` e configure a chave de API.

---

## Conclusão

`AutoMemoryTools` é uma portagem para o Spring AI dos padrões de memória que a Anthropic pioneirizou no [Claude Code](https://code.claude.com/docs/en/memory) e na [especificação da Memory Tool da API do Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) — o índice `MEMORY.md`, arquivos tipados, salvamento em dois passos e API de seis operações isoladas — disponível para qualquer provedor de LLM via Spring AI.

A memória de longo prazo é a camada que faltava entre chamadas LLM sem estado e agentes verdadeiramente úteis. Adicione o `AutoMemoryToolsAdvisor` e seu agente começará a acumular conhecimento que sobrevive à sessão.

---

## Links da Série

- **Parte 1**: [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) — Capacidades modulares e reutilizáveis
- **Parte 2**: [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) — Fluxos de trabalho interativos
- **Parte 3**: [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/) — Planejamento estruturado
- **Parte 4**: [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) — Arquiteturas de agentes hierárquicas
- **Parte 5**: [Integração A2A](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration) — Construindo agentes interoperáveis
- **Parte 6**: AutoMemoryTools (este post) — Memória de longo prazo entre sessões

---

## Recursos

### Spring AI Agent Utils

- **GitHub**: [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils)
- **Docs do AutoMemoryTools**: [AutoMemoryTools](https://spring-ai-community.github.io/spring-ai-agent-utils/latest-snapshot/tools/AutoMemoryTools/)
- **Docs do AutoMemoryToolsAdvisor**: [AutoMemoryToolsAdvisor](https://spring-ai-community.github.io/spring-ai-agent-utils/latest-snapshot/tools/AutoMemoryToolsAdvisor/)
- **Documentação completa**: [spring-ai-community.github.io/spring-ai-agent-utils](https://spring-ai-community.github.io/spring-ai-agent-utils/latest-snapshot/)

### Projetos de Exemplo

- [memory-tools-advisor-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/memory/memory-tools-advisor-demo) — pilha completa de advisors (Opção A)
- [memory-tools-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/memory/memory-tools-demo) — configuração manual (Opção B)
- [memory-filesystem-tools-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/memory/memory-filesystem-tools-demo) — abordagem com ferramentas de sistema de arquivos (Opção C)