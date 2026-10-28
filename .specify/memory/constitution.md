# Constituição do AI Hub

## Core Principles

### I. Stack Definida

O projeto DEVE usar exclusivamente a stack abaixo:

- **Backend**: NestJS + TypeScript, Prisma como ORM, PostgreSQL como banco de dados, Redis,
  BullMQ para filas e SSE (Server-Sent Events) para tempo real.
- **Frontend**: React + TypeScript, Vite, TanStack Query, Axios e Tailwind CSS.
- **Desktop**: Electron. O sistema é exclusivo para desktop; a interface React é entregue ao
  usuário como aplicação Electron.
- **Infraestrutura**: Docker e Docker Compose.

Essas tecnologias NÃO DEVEM ser substituídas sem alteração prévia desta constituição.

**Justificativa**: uma stack fixa mantém o código previsível, evita fragmentação de soluções
equivalentes e permite que agentes e pessoas trabalhem sobre as mesmas premissas.

### II. Liberdade de Dependências

- O agente PODE instalar as bibliotecas que facilitarem a implementação.
- DEVE ser dada preferência a pacotes ativamente mantidos e com tipagem TypeScript (nativa ou
  via `@types`).
- Cada nova dependência DEVE ser listada no `plan.md` da feature, com uma linha explicando o
  motivo da adoção.

**Justificativa**: não reinventar soluções acelera a entrega, e o registro no plano mantém
rastreável por que cada pacote existe.

### III. Arquitetura Modular e Escalável

**Backend**:

- Cada domínio DEVE ser um módulo NestJS próprio, com camadas separadas: controller, service
  e acesso a dados.
- Controllers NÃO DEVEM conter regra de negócio; limitam-se a receber a requisição, delegar
  ao service e devolver a resposta.
- Módulos DEVEM se comunicar por meio de services exportados e NUNCA DEVEM acessar
  diretamente as tabelas de outro domínio.

**Frontend**:

- O código DEVE ser organizado por feature: componentes, hooks, chamadas de API e tipos de
  cada feature ficam juntos.
- A pasta compartilhada DEVE conter apenas o que é reutilizado por várias features.

**Justificativa**: fronteiras claras entre domínios reduzem acoplamento, facilitam testes e
permitem que o sistema cresça sem que mudanças em um domínio quebrem outros.

### IV. TypeScript Estrito

- O modo `strict` DEVE estar ativo no backend, no frontend e no processo Electron.
- O uso de `any` NÃO DEVE ocorrer. Quando o tipo for desconhecido, DEVE-se usar `unknown`
  com validação (type guards ou schemas) antes do uso.
- Contratos de API — DTOs de request e response, payloads de eventos SSE e payloads de jobs —
  DEVEM ser tipados explicitamente.

**Justificativa**: tipagem estrita transforma erros de integração em erros de compilação e
documenta os contratos entre camadas.

### V. Banco de Dados com Prisma

- O `schema.prisma` é a fonte da verdade do modelo de dados.
- Toda alteração no banco DEVE ser feita via `prisma migrate`; alterações manuais NÃO DEVEM
  ocorrer.
- Operações que alteram mais de uma tabela DEVEM usar transação (`prisma.$transaction`).
- Consultas DEVEM usar `select`/`include` de forma intencional, para evitar N+1 e tráfego de
  dados desnecessário.

**Justificativa**: um único schema versionado garante ambientes reproduzíveis; transações
preservam a consistência; consultas intencionais mantêm a performance previsível.

### VI. Tempo Real com SSE

**Backend**:

- Endpoints SSE DEVEM usar o suporte nativo do NestJS (`@Sse` com Observables).
- Todo evento DEVE ter um tipo nomeado e um payload tipado.
- A publicação de eventos DEVE passar por um serviço de eventos que usa Redis pub/sub, para
  que eventos gerados em qualquer instância da API ou em qualquer worker cheguem a todos os
  clientes conectados.
- A conexão DEVE enviar heartbeat periódico.

**Frontend**:

- O cliente DEVE se reconectar automaticamente quando a conexão cair.
- Eventos SSE DEVEM atualizar o cache do TanStack Query (`setQueryData` ou
  `invalidateQueries`); NÃO DEVE haver estado paralelo dos mesmos dados.

