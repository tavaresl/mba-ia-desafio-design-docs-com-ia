# Da Reunião ao Documento: pacote de design docs do Sistema de Webhooks de Notificação de Pedidos

> O enunciado original do desafio está em [`ENUNCIADO.md`](ENUNCIADO.md).

## Sobre o desafio

A partir da transcrição de uma reunião técnica de cerca de 55 minutos ([`TRANSCRICAO.md`](TRANSCRICAO.md)) e do código de um Order Management System (OMS) em Node.js, TypeScript, Prisma e MySQL, a tarefa é produzir a documentação técnica de uma feature que ainda não existe no código: webhooks que notificam clientes B2B quando o status de um pedido muda. O pacote tem PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade, e deve ser acionável o suficiente para o time começar a implementar.

A parte difícil não é escrever, é filtrar e rastrear. Nem tudo que foi dito na reunião vira requisito: houve itens descartados (exactly-once, Redis, dashboard) e adiados (e-mail de alerta, rate limiting). E nada pode ser inventado sem origem identificável na transcrição ou no código. Não alterei `src/`, `prisma/`, `tests/` nem configurações: a entrega é só documental.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (modelo Claude Sonnet 5.5) | Ferramenta principal: leu o código e a transcrição, extraiu e classificou os pontos, planejou, escreveu os documentos e rodou as verificações de consistência. |
| **Plugin `project-analizer`** (marketplace `fullcycle-claude-marketplace`, comando `/project-analizer:generate-architectural-report`) | Orquestrou três agentes: `dependency-auditor` (auditoria de dependências), `architectural-analyzer` (análise de arquitetura, com 23 componentes) e `component-deep-analyzer` (uma análise por componente). Os relatórios ficam em `docs/agents/`. |
| **Subagentes do Claude Code** | Executaram em paralelo as análises do plugin (até 20 simultâneos). |

## Workflow adotado

A ordem seguiu a sugestão do enunciado (decisões primeiro, PRD por último entre os documentos grandes), com duas etapas de preparação antes:

1. **Extração dirigida da transcrição → [`docs/SUMMARY.md`](docs/SUMMARY.md).** Cada fala foi classificada em oito categorias com identificadores (`REQ`, `DEC`, `PTD`, `RES`, `SCL`, `STD`, `POS`, `DIS`), com o falante, o timestamp e um link para a linha. As categorias `POS` (adiado) e `DIS` (descartado) existem para impedir que o que ficou de fora vire requisito. O arquivo termina com observações sobre lacunas que a reunião deixou.
2. **Mapeamento do código → [`docs/agents/`](docs/agents/).** O comando do plugin gerou a auditoria de dependências, o relatório de arquitetura e 23 análises de componentes, mais um `MANIFEST.md`, um resumo (`PROJECT-OVERVIEW`) e um índice. Serviram para ancorar os documentos em caminhos de arquivo reais.
3. **Geração dos templates**: A partir de exemplos utilizados no curso e outros utilizados em projetos externos, utilizando o Google Gemini, extraí um esqueleto (template) para cada tipo de documento a ser gerado.
4. **Plano.** Antes de escrever, li o enunciado e os templates em [`docs/templates/`](docs/templates/) e montei um plano de geração com a ordem e as dependências entre documentos.
5. **ADRs → RFC → FDD → PRD → Tracker → README.** Cada documento consumiu os anteriores: o RFC linka os ADRs, o FDD detalha o que o RFC propõe, o PRD consolida, e o Tracker varre os documentos prontos.
6. **Centralização das referências no Tracker.** Numa segunda passada, simplifiquei os documentos: as citações de origem (timestamps e links para a transcrição) saíram do texto do PRD, do RFC, do FDD e dos ADRs, e cada item relevante passou a ter um identificador (`PRD-FR-01`, `RFC-ALT-02`, `FDD-INT-03`, `ADR-004-D1`...). O [`docs/TRACKER.md`](docs/TRACKER.md) liga cada identificador à fala da reunião (`[hh:mm] Nome` e link para a linha) ou ao arquivo do código.
7. **Verificação automatizada** (script somente de leitura): conferência de que todos os IDs dos documentos têm linha no Tracker (e vice-versa), de que cada `[hh:mm] Nome` do Tracker existe na transcrição com a linha certa, de que os caminhos de arquivo existem, além de seções obrigatórias, links internos e contagens.

