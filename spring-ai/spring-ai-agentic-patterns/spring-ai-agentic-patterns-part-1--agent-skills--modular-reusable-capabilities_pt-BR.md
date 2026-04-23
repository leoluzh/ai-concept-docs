# Padrões Agentivos com Spring AI (Parte 1): Agent Skills — Capacidades Modulares e Reutilizáveis

**Categoria:** Engenharia | **Autor:** Christian Tzolov | **Data:** 13 de janeiro de 2026 | **Leitura:** 8 min

> *Skills Agnósticas de LLM que Rodam no Seu Ambiente*

![Agent Skills](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260105/spring-ai-agent-skills.png)

Agent Skills são pastas modulares de instruções, scripts e recursos que agentes de IA podem descobrir e carregar sob demanda. Em vez de codificar conhecimento diretamente nos prompts ou criar ferramentas especializadas para cada tarefa, as skills oferecem uma forma flexível de estender as capacidades dos agentes.

A implementação do Spring AI traz as Agent Skills para o ecossistema Java, garantindo portabilidade de LLM — defina suas skills uma vez e use-as com OpenAI, Anthropic, Google Gemini ou qualquer outro modelo suportado.

**Este é o primeiro post da nossa série Padrões Agentivos com Spring AI.** A série explora o toolkit [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils) — um extenso conjunto de padrões agentivos para o Spring AI, inspirado no [Claude Code](https://code.claude.com/docs/en/overview). Abordaremos Agent Skills (este post), seguido de Gerenciamento de Tarefas, AskUserQuestion para fluxos interativos e Sub-Agentes Hierárquicos para sistemas multi-agente complexos.

🚀 **Quer ir direto ao ponto?** Pule para a seção de [Primeiros Passos](#primeiros-passos).

---

## O que São Agent Skills?

Agent Skills são capacidades modulares empacotadas como arquivos Markdown com `frontmatter YAML`. Cada skill é uma pasta contendo um arquivo `SKILL.md` com metadados (`name` e `description`, no mínimo) e instruções que dizem ao agente como realizar uma tarefa específica. Skills também podem incluir scripts, templates e materiais de referência. O frontmatter suporta tanto valores de string simples quanto estruturas YAML complexas (listas, objetos aninhados) para casos de uso avançados.

```
minha-skill/
├── SKILL.md          # Obrigatório: instruções + metadados
├── scripts/          # Opcional: código executável
├── references/       # Opcional: documentação
└── assets/           # Opcional: templates, recursos
```

As Skills usam **divulgação progressiva** para gerenciar o contexto com eficiência:

1. **Descoberta**: Na inicialização, os agentes carregam apenas o nome e a descrição de cada skill disponível — o suficiente para saber quando pode ser relevante.
2. **Ativação**: Quando uma tarefa corresponde à descrição de uma skill, o agente lê as instruções completas do `SKILL.md` para o contexto.
3. **Execução**: O agente segue as instruções, opcionalmente carregando arquivos referenciados ou executando código incluído conforme necessário.

Essa abordagem permite registrar centenas de skills mantendo a janela de contexto enxuta.

> **💡 Dica:** Saiba mais sobre Agent Skills na [especificação oficial](https://agentskills.io/specification).

---

## Por Que Usar Agent Skills no Spring AI

**Integração Perfeita** — Adicione Agent Skills à sua aplicação Spring AI existente simplesmente registrando algumas ferramentas — sem necessidade de mudanças arquiteturais.

**Portabilidade e Agnóstico de Modelo — Sem Dependência de Fornecedor** — Diferente de implementações vinculadas a plataformas LLM específicas, esta implementação do Spring AI funciona com vários provedores de LLM, permitindo trocar de modelo sem reescrever código ou skills.

**Reutilizável e Combinável** — Skills podem ser compartilhadas entre projetos, versionadas com seu código, combinadas para criar fluxos de trabalho complexos e estendidas com scripts auxiliares e materiais de referência. O Spring AI Skills suporta nativamente qualquer Skill existente do Claude Code.

**Ferramentas Relacionadas do Spring AI:** Agent Skills funcionam bem com outros recursos baseados em ferramentas do Spring AI, como [Dynamic Tool Discovery](https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov) para seleção eficiente de ferramentas e [Tool Argument Augmentation](https://spring.io/blog/2025/12/23/spring-ai-tool-argument-augmenter-tzolov) para capturar o raciocínio do LLM durante a execução de skills.

---

## Como Funcionam as Skills no Spring AI

O Spring AI usa a [abordagem de integração baseada em ferramentas](https://agentskills.io/integrate-skills#integration-approaches), implementando ferramentas que permitem a qualquer LLM acionar skills e acessar recursos incluídos. A implementação segue de perto as especificações de ferramentas do [Claude Code](https://code.claude.com/docs/en/settings#tools-available-to-claude) para `Skills`, `Bash` e `Read`.

O conjunto principal de ferramentas é: [SkillsTool](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/SkillsTool.md) (obrigatório), [ShellTools](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/ShellTools.md) (opcional) e [FileSystemTools](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/FileSystemTools.md) (opcional). O `SkillsTool` fornece uma função `Skill` que permite aos modelos de IA descobrir e carregar skills especificadas sob demanda, funcionando em conjunto com `FileSystemTools` (para leitura de arquivos de referência) e `ShellTools` (para execução de scripts auxiliares).

As Skills operam por meio de um processo de três etapas:

**1. Descoberta (na inicialização)** — Durante a inicialização, o `SkillsTool` escaneia os diretórios de skills configurados (como `.claude/skills/`) e analisa o frontmatter YAML de cada arquivo `SKILL.md`. Ele extrai os campos `name` e `description` para construir um registro leve de skills que é embutido diretamente na descrição da ferramenta `Skill`, tornando-o visível ao LLM sem consumir contexto de conversa.

![Figura 1: Fluxo de descoberta e registro de skills mostrando como os arquivos SKILL.md são analisados e registrados na inicialização](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260105/skillstool1.png)

**2. Correspondência Semântica (durante a conversa)** — Quando um usuário faz uma solicitação, o LLM examina as descrições de skills embutidas na definição da ferramenta. Se o LLM determinar que a solicitação do usuário corresponde semanticamente à descrição de uma skill, ele invoca a ferramenta `Skill` com o nome da skill como parâmetro.

**3. Execução (ao invocar a skill)** — Quando a ferramenta `Skill` é chamada, o `SkillsTool` carrega o conteúdo completo do `SKILL.md` do disco e o retorna ao LLM junto com o caminho do diretório base da skill. O LLM então segue as instruções no conteúdo da skill. Se a skill referencia arquivos adicionais ou scripts auxiliares, o LLM usa a função `Read` do `FileSystemTools` ou a função `Bash` do `ShellTools` para acessá-los sob demanda.

---

## Skills em Ação

Esta seção demonstra como as skills funcionam na prática com exemplos do mundo real.

### Exemplo: Skills com Referências e Scripts

O carregamento sob demanda do Passo 3 se torna poderoso quando as skills incluem recursos adicionais. Skills podem conter arquivos de referência com instruções complementares e scripts executáveis para processamento de dados — tudo carregado apenas quando necessário.

Aqui está um exemplo da skill `my-skill` que inclui um auxiliar de extração de transcrição do YouTube e instruções suplementares em `research_methodology.md`:

**Estrutura do Diretório da Skill:**

```
.claude/skills/my-skill/
├── SKILL.md
├── scripts/
│   └── get_youtube_transcript.py
└── research_methodology.md
```

**No SKILL.md:**

```
...
**Se o conceito for desconhecido ou exigir pesquisa:**
Carregue `research_methodology.md` para orientações detalhadas.

**Se o usuário fornecer um vídeo do YouTube:**
Execute `uv run scripts/get_youtube_transcript.py <url_ou_id_do_video>`
para obter a transcrição do vídeo.
...
```

Quando um usuário pergunta "Explique os conceitos deste vídeo: <https://youtube.com/watch?v=abc123>. Siga a metodologia de pesquisa", a IA:

1. Invoca a skill `my-skill` e carrega o conteúdo do seu `SKILL.md`
2. Reconhece a necessidade da metodologia de pesquisa e usa `Read` para carregar `research_methodology.md`
3. Reconhece a URL do YouTube e usa `Bash` para executar o script auxiliar via `ShellTools`
4. Usa a transcrição do vídeo para explicar os conceitos seguindo as instruções da metodologia de pesquisa

![Figura 2: Fluxo de execução da skill mostrando como o LLM usa FileSystemTools e ShellTools para acessar recursos da skill](https://raw.githubusercontent.com/spring-io/spring-io-static/refs/heads/main/blog/tzolov/20260105/skillstool2.png)

O código do script nunca entra na janela de contexto — apenas a saída entra, tornando essa abordagem altamente eficiente em tokens.

💡 **Demo:** Confira o [Skills-Demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/skills-demo) que implementa esse fluxo de trabalho.

> ⚠️ **Nota de Segurança:** Os scripts são executados diretamente na sua máquina local sem isolamento. Você precisará pré-instalar quaisquer runtimes necessários (Python, Node.js, etc.). Para uma operação mais segura, considere executar sua aplicação agentiva em um container.

---

## Primeiros Passos

Pronto para adicionar Agent Skills ao seu projeto Spring AI?

### 1. Adicione a Dependência

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.4.2</version>
</dependency>
```

> **Nota:** Para a versão estável mais recente, verifique a [página de releases do GitHub](https://github.com/spring-ai-community/spring-ai-agent-utils/releases).
>
> **Nota:** Requer Spring AI `2.0.0-M2+`.

### 2. Configure Seu Agente

```java
@SpringBootApplication
public class Application {

    @Bean
    CommandLineRunner demo(ChatClient.Builder chatClientBuilder) {
        return args -> {
            ChatClient chatClient = chatClientBuilder
                .defaultToolCallbacks(SkillsTool.builder()
                    .addSkillsDirectory(".claude/skills")
                    .build())
                .defaultTools(FileSystemTools.builder().build())
                .defaultTools(ShellTools.builder().build())
                .build();

            String response = chatClient.prompt()
                .user("Sua tarefa aqui")
                .call()
                .content();
        };
    }
}
```

> **💡 Dica para Produção:** Para aplicações empacotadas, você pode carregar skills do classpath usando Spring Resources:
>
> ```java
> .defaultToolCallbacks(SkillsTool.builder()
>     .addSkillsResource(resourceLoader.getResource("classpath:.claude/skills"))
>     .build())
> ```
>
> Isso é particularmente útil ao distribuir skills como parte do seu deployment JAR/WAR.

### 3. Crie Sua Primeira Skill

```bash
mkdir -p .claude/skills/revisor-de-codigo
cat > .claude/skills/revisor-de-codigo/SKILL.md << 'EOF'
---
name: revisor-de-codigo
description: Revisa código Java buscando boas práticas, problemas de segurança e convenções do Spring Framework. Use quando o usuário pedir para revisar, analisar ou auditar código.
---

# Revisor de Código

## Instruções
Ao revisar código:
1. Verifique vulnerabilidades de segurança (SQL injection, XSS, etc.)
2. Valide boas práticas do Spring Boot (uso correto de @Service, @Repository, etc.)
3. Procure possíveis NullPointerExceptions
4. Sugira melhorias de legibilidade e manutenibilidade
5. Forneça feedback específico linha por linha com exemplos de código
EOF
```

### 4. Use Seu Agente com a Skill

```java
String response = chatClient.prompt()
    .user("Revise esta classe controller para boas práticas: " +
          "src/main/java/com/example/UserController.java")
    .call()
    .content();

System.out.println(response);
```

Ao executar isso, o LLM irá:

1. Associar "Revise este controller" com a descrição da skill `revisor-de-codigo`
2. Invocar a ferramenta `Skill` para carregar as instruções completas do `SKILL.md`
3. Usar a ferramenta `Read` (do `FileSystemTools`) para acessar o arquivo `UserController.java`
4. Seguir as instruções de revisão e fornecer feedback detalhado

As instruções da skill guiam o comportamento do LLM sem que você precise codificar a lógica de revisão nos seus prompts — basta atualizar o arquivo da skill para mudar como as revisões funcionam.

---

## Limitações Atuais

Embora a implementação de Agent Skills no Spring AI seja poderosa e flexível, há algumas limitações a se observar:

**Segurança na Execução de Scripts** — Scripts executados via `ShellTools` rodam diretamente na sua máquina local sem isolamento. Isso significa que código potencialmente inseguro pode acessar seu sistema de arquivos, rede ou recursos do sistema. Sempre revise os scripts de skills antes de usá-los, especialmente os de fontes terceiras. Considere executar sua aplicação agentiva em um ambiente containerizado (Docker, Kubernetes) para limitar a exposição.

**Sem Human-in-the-Loop** — Atualmente, não há um mecanismo integrado para exigir aprovação humana antes de executar skills ou scripts. O LLM pode invocar qualquer skill registrada e executar qualquer script incluído automaticamente. Para ambientes de produção que lidam com operações sensíveis, pode ser necessário implementar fluxos de aprovação personalizados usando os mecanismos de callback de ferramentas do Spring AI, por exemplo, um wrapper `ToolCallback`.

**Versionamento Limitado de Skills** — Não existe atualmente um sistema de versionamento integrado para skills. Se você atualizar o comportamento de uma skill, todas as aplicações que a usam passarão a usar a nova versão imediatamente. Para implantações em produção, considere implementar sua própria estratégia de versionamento por meio da estrutura de diretórios (ex: `.claude/skills/v1/`, `.claude/skills/v2/`).

---

## Relacionado: API Nativa de Skills da Anthropic

O Spring AI também se integra com a API nativa de Skills da Anthropic, que oferece uma abordagem diferente:

- Skills rodam no container cloud isolado da Anthropic (sem acesso à rede, apenas pacotes pré-instalados)
- Geração de documentos integrada: Excel, PowerPoint, Word, PDF
- Skills são enviadas para os servidores da Anthropic e compartilhadas em seu workspace
- Requer modelos Claude (Sonnet 4, Sonnet 4.5, Opus 4)

**Diferença principal:** Skills da Anthropic rodam na infraestrutura cloud da Anthropic; Generic Agent Skills rodam no seu ambiente.

Use as Skills nativas da Anthropic quando precisar de execução segura e isolada ou das capacidades integradas de geração de documentos. Use as Generic Agent Skills quando precisar de portabilidade de LLM, acesso a recursos locais ou quando quiser skills empacotadas com sua aplicação.

**Pode-se usar as duas?** Sim. Você pode usar as Skills nativas da Anthropic para geração de documentos enquanto usa as Generic Agent Skills para outras capacidades portáveis na mesma aplicação. Elas servem a propósitos diferentes e podem se complementar.

Veja o post [Anthropic Skills no Spring AI](https://spring.io/blog/2026/01/spring-ai-anthropic-agent-skills) para detalhes.

---

## Conclusão

Agent Skills trazem capacidades modulares e reutilizáveis para aplicações Spring AI sem dependência de fornecedor. Ao fornecer conhecimento de domínio sob demanda, você pode atualizar o comportamento do agente sem mudanças de código, compartilhar skills entre projetos e trocar de provedor de LLM sem problemas.

A implementação do `spring-ai-agent-utils` torna esse padrão acessível a desenvolvedores Java com uma abordagem simples baseada em ferramentas. Seja construindo assistentes de codificação, geradores de documentação ou agentes de domínio específico, as skills fornecem uma base escalável para organizar o conhecimento do agente.

**Isso é só o começo.** Os outros posts desta série aprofundam padrões agentivos avançados:

---

## Links da Série

- **Parte 1**: Agent Skills (este post) — Capacidades modulares e reutilizáveis
- **Parte 2**: [AskUserQuestionTool](https://spring.io/blog/2026/01/16/spring-ai-ask-user-question-tool) — Fluxos interativos onde agentes coletam preferências do usuário durante a execução
- **Parte 3**: [TodoWriteTool](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite/) — Fluxos de trabalho de agentes transparentes e rastreáveis com gerenciamento de tarefas em múltiplos passos
- **Parte 4**: [Orquestração de Subagentes](https://spring.io/blog/2026/01/27/spring-ai-agentic-patterns-4-task-subagents) — Arquiteturas multi-agente hierárquicas com janelas de contexto dedicadas
- **Parte 5**: [Integração A2A](https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration) — Construindo agentes interoperáveis com o protocolo Agent2Agent
- **Em breve**: Subagent Extension Framework — Orquestração de agentes agnóstica de protocolo (A2A, MCP, customizado)

---

## Recursos

### Spring AI Agent Utils Toolkit

- **Repositório GitHub**: [spring-ai-agent-utils](https://github.com/spring-ai-community/spring-ai-agent-utils)
- **Documentação Completa**: [README.md](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/README.md)
- **Documentação das Ferramentas**: [SkillsTool](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/SkillsTool.md), [FileSystemTools](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/FileSystemTools.md), [ShellTools](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/spring-ai-agent-utils/docs/ShellTools.md)
- **Documentação do Spring AI**: [docs.spring.io/spring-ai](https://docs.spring.io/spring-ai/reference/)

### Projetos de Exemplo

- [skills-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/skills-demo) — Demonstração focada em skills (este post)
- [code-agent-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/code-agent-demo) — Integração completa do toolkit (Partes 2-3)
- [subagent-demo](https://github.com/spring-ai-community/spring-ai-agent-utils/tree/main/examples/subagent-demo) — Agentes hierárquicos e integração A2A (Partes 4-5)

### Agent Skills

- **Especificação**: [agentskills.io](https://agentskills.io/specification)
- **Documentação do Claude Code**: [code.claude.com/docs](https://code.claude.com/docs/en/skills)

### Blogs Relacionados do Spring AI

- [Dynamic Tool Discovery](https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov) — Alcance 34-64% de economia de tokens
- [Tool Argument Augmentation](https://spring.io/blog/2025/12/23/spring-ai-tool-argument-augmenter-tzolov) — Capture o raciocínio do LLM durante a execução de ferramentas