**Justificativa**: o pub/sub via Redis torna o tempo real independente de qual processo gerou
o evento, e usar o cache do TanStack Query como única fonte evita dados divergentes na tela.

### VII. Dados no Frontend

- Todo estado vindo do servidor DEVE ser gerenciado pelo TanStack Query; NÃO DEVE haver busca
  de dados em `useEffect`.
- As chamadas HTTP DEVEM usar uma instância única e centralizada do Axios, com interceptors
  para autenticação e tratamento de erros.
- As query keys DEVEM ser definidas de forma centralizada por feature.
- A estilização DEVE usar Tailwind CSS. NÃO DEVE haver CSS inline nem arquivos CSS por
  componente, exceto em casos que o Tailwind não cubra (e a exceção DEVE ser justificada no
  código).

**Justificativa**: centralizar cache, cliente HTTP e chaves elimina inconsistências de
sincronização; um único sistema de estilos mantém a interface coesa.

### VIII. Validação e Segurança

- Toda entrada recebida pela API DEVE ser validada no backend, por meio de `ValidationPipe`
  global com DTOs.
- As variáveis de ambiente DEVEM ser validadas na inicialização da aplicação; a aplicação NÃO
  DEVE subir se alguma variável estiver faltando ou inválida.
- Segredos NUNCA DEVEM ser commitados no repositório.

**Justificativa**: validar na borda impede que dados inválidos alcancem a regra de negócio, e
falhar cedo na configuração evita erros silenciosos em produção.

### IX. Testes

- Regras de negócio dos services DEVEM ter testes unitários.
- Os fluxos principais da API DEVEM ter testes de integração (e2e).
- Processors de filas DEVEM ter testes cobrindo cenários de sucesso e de falha.
- No frontend, hooks e componentes com lógica relevante DEVEM ter testes.
- Uma feature só é considerada concluída com todos os testes passando.

**Justificativa**: testes são a evidência objetiva de que a feature funciona e a rede de
segurança que permite evoluir o código sem regressões.

### X. Qualidade e Padronização

- O código DEVE passar no lint (ESLint), no type-check (`tsc --noEmit`) e na formatação
  (Prettier) sem erros.
- Erros da API DEVEM seguir um formato de resposta padronizado em toda a aplicação, tratado
  por um filtro de exceções global.

**Justificativa**: padrões automatizados eliminam discussões de estilo, e um formato único de
erro permite que o frontend trate falhas de forma uniforme.

### XI. Processamento Assíncrono com BullMQ

- Tarefas demoradas, pesadas ou que dependem de serviços externos (envio de e-mail,
  processamento de arquivos, integrações, relatórios) DEVEM ser executadas em filas BullMQ,
  NUNCA dentro do ciclo de uma requisição HTTP.
- A integração DEVE usar o módulo oficial `@nestjs/bullmq`.
- Cada fila e seu processor DEVEM pertencer ao módulo do domínio correspondente.
- Payloads de jobs DEVEM conter apenas o necessário — preferencialmente IDs, não objetos
  completos.
- Jobs DEVEM ser idempotentes, ou seja, seguros para executar mais de uma vez.
- Jobs DEVEM ter retentativas com backoff configuradas.
- Jobs que falharem definitivamente DEVEM ser mantidos para inspeção (sem remoção automática
  ao falhar).
- Quando o resultado de um job interessar ao usuário, a conclusão DEVE publicar um evento pelo
  serviço de eventos (Princípio VI), para que chegue ao frontend via SSE.
- Os workers DEVERIAM poder rodar em um processo separado da API, para escalar de forma
  independente.

**Justificativa**: tirar trabalho pesado do ciclo HTTP mantém a API responsiva; idempotência,
retentativas e retenção de falhas tornam o processamento confiável e auditável.

### XII. Ambiente com Docker

- Todo o projeto DEVE subir com um único comando (`docker compose up`), incluindo PostgreSQL,
  Redis, backend, workers e frontend.
- Os serviços DEVEM ter healthchecks, e o backend DEVE aguardar banco e Redis estarem
  saudáveis antes de iniciar.
- Os dados do PostgreSQL e do Redis DEVEM persistir em volumes.
- Em desenvolvimento, backend e frontend DEVEM ter hot reload.
- As imagens de produção DEVEM usar Dockerfiles multi-stage, sem dependências de
  desenvolvimento.