Convenção de honestidade nos documentos: o que não veio da reunião está marcado como **(derivação técnica)** ou aparece como *lacuna identificada na análise*, e não como decisão. No Tracker, esses itens têm o tipo "Derivação técnica" ou "Questão em aberto (lacuna)".

## Prompts customizados

**1. Extração dirigida da transcrição** (gerou `docs/SUMMARY.md`):

```text
Analise o conteudo do arquivo @TRANSCRICAO.md. Identifique, dentre o que foi discutido,
os seguintes pontos:

- requisitos funcionais levantados (`REQ`);
- as decisoes tomadas (`DEC`);
- as definicoes tecnicas principais (`PTD`);
- as restricoes (`RES`);
- ganchos para o codigo ja pre-existente (`SCL`);
- detalhes tecnicos secundarios (`STD`);
- pontos adiados para fases futuras (`POS`); e
- pontos descartados (`DIS`).

De um identificador para cada um dos pontos identificados, seguindo o formato
`PREFIXO` + `-` + `NUMERO` (incremental POR prefixo).

Organize os pontos em tabelas, com uma coluna para o identificador, uma coluna para uma
descricao do ponto, uma coluna com o timestamp da fala e o nome de quem eh a fala, e uma
coluna com um link para a linha da fala no arquivo.

Salve o resultado em `docs/SUMMARY.md`, em secoes para cada tipo de ponto, com uma breve
descricao do que sao aqueles pontos. Observacoes sobre outros pontos que nao se encaixem
nas categorias, ou sobre os pontos e categorias, vem no final do arquivo, sempre linkadas
a fonte de onde a informacao foi extraida.
```

**2. Mapeamento do código e análise de componentes:**

```text
/project-analizer:generate-architectural-report
```

O comando executa seis fases: cria `MANIFEST.md`; roda em paralelo a auditoria de dependências e a análise de arquitetura; roda em paralelo uma análise profunda por componente identificado; valida os arquivos gerados; e escreve o `PROJECT-OVERVIEW` e o `README` de índice. Usei os parâmetros padrão (projeto inteiro, saída em `docs/agents`).

**3. Geração de um plano para a geracao dos documentos** (reconstituição resumida dos pedidos desta etapa, não o texto literal):

```text
Com base no docs/SUMMARY.md, na TRANSCRICAO.md, nos relatorios do architectural-analyzer
e do component-deep-analyzer, elabore um plano para gerar os documentos do ENUNCIADO.md,
em ordem logica considerando a dependencia entre eles e seguindo os templates em
docs/templates/.
```

**4: Geracao dos documentos com base no plano, seguindo a ordem sugerida**;

**5. Centralização das referências no Tracker** (reconstituição resumida do pedido da segunda passada, não o texto literal):

```text
Simplifique um pouco os documentos gerados: deixe as referencias para a origem dos pontos
centralizadas no TRACKER, conectando identificadores (introduzidos nos documentos concretos
[PRD, FDD, RFC e ADRs]) com a origem, mantendo os formatos atuais para referenciar linhas
da transcricao e o codigo fonte.
```

## Iterações e ajustes

O enunciado espera de 3 a 5 ciclos de geração e revisão. Aqui houve uma passada principal de geração por documento, seguida de uma passada de verificação automatizada e de correções pontuais. Ajustes concretos:

