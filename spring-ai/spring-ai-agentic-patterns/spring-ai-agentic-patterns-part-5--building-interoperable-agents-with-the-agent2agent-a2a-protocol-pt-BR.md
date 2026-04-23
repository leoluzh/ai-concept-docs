# Padrões Agentivos com Spring AI (Parte 5): Construindo Agentes Interoperáveis com o Protocolo Agent2Agent (A2A)

**Categoria:** Engenharia | **Autor:** Ilayaperumal Gopinathan | **Data:** 29 de janeiro de 2026 | **Leitura:** 7 min

![Spring AI A2A](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/igopinathan/20260129/spring-ai-a2a-logo.png)

O **Protocolo Agent2Agent (A2A)** é um padrão aberto para comunicação perfeita entre agentes de IA. Ele permite que agentes descubram capacidades, troquem mensagens e coordenem fluxos de trabalho entre plataformas — independentemente de sua implementação.

O **[Spring AI A2A](https://github.com/spring-ai-community/spring-ai-a2a)** integra o A2A Java SDK com o Spring AI por meio de autoconfiguração do Spring Boot. Ele conecta perfeitamente o protocolo A2A ao `ChatClient` e às ferramentas do Spring AI, permitindo que você exponha seus agentes como servidores A2A.

Este post faz parte da série **Padrões Agentivos com Spring AI**. Enquanto os posts anteriores abordaram como tornar agentes individuais mais capazes ([Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills/), [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool/), [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/), [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents/)), este post mostra como o Protocolo A2A permite que agentes colaborem além dos limites do sistema por meio de exemplos práticos.

---

## Protocolo Agent2Agent (A2A)

O [Protocolo A2A](https://a2a-protocol.org/) é um padrão aberto para comunicação entre agentes de IA, fornecendo uma base neutra em relação a fornecedores que permite que agentes descubram capacidades, troquem mensagens e coordenem fluxos de trabalho. Construído sobre os padrões HTTP, SSE e JSON-RPC.

A **Descoberta de Agentes** é a base da comunicação A2A. Os agentes descobrem as capacidades uns dos outros por meio do **AgentCard** — um documento JSON padronizado exposto em `/.well-known/agent-card.json` que descreve a identidade, as capacidades e as skills de um agente. Isso segue um padrão de três etapas: **Descoberta** → **Iniciação** → **Conclusão**.

O protocolo define dois papéis: agentes **Servidor A2A** expõem endpoints para descoberta e tratamento de mensagens, enquanto agentes **Cliente A2A** iniciam a comunicação descobrindo agentes remotos e enviando mensagens.

O **[A2A Java SDK](https://github.com/a2aproject/a2a-java)** fornece uma implementação Java com componentes do lado do servidor para processar requisições e gerenciar tarefas, e componentes do lado do cliente para chamar agentes remotos. Ele suporta múltiplos transportes (HTTP, SSE, JSON-RPC) e lida com os detalhes de baixo nível do protocolo.

---

## Integração Spring AI A2A

Embora o A2A Java SDK forneça a implementação do protocolo, integrá-lo ao Spring AI requer configuração adicional. É aqui que o projeto **Spring AI A2A** entra.

O projeto `spring-ai-a2a` atualmente se concentra na **integração do lado do servidor**, permitindo que você exponha seus agentes Spring AI como servidores compatíveis com A2A.

![Integração Spring AI A2A](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/igopinathan/20260129/spring-ai-a2a-integration.png)

Isso abrange:

- **Autoconfiguração do Spring Boot**: Endpoints A2A autoconfigurados
- **Integração com Spring AI**: Integração direta com o `ChatClient` e as `tools` do Spring AI
- **Transporte JSON-RPC via REST Controllers**: Atualmente implementa transporte JSON-RPC para comunicação entre agentes. Os controllers fornecem endpoints para agent cards e tratamento de mensagens.
- **Implementação do AgentExecutor**: `DefaultAgentExecutor` que conecta o A2A SDK ao Spring AI

Veja o que a integração gerencia para você:

```
// O framework expõe automaticamente estes endpoints A2A (relativos ao seu context path):
POST   /                                  // Tratar requisições JSON-RPC sendMessage
GET    /.well-known/agent-card.json      // Agent card (localização padrão A2A)
GET    /card                              // Agent card (endpoint alternativo)
```

---

## Como Funciona

![Fluxo Spring AI A2A](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/igopinathan/20260129/spring-ai-a2a-flow.png)

**Passo a passo:**

1. **Descoberta de Agente**: Antes de a comunicação começar, o cliente busca o `AgentCard` do servidor em `/.well-known/agent-card.json` para descobrir capacidades, skills e detalhes do protocolo
2. **Recepção da Requisição**: O `MessageController` recebe a requisição A2A JSON-RPC `sendMessage` no endpoint raiz
3. **Execução**: Roteada para o bean `AgentExecutor` (o Spring AI A2A fornece a implementação padrão: `DefaultAgentExecutor`)
4. **Invocação do Handler**: O lambda `ChatClientExecutorHandler` é invocado com o `ChatClient` do Spring AI e o contexto da requisição
5. **Resposta do Spring AI ChatClient**: A resposta do `ChatClient` do Spring AI é empacotada como uma mensagem JSON-RPC A2A

---

## Primeiros Passos

Vamos construir sistemas de agentes interoperáveis com o Spring AI A2A. Começaremos com os pré-requisitos e a configuração, depois trabalharemos com exemplos práticos: um servidor de agente único e uma orquestração multi-agente.

### Pré-requisitos

- Java 17 ou superior
- Spring Boot 4.0.1
- Spring AI 2.0.0-M2
- Um provedor de LLM (OpenAI, Anthropic, etc.)

### Dependências

Adicione o starter Spring AI A2A ao seu projeto para expor seu agente Spring AI como um servidor A2A:

```xml
<dependency>
   <groupId>org.springaicommunity</groupId>
   <artifactId>spring-ai-a2a-server-autoconfigure</artifactId>
   <version>0.2.0</version>
</dependency>
```

Este starter inclui o A2A Java SDK (v0.3.3.Final) como dependência transitiva.

Para aplicações que precisam chamar agentes A2A remotos, adicione explicitamente o cliente do A2A SDK:

```xml
<dependency>
   <groupId>io.github.a2asdk</groupId>
   <artifactId>a2a-java-sdk-client</artifactId>
   <version>0.3.3.Final</version>
</dependency>
```

### Configuração

Configure sua aplicação no `application.properties`:

```properties
# Configuração do servidor
server.servlet.context-path=/weather

# Configuração do LLM (exemplo com Anthropic Claude)
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-sonnet-4-5-20250929

# Para orquestração multi-agente (Exemplo 2), configure as URLs dos agentes remotos:
# remote.agents.urls=http://localhost:10001/foo/,http://localhost:10002/bar/
```

---

## Exemplo 1: Servidor de Agente

Exponha uma aplicação Spring AI com ferramentas como um servidor A2A:

```java
@Configuration
public class WeatherAgentConfiguration {

   @Bean
   public AgentCard agentCard(@Value("${server.port:8080}") int port,
           @Value("${server.servlet.context-path:/}") String contextPath) {
       // Este AgentCard é automaticamente exposto em /.well-known/agent-card.json
       // Outros agentes descobrem as capacidades deste agente por meio deste endpoint
       return new AgentCard.Builder()
           .name("Agente de Clima")
           .description("Fornece informações climáticas para cidades")
           .url("http://localhost:" + port + contextPath + "/")
           .version("1.0.0")
           .capabilities(new AgentCapabilities.Builder().streaming(false).build())
           .defaultInputModes(List.of("text"))
           .defaultOutputModes(List.of("text"))
           .skills(List.of(new AgentSkill.Builder()
               .id("weather_search")
               .name("Buscar clima")
               .description("Obter temperatura para qualquer cidade")
               .tags(List.of("weather"))
               .examples(List.of("Qual é o clima em São Paulo?"))
               .build()))
           .protocolVersion("0.3.0")
           .build();
   }

   @Bean
   public AgentExecutor agentExecutor(ChatClient.Builder chatClientBuilder, WeatherTools weatherTools) {

       ChatClient chatClient = chatClientBuilder.clone()
           .defaultSystem("Você é um assistente de clima. Use a ferramenta de temperatura para responder perguntas.")
           .defaultTools(weatherTools)  // Registra ferramentas Spring AI
           .build();

       return new DefaultAgentExecutor(chatClient, (chat, requestContext) -> {
           String userMessage = DefaultAgentExecutor.extractTextFromMessage(requestContext.getMessage());
           return chat.prompt().user(userMessage).call().content();
       });
   }
}

@Service
class WeatherTools {
    ...
}
```

Seu agente Spring AI agora é um servidor compatível com A2A. Outros agentes podem **descobrir** suas capacidades por meio do endpoint padrão `/.well-known/agent-card.json` e então enviar consultas de clima via POST para o endpoint raiz. A autoconfiguração se encarrega de expor tanto o AgentCard quanto os endpoints de mensagens.

---

## Exemplo 2: Cliente de Agente

Aqui está um exemplo prático de um cliente de agente: um agente host que orquestra agentes especializados para planejamento de viagens (acomodações no Airbnb e informações climáticas).

```java
@Service
public class RemoteAgentConnections {

   private final Map<String, AgentCard> agentCards = new HashMap<>();

   public RemoteAgentConnections(@Value("${remote.agents.urls}") List<String> agentUrls) {
       // Descobre agentes remotos na inicialização (veja a seção de Descoberta de Agente acima)
       for (String url : agentUrls) {
           String path = new URI(url).getPath();
           AgentCard card = A2A.getAgentCard(url, path + ".well-known/agent-card.json", null);
           this.agentCards.put(card.name(), card);
       }
   }

   @Tool(description = "Envia uma tarefa para um agente remoto. Use isso para delegar trabalho a agentes especializados.")
   public String sendMessage(
           @ToolParam(description = "O nome do agente") String agentName,
           @ToolParam(description = "A descrição da tarefa a enviar") String task) {

       AgentCard agentCard = this.agentCards.get(agentName);

       // Cria mensagem A2A
       Message message = new Message.Builder()
           .role(Message.Role.USER)
           .parts(List.of(new TextPart(task, null)))
           .build();

       // Usa o Cliente do A2A Java SDK
       CompletableFuture<String> responseFuture = new CompletableFuture<>();

       Client client = Client.builder(agentCard)
           .clientConfig(new ClientConfig.Builder()
               .setAcceptedOutputModes(List.of("text"))
               .build())
           .withTransport(JSONRPCTransport.class, new JSONRPCTransportConfig())
           .addConsumers(List.of(consumer -> {
               if (consumer instanceof TextPart textPart) {
                   responseFuture.complete(textPart.getText());
               }
           }))
           .build();

       client.sendMessage(message);
       return responseFuture.get(60, TimeUnit.SECONDS);
   }

   public String getAgentDescriptions() {
       return agentCards.values().stream()
           .map(card -> card.name() + ": " + card.description())
           .collect(Collectors.joining("\n"));
   }
}

@Configuration
public class HostAgentConfiguration {

   @Bean
   public ChatClient routingChatClient(ChatClient.Builder chatClientBuilder,
           RemoteAgentConnections remoteAgentConnections) {

       String systemPrompt = """
           Você coordena tarefas entre agentes especializados.
           Agentes disponíveis:
           %s
           Use a ferramenta sendMessage para delegar tarefas ao agente apropriado.
           """.formatted(remoteAgentConnections.getAgentDescriptions());

       return chatClientBuilder
           .defaultSystem(systemPrompt)
           .defaultTools(remoteAgentConnections)  // Registra como ferramenta Spring AI
           .build();
   }
}
```

**O que está acontecendo aqui:**

1. O agente host (cliente) **descobre** agentes remotos na inicialização buscando seus `AgentCard` nos endpoints padrão `.well-known/agent-card.json`
2. `RemoteAgentConnections` é registrado como uma `@Tool` do Spring AI no `ChatClient`
3. Quando um usuário pergunta "Planeje uma viagem para Londres", o LLM decide quais agentes chamar via a ferramenta `sendMessage`
4. A ferramenta usa o **Cliente do A2A Java SDK** para se comunicar com agentes remotos
5. Os resultados são agregados e retornados ao usuário

Esse padrão permite roteamento orientado por LLM — o modelo decide quais agentes especializados invocar com base na consulta do usuário.

---

## O Que Vem a Seguir

Embora a versão atual se concentre na integração do lado do servidor, a comunidade Spring AI está explorando oportunidades para melhorar a experiência do cliente A2A em aplicações Spring AI.

**Melhorias Futuras Potenciais:**

- **Segurança**: Suporte para autenticação e autorização A2A
- **Descoberta de Agentes**: Autoconfiguração do Spring Boot para descoberta e roteamento para agentes A2A
- **Autoconfiguração do Cliente**: Conexões de cliente A2A autoconfiguradas com abstrações amigáveis ao Spring
- **Suporte a Múltiplos Transportes**: SSE (Server-Sent Events) para respostas de streaming em tempo real, expandindo além da implementação atual de JSON-RPC
- **Observabilidade Aprimorada**: Integração com o Spring Boot Actuator para monitoramento de interações A2A

Essas melhorias forneceriam suporte adicional para construção de aplicações Spring AI que atuam como clientes de agentes A2A, proporcionando padrões de integração consistentes tanto para implementações de cliente quanto de servidor.

**Quer contribuir?** O projeto Spring AI A2A acolhe contribuições da comunidade. Confira o [repositório GitHub](https://github.com/spring-ai-community/spring-ai-a2a) para participar.

---

## Conclusão

O Protocolo A2A representa um passo significativo em direção a ecossistemas de agentes de IA interoperáveis. Ao padronizar como os agentes se comunicam, ele remove barreiras para a construção de sistemas multi-agente sofisticados.

O projeto comunitário Spring AI A2A fornece a integração necessária para participar desse ecossistema. Por meio da autoconfiguração do Spring Boot, você pode expor seus agentes Spring AI como servidores A2A, integrar-se a outros agentes compatíveis com A2A e construir padrões de orquestração que aproveitam as convenções do Spring Boot.

À medida que o ecossistema A2A cresce, mais agentes, ferramentas e plataformas podem adotar esse padrão, expandindo as opções de colaboração e composição de agentes.

Para começar a integrar o suporte ao Protocolo A2A com seus agentes Spring AI, consulte os recursos abaixo.

---

## Recursos

- [Especificação do Protocolo A2A](https://a2a-protocol.org/)
- [A2A Java SDK](https://github.com/a2aproject/a2a-java)
- [Projeto Comunitário Spring AI A2A](https://github.com/spring-ai-community/spring-ai-a2a)
- [Documentação do Spring AI](https://docs.spring.io/spring-ai/reference/)
- [Exemplo: Sistema Multi-Agente de Planejamento de Viagens](https://github.com/spring-ai-community/spring-ai-a2a/tree/main/spring-ai-a2a-examples/airbnb-planner)

---

## Links da Série

- **Parte 1**: [Agent Skills](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills) — Capacidades modulares e reutilizáveis
- **Parte 2**: [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) — Fluxos interativos
- **Parte 3**: [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/) — Planejamento estruturado
- **Parte 4**: [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) — Arquiteturas de agentes hierárquicas
- **Parte 5**: Integração A2A (este post) — Construindo agentes interoperáveis com o protocolo Agent2Agent
- **Em breve**: Subagent Extension Framework — Orquestração de agentes agnóstica de protocolo

---

## Blogs Relacionados do Spring AI

- [Dynamic Tool Discovery](https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov) — Alcance 34-64% de economia de tokens
- [Tool Argument Augmentation](https://spring.io/blog/2025/12/23/spring-ai-tool-argument-augmenter-tzolov) — Capture o raciocínio do LLM durante a execução de ferramentas