- As migrations DEVEM ser aplicadas com `prisma migrate deploy` na subida do backend em
  produção.
- Toda configuração DEVE vir de variáveis de ambiente, e o repositório DEVE ter um
  `.env.example` listando todas as variáveis usadas pelos containers.

**Justificativa**: um ambiente reproduzível com um único comando reduz o tempo de onboarding e
elimina diferenças entre máquinas e entre desenvolvimento e produção.

### XIII. Git e GitHub

- Commits DEVEM ser curtos e diretos: uma linha de assunto no imperativo, com no máximo 72
  caracteres, descrevendo o que a mudança faz.
- Mensagens de commit DEVEM seguir o padrão Conventional Commits (`feat:`, `fix:`, `docs:`,
  `refactor:`, `test:`, `chore:` etc.).
- Cada commit DEVE ter um único propósito; mudanças não relacionadas DEVEM ir em commits
  separados.
- Commits e pull requests NÃO DEVEM conter trailers `Co-Authored-By` nem assinaturas ou
  rodapés referentes ao Claude ou a outras ferramentas de IA.
- Features DEVEM ser desenvolvidas em branches próprias e integradas à `main` via pull request.

**Justificativa**: um histórico limpo e padronizado facilita revisão, busca e geração de
changelog, e a autoria dos commits fica com quem é responsável pelo código.

## Definição de Pronto (Quality Gates)

Uma feature só PODE ser considerada concluída quando todos os itens abaixo forem verdadeiros:

1. Lint, type-check e formatação passam sem erros no backend, no frontend e no Electron
   (Princípios IV e X).
2. Todos os testes exigidos pelo Princípio IX existem e passam.
3. Alterações de banco estão em migrations do Prisma geradas via `prisma migrate`
   (Princípio V).
4. Novas dependências estão listadas no `plan.md` da feature, com o motivo de cada uma
   (Princípio II).
5. Novas variáveis de ambiente estão no `.env.example` e na validação de inicialização
   (Princípios VIII e XII).
6. Novos eventos SSE e payloads de jobs têm tipo nomeado e payload tipado (Princípios IV, VI
   e XI).
7. O projeto continua subindo com `docker compose up`, com todos os healthchecks saudáveis
   (Princípio XII).
8. Os commits da feature seguem o Princípio XIII.

## Fluxo de Desenvolvimento e Conformidade

- Toda feature segue o fluxo Spec Kit: especificação (`spec.md`) → plano (`plan.md`) →
  tarefas (`tasks.md`) → implementação.
- A seção "Constitution Check" do `plan.md` DEVE verificar a conformidade com cada princípio
  aplicável antes da pesquisa (Fase 0) e novamente após o design (Fase 1).
- Qualquer desvio de um princípio DEVE ser registrado na seção "Complexity Tracking" do
  `plan.md`, com a justificativa e a alternativa mais simples que foi rejeitada. Desvios não
  justificados bloqueiam o avanço da feature.
- O `tasks.md` DEVE incluir as tarefas de teste exigidas pelo Princípio IX.
- Revisões de código DEVEM verificar a conformidade com esta constituição.

## Governance

- Esta constituição prevalece sobre qualquer outra prática, convenção ou documento do projeto.
  Em caso de conflito, vale o que está escrito aqui.
- Alterações DEVEM ser feitas executando `/speckit.constitution` novamente, com justificativa
  explícita da mudança e atualização de versão.
- O versionamento segue SemVer:
  - **MAJOR**: remoção ou redefinição incompatível de princípios ou regras de governança
    (incluindo a troca de qualquer tecnologia do Princípio I).
  - **MINOR**: novo princípio ou seção, ou expansão material de orientações existentes.
  - **PATCH**: esclarecimentos, ajustes de redação e correções sem mudança de significado.
- Toda alteração DEVE atualizar a data de "Last Amended" e registrar o impacto nos artefatos
  dependentes (templates e features em andamento).
- A conformidade é verificada em cada `plan.md` (Constitution Check), em cada revisão de
  código e na Definição de Pronto.

**Version**: 1.1.0 | **Ratified**: 2026-10-27 | **Last Amended**: 2026-10-27