- **Instalação do plugin:** a primeira tentativa usou o nome de marketplace `devfullcycle` e falhou (`Marketplace "devfullcycle" not found`). O nome correto é o que o `marketplace add` registrou: `fullcycle-claude-marketplace`.
- **Limite de subagentes:** o comando de análise tentou disparar 23 agentes de componente em paralelo, mas o limite é de 20 simultâneos. Três análises (Seed Script, Test Suite e MySQL Container) só rodaram depois que outras terminaram, e o `MANIFEST.md` foi atualizado à mão durante a espera.
- **Verificação de links com falha de método:** meu primeiro script de checagem de links tratou os espaços dos nomes de arquivo (`... 2026-10-29 14:55:48.md`) como separadores e reportou dezenas de arquivos "faltando". Refiz a checagem lendo linha a linha, e os 51 links passaram.
- **Links do SUMMARY:** os links foram escritos como `../TRANSCRICAO.md#L<n>`, e não como `TRANSCRICAO.md#L<n>` do exemplo, porque o arquivo está em `docs/`.
- **Templates com defeitos:** o template de FDD tinha um bloco de código solto e uma frase de chat no final, além de não ter as seções "Matriz de erros `WEBHOOK_*`" e "Integração com o sistema existente", exigidas pelo enunciado. Corrigi isso na geração do FDD, sem tocar nos arquivos de template.
- **Datas:** a data do template de RFC chegou a ser alterada. A orientação passou a ser não mexer nos templates, então a edição foi revertida e usei a data de hoje (2026-10-29) apenas nos documentos gerados.
- **Lacunas que a IA não deveria preencher sozinha:** em vez de inventar decisões, registrei como premissas ou questões em aberto: a contagem das 5 tentativas, a secret usada durante a rotação de 24h, onde guardar o histórico de entregas, o armazenamento da secret e a recuperação de eventos presos em `processando`. Uma dessas lacunas surgiu na releitura: o retry pode inverter a ordem de eventos do mesmo pedido (RFC Q-09 e FDD seção 14).
- **Decisões sobre as lacunas:** resolvi as lacunas em 4 rodadas de perguntas, com opções ordenadas da mais para a menos recomendada, e cheguei a aplicar as decisões nos documentos. A aplicação foi revertida a pedido, e as decisões ficaram registradas na seção "Caminhos sugeridos para as lacunas", abaixo, sem alterar os documentos de design.
- **Referências espalhadas pelo texto (segunda passada de refatoração):** na primeira versão, cada frase do PRD, do RFC, do FDD e dos ADRs carregava citações da transcrição (`[09:17] Diego, [L108](...)`), o que poluía a leitura e duplicava o que o Tracker já fazia. A segunda passada moveu toda a origem para o [`docs/TRACKER.md`](docs/TRACKER.md): um script atribuiu IDs sequenciais por seção (`PRD-FR-01`, `RFC-Q-03`, `FDD-CONTRATO-02`, `ADR-003-D1`...), extraiu as citações de cada item para montar as linhas do Tracker e as removeu do texto. O Tracker passou de 222 para 326 linhas, e os formatos foram mantidos: `[hh:mm] Nome` com o link `[Ln](../TRANSCRICAO.md#Ln)` para a transcrição, e o caminho do arquivo para o código. Os requisitos do PRD deixaram de se chamar `RF-001` e passaram a `PRD-FR-01`, e as premissas do FDD, de `P-01` para `FDD-PRM-01`. O script deixou frases órfãs (por exemplo, "Requisito:." nos endpoints do FDD e um ponto duplo em um ADR), que corrigi à mão, além de uma coluna "Origem" que precisou ser removida das tabelas do FDD. Itens sem origem identificável não receberam ID. Ao final, conferi que todos os IDs dos documentos têm linha no Tracker e vice-versa, e que nenhum documento ficou com citação de timestamp.
- **Erro de formatação:** uma linha em branco inserida no meio da tabela de questões em aberto do RFC quebrava a tabela. Foi encontrada ao conferir o arquivo e corrigida.
- **Erros de texto:** typos no PRD e no Tracker ("pedintes", "o quem fez") apareceram na releitura e foram corrigidos.

Resultado da verificação automatizada ao final: 326 identificadores nos documentos e 326 linhas no Tracker, sem nenhum de um lado só. Dessas linhas, 286 (88%) têm fonte `TRANSCRICAO`, todas com `[hh:mm] Nome` e linha existentes na transcrição, e 40 têm fonte `CODIGO`, todas com caminho de arquivo real. Os documentos não têm mais nenhuma citação de timestamp ou link de linha no texto. Há 13 requisitos funcionais no PRD e 8 ADRs com todas as seções. Os únicos caminhos `src/` citados que não existem no repositório são os que a feature vai criar (`src/worker.ts`, `src/modules/webhooks/*`), marcados como "a criar".

## Caminhos sugeridos para as lacunas

