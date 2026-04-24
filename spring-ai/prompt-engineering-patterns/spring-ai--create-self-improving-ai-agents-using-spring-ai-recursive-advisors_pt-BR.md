# Criando Agentes de IA Auto-Aprimoráveis com Advisors Recursivos do Spring AI

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 04 de novembro de 2025 | **Leitura:** 5 min

![Advisors Flow](https://docs.spring.io/spring-ai/reference/_images/advisors-flow.jpg)

O [ChatClient](https://docs.spring.io/spring-ai/reference/1.1.1/api/chatclient.html) do Spring AI oferece uma API fluente para comunicação com um modelo de IA. Essa API fluente fornece métodos para construir as partes constituintes de um prompt que é passado ao modelo de IA como entrada.

Os [Advisors](https://docs.spring.io/spring-ai/reference/1.1.1/api/advisors.html) são uma parte fundamental da API fluente que interceptam, modificam e aprimoram interações orientadas por IA. Os principais benefícios incluem o encapsulamento de padrões comuns de IA Generativa, a transformação de dados enviados e recebidos dos Modelos de Linguagem de Grande Porte (LLMs) e a portabilidade entre diversos modelos e casos de uso.

Os Advisors processam objetos `ChatClientRequest` e `ChatClientResponse`. O framework encadeia advisors pelos seus valores de `getOrder()` (valores menores executam primeiro), com o advisor final chamando o LLM.

O Spring AI fornece advisors integrados para [Memória de Conversa](https://docs.spring.io/spring-ai/reference/1.1.1/api/chat-memory.html#_memory_in_chat_client), [Geração Aumentada por Recuperação (RAG)](https://docs.spring.io/spring-ai/reference/1.1.1/api/retrieval-augmented-generation.html#_advisors), Logging e Guardrails. Os desenvolvedores também podem criar advisors personalizados.

Estrutura típica de um advisor:

```java
public class MyAdvisor implements CallAdvisor {

    // 1. Define a ordem do advisor
    public int getOrder() { return MY_ADVISOR_ORDER; }

    // 2. Define o nome do advisor
    public String getName() { return MY_ADVISOR_NAME; }

    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        // 3. Pré-processa a requisição (modificar, validar, registrar, etc.)
        request = preProcess(request);
        // 4. Chama o próximo advisor na cadeia
        ChatClientResponse response = chain.nextCall(request);
        // 5. Pós-processa a resposta (modificar, validar, registrar, etc.)
        return postProcess(response);
    }
}
```

---

## Advisors Recursivos

A partir da versão `1.1.0-M4`, o Spring AI introduz os [Advisors Recursivos](https://docs.spring.io/spring-ai/reference/1.1.1/api/advisors-recursive.html), que permitem percorrer a cadeia de advisors múltiplas vezes para suportar fluxos de trabalho iterativos:

- **Loops de chamada de ferramentas**: Executar múltiplas ferramentas em sequência, onde a saída de cada ferramenta informa a próxima decisão
- **Validação de saída**: Validar respostas estruturadas e tentar novamente com feedback quando a validação falha
- **Lógica de retry**: Refinar requisições com base na qualidade da resposta ou em critérios externos
- **Fluxos de avaliação**: Avaliar e modificar respostas antes da entrega
- **Loops agentivos**: Construir agentes autônomos que planejam, executam e iteram sobre tarefas analisando resultados e determinando próximas ações até atingir um objetivo

Padrões tradicionais de advisor de passagem única não conseguem lidar adequadamente com esses cenários.

Os advisors recursivos percorrem a cadeia de advisors downstream múltiplas vezes, chamando repetidamente o LLM até que uma condição seja atendida.

O método `CallAdvisorChain.copy(CallAdvisor after)` cria uma sub-cadeia contendo apenas os advisors downstream, permitindo iteração controlada enquanto mantém ordenação e observabilidade adequadas, e prevenindo a re-execução de advisors upstream.

![Fluxo de Advisors Recursivos](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20251031/spring-ai-recursive-advisors-flow.png)

*O diagrama ilustra como os advisors recursivos permitem o processamento iterativo ao permitir que o fluxo percorra a cadeia de advisors múltiplas vezes. Diferente da execução tradicional de passagem única, os advisors recursivos podem revisitar advisors anteriores com base nas condições da resposta, criando fluxos de trabalho iterativos sofisticados entre o cliente, os advisors e o LLM.*

Aqui está o padrão básico para implementar um advisor recursivo:

```java
public class MyRecursiveAdvisor implements CallAdvisor {

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {

        // Chama a cadeia inicialmente
        ChatClientResponse response = chain.nextCall(request);

        // Verifica se precisa tentar novamente
        while (!isConditionMet(response)) {
            // Modifica a requisição com base na resposta
            ChatClientRequest modifiedRequest = modifyRequest(request, response);

            // Cria uma sub-cadeia e recorre
            response = chain.copy(this).nextCall(modifiedRequest);
        }

        return response;
    }
}
```

A diferença fundamental em relação a advisors não recursivos é o uso de `chain.copy(this).nextCall(...)` em vez de `chain.nextCall(...)` para iterar em uma cópia da cadeia interna. Isso garante que cada iteração percorra a cadeia downstream completa, permitindo que outros advisors observem e interceptem enquanto mantêm observabilidade adequada.

> ### ⚠️ Nota Importante
>
> **Advisors Recursivos são uma nova funcionalidade experimental no Spring AI 1.1.0-M4.** São apenas não-streaming, exigem ordenação cuidadosa dos advisors e podem aumentar custos devido a múltiplas chamadas de LLM.
>
> Tenha especial cuidado com advisors internos que mantêm estado externo — eles podem exigir atenção extra para manter a corretude entre iterações.
>
> Sempre defina condições de terminação e limites de retry para evitar loops infinitos.
>
> Considere se o seu caso de uso se beneficia mais dos advisors recursivos ou de implementar um loop `while` explícito em torno das chamadas do `ChatClient` no código da aplicação.

---

## Advisors Recursivos Integrados

O Spring AI 1.1.0-M4 fornece dois advisors recursivos integrados:

### ToolCallAdvisor

#### Suporte Padrão de Chamada de Ferramentas

Por padrão, a [Execução de Ferramentas do Spring AI](https://docs.spring.io/spring-ai/reference/1.1.1/api/tools.html#_framework_controlled_tool_execution) é implementada dentro de cada implementação de `ChatModel` usando um `ToolCallingManager`. Isso significa que as requisições e o fluxo de resposta de chamada de ferramentas são opacos para os Advisors do ChatClient, pois ocorrem fora da cadeia de execução dos advisors.

![Chamada de Ferramentas Padrão no ChatModel](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20251031/spring-ai-in-chatmodel-toolcalling.png)

#### ToolCallAdvisor

Aproveitando a [execução de ferramentas controlada pelo usuário](https://docs.spring.io/spring-ai/reference/1.1.1/api/tools.html#_user_controlled_tool_execution), o `ToolCallAdvisor` implementa o loop de chamada de ferramentas dentro da cadeia de advisors, fornecendo controle explícito sobre a execução de ferramentas em vez de delegar para a execução interna de ferramentas do modelo.

![Chamada de Ferramentas via Advisor](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20251031/spring-ai-advisor-toolcalling.png)

- Percorre em loop até que não haja mais chamadas de ferramentas
- Permite que outros advisors (como o Advisor ABC) interceptem e alterem cada requisição e resposta de chamada de ferramenta
- Suporta funcionalidade de "retorno direto" (`return direct`)
- Desabilita a execução interna de ferramentas do chat-model

**Exemplo de Uso:**

```java
var toolCallAdvisor = ToolCallAdvisor.builder()
    .toolCallingManager(toolCallingManager)
    .advisorOrder(BaseAdvisor.HIGHEST_PRECEDENCE + 300)
    .build();

public record Request(String location) {}

var weatherTool = FunctionToolCallback.builder("getWeather", (Request request) -> "15.0°C")
        .description("Obtém o clima para uma localização")
        .inputType(Request.class)
        .build();

var chatClient = ChatClient.builder(chatModel)
    .defaultToolCallbacks(weatherTool)  // Registro de ferramentas
    .defaultAdvisors(toolCallAdvisor)   // Execução de chamadas de ferramentas como Advisor
    .build();

String response = chatClient.prompt()
    .user("Qual é o clima em Paris e Amsterdam e converta a temperatura para Fahrenheit?")
    .call()
    .content();
```

**Nota**: Por padrão, a ordem do `ToolCallAdvisor` é definida próxima a `Ordered.HIGHEST_PRECEDENCE` para garantir que o advisor execute primeiro na cadeia (primeiro para processamento de requisição, último para processamento de resposta), permitindo que advisors internos interceptem e processem mensagens de requisição e resposta de ferramentas.

Quando uma execução de ferramenta tem `returnDirect=true`, o advisor executa a ferramenta, detecta o flag, interrompe o loop e retorna a saída diretamente ao cliente sem enviar ao LLM. Isso reduz a latência quando a saída da ferramenta é a resposta final.

> **💡 Projeto Demo**: Veja um exemplo funcional completo do `ToolCallAdvisor` no projeto [Recursive Advisor Demo](https://github.com/spring-projects/spring-ai-examples/tree/main/advisors/recursive-advisor-demo).

---

### StructuredOutputValidationAdvisor

O `StructuredOutputValidationAdvisor` valida saída JSON estruturada contra um schema gerado e tenta novamente se a validação falhar:

- Gera automaticamente schemas JSON a partir dos tipos de saída esperados
- Valida respostas do LLM contra o schema e tenta novamente com mensagens de erro de validação
- Número máximo de tentativas configurável
- Suporta `ObjectMapper` personalizado para processamento de JSON

**Exemplo de Uso:**

```java
public record ActorFilms(String actor, List<String> movies) {}

var validationAdvisor = StructuredOutputValidationAdvisor.builder()
    .outputType(ActorFilms.class)
    .maxRepeatAttempts(3)
    .build();

var chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(validationAdvisor)
    .build();

ActorFilms actorFilms = chatClient.prompt()
    .user("Gere a filmografia de Tom Hanks")
    .call()
    .entity(ActorFilms.class);
```

Quando a validação falha, o advisor aumenta o prompt com detalhes do erro e tenta novamente até o número máximo configurado de tentativas.

**Nota**: Por padrão, a [ordem](https://docs.spring.io/spring-ai/reference/api/advisors.html#_advisor_order) do `StructuredOutputValidationAdvisor` é definida próxima a `Ordered.LOWEST_PRECEDENCE` para garantir que o advisor execute próximo ao final da cadeia (mas antes da chamada ao modelo), ou seja, é o último para processamento de requisição e o primeiro para processamento de resposta.

---

## Melhores Práticas

- **Defina Condições de Terminação Claras**: Garanta que os loops tenham condições de saída definitivas para prevenir loops infinitos
- **Use Ordenação Adequada**: Posicione advisors recursivos no início da cadeia para permitir que outros advisors observem as iterações, ou no final para prevenir a observação
- **Forneça Feedback**: Aumente as requisições de retry com informações sobre o motivo da nova tentativa para ajudar o LLM a melhorar
- **Limite as Iterações**: Defina limites máximos de tentativas para prevenir execução descontrolada
- **Monitore a Execução**: Use os recursos de observabilidade do Spring AI para rastrear contagens de iterações e desempenho
- **Escolha a Abordagem Certa**: Avalie se os advisors recursivos ou loops `while` explícitos em torno das chamadas do `ChatClient` se adequam melhor ao seu caso de uso e arquitetura específicos

### Considerações de Desempenho

Os advisors recursivos aumentam o número de chamadas ao LLM, afetando:

- **Custo**: Mais chamadas à API aumentam os custos
- **Latência**: Múltiplas iterações adicionam tempo de processamento
- **Uso de Tokens**: Cada iteração consome tokens adicionais

Para otimizar, defina limites razoáveis de retry, monitore as contagens de iteração e faça cache de resultados intermediários quando possível.

---

## Recursos

- [Documentação de Advisors Recursivos](https://docs.spring.io/spring-ai/reference/api/advisors-recursive.html)
- [Guia da API de Advisors](https://docs.spring.io/spring-ai/reference/api/advisors.html)
- [Documentação da API ChatClient](https://docs.spring.io/spring-ai/reference/api/chatclient.html)
- [Repositório GitHub do Spring AI](https://github.com/spring-projects/spring-ai)
- [Projeto Demo: Recursive Advisor](https://github.com/spring-projects/spring-ai-examples/tree/main/advisors/recursive-advisor-demo)
- [Artigo relacionado: LLM-as-a-Judge com Advisors Recursivos](https://spring.io/blog/2025/11/10/spring-ai-llm-as-judge-blog-post)