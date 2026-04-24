# Construindo Agentes Eficazes com Spring AI (Parte 1)

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 21 de janeiro de 2025 | **Leitura:** 6 min

Em uma publicação de pesquisa recente — [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — a Anthropic compartilhou percepções valiosas sobre a construção de agentes eficazes com Modelos de Linguagem de Grande Porte (LLMs). O que torna essa pesquisa particularmente interessante é sua ênfase em simplicidade e composabilidade em vez de frameworks complexos. Vamos explorar como esses princípios se traduzem em implementações práticas usando o [Spring AI](https://docs.spring.io/spring-ai/reference/index.html).

![Sistemas Agentivos](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/spring-ai-agentic-systems.jpg)

Embora as descrições dos padrões e os diagramas sejam provenientes da publicação original da Anthropic, vamos nos concentrar em como implementar esses padrões usando os recursos de portabilidade de modelos e saída estruturada do Spring AI. Recomendamos ler o artigo original primeiro.

O projeto [agentic-patterns](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns) implementa os padrões discutidos abaixo.

---

## Sistemas Agentivos

A publicação de pesquisa faz uma distinção arquitetural importante entre dois tipos de sistemas agentivos:

1. **Workflows**: Sistemas onde LLMs e ferramentas são orquestrados por caminhos de código predefinidos (sistemas prescritivos)
2. **Agentes**: Sistemas onde LLMs direcionam dinamicamente seus próprios processos e uso de ferramentas

A percepção central é que, embora agentes totalmente autônomos possam parecer atraentes, os workflows frequentemente oferecem melhor previsibilidade e consistência para tarefas bem definidas. Isso se alinha perfeitamente com requisitos corporativos onde confiabilidade e manutenibilidade são cruciais.

Vamos examinar como o Spring AI implementa esses conceitos por meio de cinco padrões fundamentais, cada um servindo a casos de uso específicos:

---

## 1. Chain Workflow (Fluxo em Cadeia)

O padrão Chain Workflow exemplifica o princípio de decompor tarefas complexas em etapas mais simples e gerenciáveis.

![Prompt Chaining Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

**Quando Usar:**

- Tarefas com etapas sequenciais claras
- Quando se deseja trocar latência por maior precisão
- Quando cada etapa se baseia na saída da etapa anterior

Aqui está um exemplo prático da implementação do Spring AI:

```java
public class ChainWorkflow {
    private final ChatClient chatClient;
    private final String[] systemPrompts;

    // Processa a entrada por uma série de prompts, onde a saída de cada etapa
    // se torna entrada para a próxima etapa na cadeia.
    public String chain(String userInput) {
        String response = userInput;
        for (String prompt : systemPrompts) {
            // Combina o prompt de sistema com a resposta anterior
            String input = String.format("{%s}\n {%s}", prompt, response);
            // Processa pelo LLM e captura a saída
            response = chatClient.prompt(input).call().content();
        }
        return response;
    }
}
```

Esta implementação demonstra vários princípios-chave:

- Cada etapa tem uma responsabilidade focada
- A saída de uma etapa se torna entrada para a próxima
- A cadeia é facilmente extensível e manutenível

---

## 2. Parallelization Workflow (Fluxo de Paralelização)

LLMs podem trabalhar simultaneamente em tarefas e ter suas saídas agregadas programaticamente. O fluxo de paralelização se manifesta em duas variações principais:

- **Segmentação**: Dividir tarefas em subtarefas independentes para processamento paralelo
- **Votação**: Executar múltiplas instâncias da mesma tarefa para obter consenso

![Parallelization Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

**Quando Usar:**

- Processamento de grandes volumes de itens similares, mas independentes
- Tarefas que requerem múltiplas perspectivas independentes
- Quando o tempo de processamento é crítico e as tarefas são paralelizáveis

O padrão Parallelization Workflow demonstra o processamento concorrente eficiente de múltiplas operações de LLM. Este padrão é particularmente útil para cenários que exigem execução paralela de chamadas de LLM com agregação automatizada de saídas.

Aqui está um exemplo básico de uso do Parallelization Workflow:

```java
List<String> parallelResponse = new ParallelizationWorkflow(chatClient)
    .parallel(
        "Analise como as mudanças de mercado impactarão este grupo de stakeholders.",
        List.of(
            "Clientes: ...",
            "Funcionários: ...",
            "Investidores: ...",
            "Fornecedores: ..."
        ),
        4
    );
```

Este exemplo demonstra o processamento paralelo de análise de stakeholders, onde cada grupo é analisado de forma concorrente.

---

## 3. Routing Workflow (Fluxo de Roteamento)

O padrão de Roteamento implementa distribuição inteligente de tarefas, permitindo tratamento especializado para diferentes tipos de entrada.

![Routing Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

Este padrão é projetado para tarefas complexas onde diferentes tipos de entradas são melhor tratados por processos especializados. Usa um LLM para analisar o conteúdo da entrada e roteá-lo para o prompt ou handler especializado mais apropriado.

**Quando Usar:**

- Tarefas complexas com categorias distintas de entrada
- Quando diferentes entradas requerem processamento especializado
- Quando a classificação pode ser tratada com precisão

Aqui está um exemplo básico de uso do Routing Workflow:

```java
@Autowired
private ChatClient chatClient;

// Cria o workflow
RoutingWorkflow workflow = new RoutingWorkflow(chatClient);

// Define prompts especializados para diferentes tipos de entrada
Map<String, String> routes = Map.of(
    "cobranca", "Você é um especialista em faturamento. Ajude a resolver problemas de cobrança...",
    "tecnico", "Você é um engenheiro de suporte técnico. Ajude a resolver problemas técnicos...",
    "geral", "Você é um representante de atendimento ao cliente. Ajude com consultas gerais..."
);

// Processa a entrada
String input = "Minha conta foi cobrada duas vezes na semana passada";
String response = workflow.route(input, routes);
```

---

## 4. Orchestrator-Workers (Orquestrador-Trabalhadores)

Este padrão demonstra como implementar comportamento mais complexo, semelhante a agentes, mantendo o controle:

- Um LLM central orquestra a decomposição de tarefas
- Trabalhadores especializados lidam com subtarefas específicas
- Limites claros mantêm a confiabilidade do sistema

![Orchestration Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

**Quando Usar:**

- Tarefas complexas onde as subtarefas não podem ser previstas antecipadamente
- Tarefas que requerem diferentes abordagens ou perspectivas
- Situações que necessitam de resolução de problemas adaptativa

A implementação usa o `ChatClient` do Spring AI para interações com LLMs e consiste em:

```java
public class OrchestratorWorkersWorkflow {
    public WorkerResponse process(String taskDescription) {
        // 1. O orquestrador analisa a tarefa e determina as subtarefas
        OrchestratorResponse orchestratorResponse = // ...

        // 2. Os trabalhadores processam as subtarefas em paralelo
        List<String> workerResponses = // ...

        // 3. Os resultados são combinados em uma resposta final
        return new WorkerResponse(/*...*/);
    }
}
```

**Exemplo de Uso:**

```java
ChatClient chatClient = // ... inicializa o chat client
OrchestratorWorkersWorkflow workflow = new OrchestratorWorkersWorkflow(chatClient);

// Processa uma tarefa
WorkerResponse response = workflow.process(
    "Gere documentação técnica e amigável para um endpoint de API REST"
);

// Acessa os resultados
System.out.println("Análise: " + response.analysis());
System.out.println("Saídas dos Trabalhadores: " + response.workerResponses());
```

---

## 5. Evaluator-Optimizer (Avaliador-Otimizador)

O padrão Evaluator-Optimizer implementa um processo de dois LLMs onde um modelo gera respostas enquanto outro fornece avaliação e feedback em um loop iterativo, similar ao processo de refinamento de um escritor humano. O padrão consiste em dois componentes principais:

- **LLM Gerador**: Produz respostas iniciais e as refina com base no feedback
- **LLM Avaliador**: Analisa as respostas e fornece feedback detalhado para melhoria

![Evaluator-Optimizer Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

**Quando Usar:**

- Existem critérios de avaliação claros
- O refinamento iterativo oferece valor mensurável
- As tarefas se beneficiam de múltiplas rodadas de crítica

A implementação usa o `ChatClient` do Spring AI para interações com LLMs e consiste em:

```java
public class EvaluatorOptimizerWorkflow {
    public RefinedResponse loop(String task) {
        // 1. Gera solução inicial
        Generation generation = generate(task, context);
        
        // 2. Avalia a solução
        EvaluationResponse evaluation = evaluate(generation.response(), task);
        
        // 3. Se APROVADO, retorna a solução
        // 4. Se PRECISA_MELHORAR, incorpora o feedback e gera nova solução
        // 5. Repete até estar satisfatório
        return new RefinedResponse(finalSolution, chainOfThought);
    }
}
```

**Exemplo de Uso:**

```java
ChatClient chatClient = // ... inicializa o chat client
EvaluatorOptimizerWorkflow workflow = new EvaluatorOptimizerWorkflow(chatClient);

// Processa uma tarefa
RefinedResponse response = workflow.loop(
    "Crie uma classe Java implementando um contador thread-safe"
);

// Acessa os resultados
System.out.println("Solução Final: " + response.solution());
System.out.println("Evolução: " + response.chainOfThought());
```

---

## Vantagens da Implementação do Spring AI

A implementação do Spring AI desses padrões oferece vários benefícios alinhados com as recomendações da Anthropic:

**1. [Portabilidade de Modelos](https://docs.spring.io/spring-ai/reference/api/chat/comparison.html)**

```xml
<!-- Troca fácil de modelos por meio de dependências -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

**2. [Saída Estruturada](https://docs.spring.io/spring-ai/reference/api/structured-output-converter.html)**

```java
// Tratamento type-safe de respostas de LLM
EvaluationResponse response = chatClient.prompt(prompt)
    .call()
    .entity(EvaluationResponse.class);
```

**3. [API Consistente](https://docs.spring.io/spring-ai/reference/api/chatclient.html)**

- Interface uniforme entre diferentes provedores de LLM
- Tratamento de erros e retentativas integrados
- Gerenciamento flexível de prompts

---

## Melhores Práticas e Recomendações

Com base tanto na pesquisa da Anthropic quanto nas implementações do Spring AI, aqui estão as principais recomendações para construir sistemas eficazes baseados em LLM:

**Comece Simples**
- Comece com workflows básicos antes de adicionar complexidade
- Use o padrão mais simples que atenda aos seus requisitos
- Adicione sofisticação apenas quando necessário

**Projete para Confiabilidade**
- Implemente tratamento de erros claro
- Use respostas type-safe sempre que possível
- Incorpore validação em cada etapa

**Considere os Trade-offs**
- Equilibre latência versus precisão
- Avalie quando usar processamento paralelo
- Escolha entre workflows fixos e agentes dinâmicos

---

## Trabalho Futuro

Na Parte 2 desta série, exploraremos como construir Agentes mais avançados que combinam esses padrões fundamentais com recursos sofisticados:

**Composição de Padrões**
- Combinando múltiplos padrões para criar workflows mais poderosos
- Construindo sistemas híbridos que aproveitam os pontos fortes de cada padrão
- Criando arquiteturas flexíveis que podem se adaptar a requisitos em constante mudança

**Gerenciamento Avançado de Memória de Agentes**
- Implementando memória persistente entre conversas
- Gerenciando janelas de contexto de forma eficiente
- Desenvolvendo estratégias para retenção de conhecimento de longo prazo

**Ferramentas e Integração com Model Context Protocol (MCP)**
- Aproveitando ferramentas externas por meio de interfaces padronizadas
- Implementando MCP para interações aprimoradas com modelos
- Construindo arquiteturas de agentes extensíveis

Fique ligado para implementações detalhadas e melhores práticas para esses recursos avançados.

---

## Tanzu Gen AI Solutions

O [VMware Tanzu Platform 10](https://blogs.vmware.com/tanzu/broadcom-announces-the-general-availability-of-vmware-tanzu-platform-10-making-it-easier-for-customers-to-build-and-launch-new-applications-in-the-private-cloud/) com o Tanzu AI Server, impulsionado pelo Spring AI, fornece:

- **Implantação de IA de Nível Corporativo**: Solução pronta para produção para implantar aplicações de IA em seu ambiente VMware Tanzu
- **Acesso Simplificado a Modelos**: Acesso simplificado aos modelos Amazon Bedrock Nova por meio de uma interface unificada
- **Segurança e Governança**: Controles de segurança e recursos de governança de nível corporativo
- **Infraestrutura Escalável**: Construída sobre o Spring AI, a integração suporta implantação escalável de aplicações de IA mantendo alto desempenho

Para mais informações sobre a implantação de aplicações de IA com o Tanzu AI Server, visite a [documentação do VMware Tanzu AI](https://www.vmware.com/solutions/app-platform/ai).

---

## Conclusão

A combinação das percepções de pesquisa da Anthropic com as implementações práticas do Spring AI fornece um framework poderoso para construir sistemas eficazes baseados em LLM. Seguindo esses padrões e princípios, os desenvolvedores podem criar aplicações de IA robustas, manuteníveis e eficazes que entregam valor real enquanto evitam complexidade desnecessária.

A chave é lembrar que às vezes a solução mais simples é a mais eficaz. Comece com padrões básicos, entenda seu caso de uso completamente e só adicione complexidade quando isso demonstravelmente melhora o desempenho ou as capacidades do seu sistema.

---

## Recursos

- **Projeto de Exemplos**: [agentic-patterns](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns)
- **Pesquisa original**: [Building effective agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
- **Documentação Spring AI**: [docs.spring.io/spring-ai](https://docs.spring.io/spring-ai/reference/index.html)
- **Portabilidade de Modelos**: [Comparação de modelos Spring AI](https://docs.spring.io/spring-ai/reference/api/chat/comparison.html)
- **Saída Estruturada**: [Structured Output Converter](https://docs.spring.io/spring-ai/reference/api/structured-output-converter.html)
- **API ChatClient**: [Documentação ChatClient](https://docs.spring.io/spring-ai/reference/api/chatclient.html)

### Padrões Implementados

- [Chain Workflow](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/chain-workflow) — Fluxo em cadeia sequencial
- [Parallelization Workflow](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/parallelization-workflow) — Processamento paralelo de LLMs
- [Routing Workflow](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/routing-workflow) — Roteamento inteligente de tarefas
- [Orchestrator-Workers](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/orchestrator-workers) — Orquestração hierárquica
- [Evaluator-Optimizer](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/evaluator-optimizer) — Refinamento iterativo com avaliação