Os documentos de design mantêm como premissas ou questões em aberto os pontos que a reunião não decidiu (`RFC-Q-06` a `RFC-Q-09` e `FDD-PRM-01` a `FDD-PRM-07`, além de alguns detalhes marcados como derivação técnica). Para cada um, levantei opções, ordenei da mais para a menos recomendada e escolhi o caminho abaixo. **Esses caminhos não foram aplicados nos documentos**: são o ponto de partida sugerido para quando o time confirmar as decisões, e a última coluna diz o que mudaria em cada documento se forem adotados.

| # | Lacuna (onde aparece) | Caminho sugerido | Outras opções consideradas | Se adotado, mudaria |
| --- | --- | --- | --- | --- |
| 1 | Contagem das "5 tentativas" (`RFC-Q-06`, `FDD-PRM-01`, `ADR-003`) | Envio inicial + 5 retries (6 envios no total), nos intervalos 1m/5m/30m/2h/12h; a DLQ recebe o evento na 6ª falha. É a única leitura que bate com a janela de ~15h dita por Diego | 5 envios no total, só com os 4 primeiros intervalos (janela de ~2h36, contradiz a reunião) | `ADR-003`, `FDD` (fluxos C e D, parâmetros), `PRD-FR-09` e `PRD-OBJ-04` |
| 2 | Retry pode inverter a ordem de eventos do mesmo pedido (`RFC-Q-09`, `FDD-RSK-02`) | Aceitar e documentar; o cliente reordena com `timestamp`, `from_status` e `to_status`, que já estão no payload | Aceitar e acrescentar um número de sequência por pedido no payload; bloquear os eventos seguintes do pedido (quebraria o requisito de menos de 10s) | `ADR-002`, `RFC` (riscos), `FDD` (risco), `PRD-RSK-05` |
| 3 | Payload acima de 64KB (`FDD-PRM-04`, `FDD-ERR-04`) | O worker verifica na hora do envio, não envia, não faz retry e move para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`; a mudança de status do pedido nunca falha por isso | Rollback da mudança de status (segue "erra" ao pé da letra); descartar com log (perde o evento sem rastro) | `FDD` (fluxo B, matriz de erros: o código vira interno do worker), `ADR-003`, `PRD-FR-12` |
| 4 | Eventos presos em `PROCESSING` se o worker cair (`FDD-PRM-06`, `FDD-RSK-04`) | Ao iniciar, o worker devolve `PROCESSING` para `PENDING`; é seguro porque há um único worker, e o reenvio é coberto pelo at-least-once | Lease com `locked_until`; não tratar nesta fase | `ADR-002`, `ADR-001` (consequências), `FDD` (resiliência, risco) |
| 5 | Qual secret assina durante o grace de 24h (`RFC-Q-07`, `FDD-PRM-02`) | `X-Signature` com duas assinaturas separadas por vírgula (nova e antiga); depois das 24h, uma só | Header extra `X-Signature-Previous`; assinar só com a antiga até o fim do grace (quem já migrou falharia) | `ADR-004`, `FDD` (fluxo E, headers, exemplo de verificação), `PRD-FR-06` e `PRD-FR-07` |
| 6 | Como armazenar a secret (`RFC-Q-07`, `FDD-PRM-05`) | Cifrada em repouso com AES-256-GCM (`node:crypto`), com a chave em variável de ambiente validada em `src/config/env.ts` | Texto claro no banco (um dump expõe todas as secrets) | `ADR-004`, `FDD` (modelo de dados, parâmetros, integração com `env.ts`, novo risco da chave de cifra), `RFC` (impacto), `PRD-DEP-03` |
| 7 | O replay muda o `event_id`? (`FDD-PRM-07`) | Mantém o `event_id` e o payload originais e zera as tentativas, para o cliente deduplicar normalmente | Gerar um novo `event_id` (o cliente processaria o mesmo conteúdo duas vezes) | `ADR-003`, `ADR-005`, `FDD` (fluxo D, contrato do replay), `PRD-FR-10` |
| 8 | Onde guardar o histórico de entregas (`RFC-Q-08`, `FDD-PRM-03`) | Tabela `webhook_deliveries`, uma linha por tentativa, com resposta truncada (ex.: 4KB) | Colunas só da última tentativa na outbox; derivar de outbox + DLQ (não tem `response` nem tempo) | `FDD` (modelo de dados, contrato de deliveries), `RFC` (impacto) |
| 9 | Autorização por customer: o CRUD aceita qualquer role e o `customerId` vem do body (`FDD-RSK-07`, `ADR-008`) | Manter como decidido, como risco aceito, e endurecer na próxima fase; sem mudar o schema de `users` | Vincular usuário a customer (muda `User` e o JWT); restringir o CRUD a `ADMIN` | `ADR-008` (consequências), `FDD` (risco) |
| 10 | Gatilho de `WEBHOOK_SECRET_REQUIRED` (`FDD-ERR-03`) | Erro interno do worker quando a secret não existe ou não pode ser decifrada; DLQ direta, sem retry | Retirar o código da matriz; devolver 400 se o cliente enviar `secret` | `FDD` (matriz de erros) |
| 11 | Eventos pendentes de um webhook removido ou inativo (`FDD-FLX-04`) | Vão para a DLQ com `WEBHOOK_INACTIVE`, sem retry, permitindo replay se o webhook voltar | Descartar os pendentes; continuar enviando os já enfileirados | `FDD` (fluxo D, matriz de erros), `ADR-003` |
| 12 | Caminho da rotação de secret (`FDD-CONTRATO-05`) | `POST /webhooks/:id/secret/rotate` (o que já está no FDD) | `POST /webhooks/:id/rotate-secret`; `PATCH` com um flag | Nenhuma mudança |
| 13 | Semântica do `DELETE /webhooks/:id` (consequência do item 11) | Remoção física, com as tabelas de eventos guardando `webhook_id` sem chave estrangeira | Remoção lógica (`active=false`) | `FDD` (contrato do `DELETE`, modelo de dados), `ADR-001` (consequências) |
| 14 | Critério de sucesso do envio (`FDD-FLX-02`) | Qualquer resposta 2xx; redirects não são seguidos, para não enviar o payload assinado a outro host | Somente 200 | `FDD` (fluxo B), `ADR-003` |

Pontos a validar com o time antes de adotar: a chave de cifra do item 6 cria um ponto novo para a revisão de segurança da Sofia e uma nova variável de ambiente; e o item 13 deixa a integridade referencial das tabelas de eventos por conta do código.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md): por que e o quê (problema, escopo, requisitos funcionais, métricas, riscos).
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica de alto nível, alternativas descartadas e questões em aberto, submetida a revisão.
3. [`docs/adrs/`](docs/adrs/README.md): cada decisão fechada, com contexto, alternativas e consequências:
   - [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md) outbox no MySQL
   - [ADR-002](docs/adrs/ADR-002-worker-separado-polling.md) worker separado em polling
   - [ADR-003](docs/adrs/ADR-003-retry-backoff-e-dlq.md) retry com backoff e DLQ
   - [ADR-004](docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) HMAC-SHA256 com secret por endpoint
   - [ADR-005](docs/adrs/ADR-005-at-least-once-x-event-id.md) at-least-once com `X-Event-Id`
   - [ADR-006](docs/adrs/ADR-006-reuso-padroes-do-projeto.md) reuso dos padrões do projeto
   - [ADR-007](docs/adrs/ADR-007-publicacao-na-transacao-do-changestatus.md) publicação dentro da transação do `changeStatus`
   - [ADR-008](docs/adrs/ADR-008-autorizacao-crud-e-replay-admin.md) autorização do CRUD e do replay
4. [`docs/FDD.md`](docs/FDD.md): como construir, em detalhe (fluxos, contratos HTTP, matriz de erros `WEBHOOK_*`, observabilidade e a seção "Integração com o sistema existente").
5. [`docs/TRACKER.md`](docs/TRACKER.md): de onde veio cada item. Os documentos anteriores só carregam os identificadores; a fala da reunião (`[hh:mm] Nome` e link para a linha) ou o arquivo do código de cada um fica aqui.

Material de apoio ao processo:
- [`docs/SUMMARY.md`](docs/SUMMARY.md): extração classificada da transcrição, com as lacunas observadas.
- [`docs/agents/`](docs/agents/): relatórios de arquitetura, dependências e componentes, com `MANIFEST.md`.
- [`docs/templates/`](docs/templates/): templates usados na geração.
- [`TRANSCRICAO.md`](TRANSCRICAO.md): transcrição original, sem alterações.
