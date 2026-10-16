# 11. Registros de Decisão Arquitetural (ADRs)

## 11.0 Visão Geral e Catálogo Resumo de ADRs

Os ADRs registram decisões arquiteturais duradouras, seu contexto, alternativas consideradas, consequências e critérios de revisão. Eles não substituem os requisitos do Checkpoint 1: materializam respostas técnicas rastreáveis aos IDs oficiais de User Stories e RNFs.

| ID do ADR | Título | Status | Decisão central | Requisitos/RNFs motivadores |
|---|---|---|---|---|
| **ADR-001** | Persistência transacional no PostgreSQL com multi-tenancy por RLS e concorrência de reservas por GiST | Aceito | PostgreSQL é a fonte de verdade; `condominium_id` e RLS isolam tenants; exclusion constraint impede reservas sobrepostas. | `US01`, `US02`, `US06`, `US07`, `RNF-CONF-01`, `RNF-CONF-02`, `RNF-PERF-02`, `RNF-ESC-01`, `RNF-ESC-02`, `RNF-SEG-04`, `RNF-SEG-05` |
| **ADR-002** | Redis para cache distribuído, sessão/revogação stateless e lock auxiliar | Aceito | Redis otimiza leitura, sessão/revogação e contingência, mas nunca é fonte única de verdade ou garantia final de reserva. | `RNF-PERF-01`, `RNF-PERF-02`, `RNF-ESC-01`, `RNF-SEG-03`, `RNF-DISP-02` |
| **ADR-003** | Mensageria assíncrona com RabbitMQ, Transactional Outbox e DLQ | Aceito | Eventos técnicos são persistidos em Outbox, publicados no RabbitMQ e consumidos por workers idempotentes com retry e DLQ. | `US01`, `US07`, `US09`, `RNF-PERF-03`, `RNF-DISP-03`, `RNF-ESC-03`, `RNF-OBS-03` |
| **ADR-004** | Monólito Modular com Arquitetura Hexagonal e Vertical Slices | Aceito | O Backend Core é implantado como monólito modular stateless; módulos usam portas/adaptadores e casos de uso coesos. | `RNF-MANT-01`, `RNF-MANT-02`, `RNF-MANT-03`, `RNF-ESC-01`, `RNF-ESC-03` |
| **ADR-005** | Estratégia criteriosa de padrões GoF guiada pela Regra dos Três | Aceito | Strategy, Factory Method, Observer, Adapter e Decorator são adotados somente diante de variação ou acoplamento recorrente demonstrado. | `US01`, `US03`, `RNF-MANT-01`, `RNF-MANT-02`, `RNF-MANT-03`, `RNF-PERF-03` |
| **ADR-006** | Idempotência em webhooks e consumidores assíncronos | Aceito | Toda entrada com entrega ao-menos-uma-vez é deduplicada por identificador estável e produz efeitos transacionais auditáveis. | `US07`, `US12`, `RNF-CONF-02`, `RNF-DISP-03`, `RNF-SEG-10`, `RNF-OBS-01` |

## 11.1 ADR-001 — Persistência Transacional no PostgreSQL com Multi-tenancy via RLS e Concorrência de Reservas via `EXCLUDE USING gist`

### Status, data e decisores

| Campo | Registro |
|---|---|
| Status | Aceito |
| Data | 2026-10-11 |
| Decisores | Arquitetura de Solução, responsáveis de Backend, Dados, Segurança e Operações |
| Escopo | Dados transacionais do SaaS, isolamento multi-tenant, reservas e históricos auditáveis |

### Contexto e problema de negócio/técnico

O SaaS centraliza processos de múltiplos condomínios no mesmo produto. `US01` exige que uma reserva seja confirmada sem duplicidade; `US02` exige que um cancelamento torne o horário disponível; `RNF-CONF-01` exige que duas requisições simultâneas não criem duas reservas confirmadas. Ao mesmo tempo, `RNF-ESC-01` e `RNF-ESC-02` requerem crescimento multi-tenant sem infraestrutura exclusiva por cliente, e `RNF-SEG-04`/`RNF-SEG-05` exigem controle estrito por perfil e unidade.

Uma filtragem de `condominium_id` apenas na aplicação é insuficiente: um erro de implementação, uma consulta administrativa mal construída ou o reuso indevido de conexão pode expor ou alterar dados de outro tenant. Também não é suficiente validar disponibilidade antes de gravar a reserva; duas transações concorrentes podem ler o mesmo período como livre.

`US07` e `RNF-CONF-02` exigem que o histórico de status de ocorrência seja preservado, impedindo edição retroativa silenciosa. A solução precisa combinar isolamento, consistência transacional e evidência de auditoria.

### Decisão técnica detalhada

1. **PostgreSQL é a fonte transacional de verdade.** Recursos multi-tenant carregam `condominium_id` obrigatório, inclusive onde a relação poderia ser inferida por `unit_id`. Essa redundância controlada torna políticas RLS, índices e diagnósticos explícitos.
2. **A API resolve o tenant de uma identidade autenticada e configura o contexto no banco por transação.** Após validar o JWT e o vínculo do usuário, o adaptador de banco executa `SET LOCAL app.current_tenant = '<tenant-id>'` dentro da transação. `SET LOCAL` limita o contexto à transação e reduz o risco de vazamento por pooling de conexões. Conexões são devolvidas ao pool somente após `COMMIT` ou `ROLLBACK`.
3. **RLS é habilitado e forçado nas tabelas multi-tenant.** As políticas comparam `condominium_id` ao valor de `current_setting('app.current_tenant', true)`. A função de aplicação usa uma role de banco sem `BYPASSRLS`; migrações e operações break-glass usam identidade separada, auditada e restrita.
4. **A reserva recebe proteção final no banco.** A extensão `btree_gist` permite combinar a igualdade de `common_area_id` com o intervalo `tstzrange(start_datetime, end_datetime, '[)')`. Uma constraint `EXCLUDE USING gist` impede sobreposição de reservas com estado bloqueante, como `Confirmed`. O intervalo semiaberto permite que uma reserva termine exatamente quando a seguinte começa.
5. **Tabelas de histórico são append-only.** `INCIDENT_STATUS_HISTORY` e `ACCESS_LOG` permitem somente inserção pela role operacional apropriada; atualizações e remoções são negadas. O estado atual de `INCIDENT` e seu histórico são gravados na mesma transação.
6. **Índices e consultas iniciam pelo tenant.** Índices compostos usam `condominium_id` seguido dos filtros reais de consulta, como unidade, área, status e data. Isso preserva seletividade e limita o impacto de cargas de um condomínio sobre outro.

### Alternativas consideradas e motivos de descarte

| Alternativa | Benefícios reconhecidos | Motivo de descarte para o estágio atual |
|---|---|---|
| Banco separado por tenant | Isolamento físico forte; restauração e tuning individual. | Custo operacional e financeiro cresce linearmente; migrações, observabilidade, backups e pooling se multiplicam; não é proporcional à volumetria inicial. |
| Schema separado por tenant | Isolamento lógico mais rígido que uma tabela compartilhada. | Multiplica DDL, migrações, conexões e metadados; dificulta consultas de administradora e automação; não elimina necessidade de autorização na aplicação. |
| NoSQL, como MongoDB ou DynamoDB | Escala horizontal e flexibilidade de documento em determinados cenários. | Não oferece exclusion constraint nativa para intervalos temporais equivalente ao GiST; aumenta complexidade para relações, relatórios financeiros e integridade transacional do domínio. |
| Validação somente em memória/aplicação | Implementação inicial aparentemente simples. | Falha sob concorrência, múltiplas réplicas e reprocessamento; não atende `RNF-CONF-01`; não oferece defesa de dados equivalente a RLS. |

### Consequências positivas

- Garante integridade de reservas na camada que arbitra a concorrência real.
- Implementa defesa em profundidade: tenant é validado por identidade, aplicação, RLS e índices de dados.
- Permite transações atômicas entre estado atual, histórico e Outbox.
- Mantém um caminho auditável de transição de ocorrências e de acessos de visitantes.
- Mantém o custo operacional inicial menor que bancos ou schemas separados por tenant.

### Consequências negativas, custos, riscos assumidos e mitigação

| Custo ou risco | Consequência possível | Estratégia de mitigação |
|---|---|---|
| RLS e `SET LOCAL` mal configurados | Vazamento entre tenants ou bloqueio indevido de consultas. | Testes de integração com dois tenants; role sem `BYPASSRLS`; helper transacional único; teste automatizado que falha se o contexto não estiver definido. |
| GiST e `btree_gist` aumentam custo de escrita/índice | Maior consumo de CPU, memória e espaço em alta taxa de reserva. | Monitorar tamanho/uso de índice; índices mínimos; testes de carga; particionamento somente após evidência de crescimento. |
| Exclusion constraint retorna erro de banco | Experiência ruim se o erro vazar como 500. | Adaptador mapeia violação de constraint para `BookingConflict`; controller devolve HTTP `409` com mensagem de negócio. |
| Tabelas append-only crescem continuamente | Aumento de storage e consultas históricas mais lentas. | Política de retenção conforme requisitos legais; índices temporais; arquivamento auditável sem apagar evidência necessária. |
| RLS pode dificultar rotinas administrativas legítimas | Relatórios globais e suporte podem ser bloqueados. | Fluxos administrativos explícitos e auditados; papel de suporte com escopo concedido; break-glass com expiração, aprovação e log. |

### Evidência de validação, critério de aceite e gatilhos para revisão futura

- **Aceite:** testes concorrentes executam N tentativas no mesmo espaço/período e persistem exatamente uma reserva confirmada; as demais retornam conflito de domínio. Testes pgTAP/integração comprovam que um tenant não lê, insere, atualiza ou exclui linhas de outro tenant.
- **Aceite:** tentativa operacional de alterar ou apagar `INCIDENT_STATUS_HISTORY` e `ACCESS_LOG` é negada; uma transição válida cria o estado atual e histórico na mesma transação.
- **Revisar:** avaliar particionamento, read model ou banco por tenant somente se métricas sustentadas indicarem saturação de I/O, índices/queries por tenant fora de SLO, ou exigência contratual/regulatória de isolamento físico.
- **Revisar:** avaliar reescrita do modelo de reservas se houver novos tipos de bloqueio temporal que não possam ser representados pelo intervalo e estado atuais, preservando garantia equivalente à constraint.

## 11.2 ADR-002 — Utilização do Redis para Cache Distribuído, Sessão/Revogação Stateless e Lock Auxiliar

### Status, data e decisores

| Campo | Registro |
|---|---|
| Status | Aceito |
| Data | 2026-10-11 |
| Decisores | Arquitetura de Solução, responsáveis de Backend, Segurança e Operações |
| Escopo | Cache, sessão, revogação, rate limiting, contingência de portaria e otimização de concorrência |

### Contexto e problema de negócio/técnico

`RNF-PERF-01` define resposta de 1–2 segundos para consultas comuns. `RNF-ESC-01` exige múltiplas réplicas stateless, tornando cache e sessão locais incoerentes. `RNF-SEG-03` exige expiração de sessão por inatividade. `RNF-DISP-02` exige degradação graciosa na consulta de autorização de visitante, sem permitir que uma falha de banco resulte em autorização inventada.

O banco transacional preserva consistência, mas leituras repetitivas de disponibilidade, configuração, sessão/revogação e autorização vigente podem impor carga desnecessária. A solução precisa acelerar tais leituras mantendo o PostgreSQL como fonte de verdade.

### Decisão técnica detalhada

1. Redis é usado para cache distribuído, revogação/controle de sessão, rate limiting e lock auxiliar de curta duração.
2. Nenhuma chave Redis é a única cópia persistente de reserva, ocorrência, visitante, documento ou financeiro. Uma resposta crítica é confirmada pelo PostgreSQL, não pelo cache.
3. Chaves são prefixadas pelo tenant: `tenant:{tenantId}:booking-availability:{areaId}:{date}:{version}`, `tenant:{tenantId}:visitor-auth:{hash}`, `session:{sessionId}` e `revocation:{tokenId}`. O prefixo é requisito de segurança e de diagnóstico, não apenas convenção.
4. TTLs são específicos à natureza do dado: revogação até a expiração do token; sessão conforme inatividade; disponibilidade por janela curta; autorização de visitante somente até o menor valor entre TTL operacional e `valid_until`.
5. Invalidação é orientada a evento: confirmação/cancelamento de reserva invalida disponibilidade; alteração de configuração incrementa versão de chave; alteração/revogação de autorização de visitante remove a entrada correspondente.
6. Redis pode fornecer lock auxiliar para reduzir contenção de reserva, mas a confirmação depende da constraint GiST no PostgreSQL. Lock expirado, perdido ou duplicado jamais pode permitir duas reservas confirmadas.
7. Em indisponibilidade de Redis, a API continua com leitura do banco para fluxos não críticos, com rate limit degradado e alertado. Para portaria, apenas cache previamente obtido e ainda válido pode ser usado como contingência; sem cache válido, o sistema exige fluxo de exceção seguro em vez de liberar acesso automaticamente.

### Alternativas consideradas e motivos de descarte

| Alternativa | Benefícios reconhecidos | Motivo de descarte |
|---|---|---|
| Cache em memória local do processo Node.js | Baixa latência e simplicidade inicial. | Quebra coerência entre réplicas, perde dados em restart, impede invalidação distribuída e viola o modelo stateless. |
| Memcached | Cache simples e eficiente. | Não cobre de maneira integrada revogação/sessão, estruturas/TTL necessários e recursos operacionais adotados; Redis reduz número de componentes. |
| Consulta direta ao banco sem cache | Sem invalidação e sem nova dependência. | Pressiona o Primary em leituras repetitivas, aumenta latência e reduz margem de disponibilidade na portaria. |

### Consequências positivas

- Escala horizontalmente sem sessão de usuário presa a uma réplica.
- Reduz pressão de leitura no PostgreSQL e melhora latência de consultas elegíveis.
- Permite cache de contingência limitado para portaria.
- Centraliza revogação e expiração sem tornar a aplicação stateful.

### Consequências negativas, custos, riscos assumidos e mitigação

| Custo ou risco | Consequência possível | Estratégia de mitigação |
|---|---|---|
| Cache desatualizado | Exibição temporária de disponibilidade ou configuração antiga. | TTL curto, invalidação por evento, versão de chave e leitura do Primary para decisões críticas. |
| Redis indisponível | Aumento de latência/carga no banco; perda de contingência de consulta. | Circuit breaker, fallback explícito ao banco, limites de carga, alertas e procedimento de portaria seguro sem cache válido. |
| Prefixo de tenant omitido | Vazamento de dados entre condomínios. | Biblioteca única de construção de chave, testes automatizados de isolamento e revisão arquitetural. |
| Lock distribuído mal interpretado | Falsa percepção de garantia de concorrência. | Documentação explícita e testes que comprovam que a GiST mantém integridade mesmo sem Redis. |
| Acúmulo de chaves ou TTL inadequado | Consumo excessivo e eviction de dados úteis. | Métricas de memória, evictions e hit ratio; limites de memória; TTL por classe de dado; revisão periódica. |

### Evidência de validação, critério de aceite e gatilhos para revisão futura

- **Aceite:** em múltiplas réplicas, revogação é reconhecida consistentemente; nenhuma sessão expirada é aceita; métricas demonstram p95 de consulta dentro do SLO com taxa de cache hit definida por endpoint.
- **Aceite:** com Redis indisponível, reserva ainda é protegida por PostgreSQL/GiST e a portaria somente usa autorização cacheada, vigente e marcada como contingência.
- **Revisar:** separar workloads de cache, sessão e rate limit se concorrência de memória, eviction persistente ou requisitos de disponibilidade distintos exigirem clusters independentes.
- **Revisar:** remover lock auxiliar se métricas mostrarem baixo benefício ou incidentes de complexidade, pois ele não é requisito de correção.

## 11.3 ADR-003 — Mensageria Assíncrona com RabbitMQ, Transactional Outbox e Dead Letter Queue (DLQ)

### Status, data e decisores

| Campo | Registro |
|---|---|
| Status | Aceito |
| Data | 2026-10-11 |
| Decisores | Arquitetura de Solução, responsáveis de Backend, Integrações e Operações |
| Escopo | Eventos técnicos, notificações, isolamento de efeitos colaterais e recuperação de falhas |

### Contexto e problema de negócio/técnico

Reservas confirmadas (`US01`), atualizações de ocorrência (`US07`) e comunicados (`US09`) precisam notificar usuários. No entanto, `RNF-PERF-03` exige que push/e-mail não bloqueiem a ação principal; `RNF-DISP-03` impede que falha de serviço não crítico indisponibilize reserva ou autorização; `RNF-ESC-03` exige escala independente de componentes de alto volume.

Persistir o estado no PostgreSQL e, em seguida, publicar diretamente uma mensagem cria uma falha clássica: se houver commit sem publicação, o evento se perde; se houver publicação sem commit, um consumidor pode observar um estado inexistente. O sistema requer entrega ao-menos-uma-vez, idempotência e operação reprocessável.

### Decisão técnica detalhada

1. O caso de uso grava a alteração de negócio e um registro em `outbox_events` na **mesma transação PostgreSQL**.
2. O evento contém `event_id`, tipo, versão de schema, `tenant_id`, `aggregate_id`, `correlation_id`, payload mínimo, data de criação, tentativas e status.
3. Um `OutboxDispatcher` busca eventos pendentes com claim/lock seguro, publica-os no RabbitMQ e marca o despacho somente após confirmação do broker.
4. RabbitMQ organiza filas por responsabilidade e criticidade. Mensagens de entrega de e-mail/push são consumidas por `Notification Worker`, separado do Backend Core.
5. Falhas transitórias recebem retry com backoff exponencial e jitter. Após limite de tentativas, a mensagem é roteada para DLQ, com alerta, contexto de erro e procedimento de inspeção/reprocessamento.
6. Consumidores são idempotentes por `event_id` e efeito de destino. Reentrega é comportamento esperado, não exceção.
7. O core responde ao usuário após o commit da operação. A entrega de notificação possui estado observável e recuperável, mas não muda retroativamente o sucesso da reserva, ocorrência ou comunicado.

### Alternativas consideradas e motivos de descarte

| Alternativa | Benefícios reconhecidos | Motivo de descarte |
|---|---|---|
| Chamada HTTP/SMTP síncrona no fluxo da requisição | Menos infraestrutura e entendimento inicial direto. | Provedor lento/falho bloqueia resposta; acopla disponibilidade do core à integração; não atende `RNF-PERF-03` e `RNF-DISP-03`. |
| Apache Kafka | Alta capacidade, retenção e replay robusto para streams extensos. | Overhead operacional, custo de plataforma e governança de tópicos não proporcionais à volumetria inicial; RabbitMQ atende filas de trabalho e retry/DLQ do caso atual. |
| Polling simples em tabela sem broker dedicado | Usa somente o banco e reduz componentes. | Mistura workload de mensageria e transacional, oferece controle de consumo/retry/DLQ menos adequado e pode pressionar o Primary. |

### Consequências positivas

- Falha de e-mail/push não impede persistência de operação crítica.
- A Outbox elimina a lacuna entre commit de banco e publicação.
- Worker pode escalar de forma independente conforme backlog.
- DLQ torna falhas visíveis, auditáveis e reprocessáveis.
- Eventos habilitam auditoria, notificações e métricas como consumidores independentes.

### Consequências negativas, custos, riscos assumidos e mitigação

| Custo ou risco | Consequência possível | Estratégia de mitigação |
|---|---|---|
| Consistência eventual | Notificação pode chegar após a resposta principal. | UX informa ação confirmada; dashboards de idade de evento; SLO de entrega e reprocessamento. |
| Duplicidade de entrega | E-mail, push ou atualização pode ocorrer mais de uma vez. | Chave de idempotência em consumidor/provedor; tabela de processados/entregas; handlers sem efeitos não idempotentes. |
| DLQ esquecida | Eventos falhos deixam de ser tratados. | Alertas por profundidade/idade da DLQ, runbook, ownership e revisão periódica. |
| Complexidade operacional do broker | Novos componentes, credenciais e monitoramento. | RabbitMQ gerenciado/clusterizado conforme ambiente; IaC; dashboards, probes e testes de recuperação. |
| Payload excessivo ou com PII | Vazamento em broker, logs ou DLQ. | Payload mínimo, referências em vez de documentos/dados desnecessários, criptografia de transporte, mascaramento e controle de acesso. |

### Evidência de validação, critério de aceite e gatilhos para revisão futura

- **Aceite:** ao desligar o provedor de e-mail/push, reserva, ocorrência e comunicado ainda retornam sucesso após commit; o evento fica pendente/retry/DLQ com correlação rastreável.
- **Aceite:** simulação de falha entre commit e publicação demonstra que o dispatcher recupera evento não enviado; reentrega não duplica efeito observável.
- **Revisar:** considerar Kafka somente quando métricas indicarem necessidade comprovada de retenção longa, replay de streams em grande volume, múltiplos consumidores independentes em escala ou throughput acima da capacidade operacional do RabbitMQ.
- **Revisar:** extrair Notification em serviço independente quando backlog, cadência de deploy ou ownership comprovarem autonomia operacional, conforme gatilhos do DAS.

## 11.4 ADR-004 — Estilo Arquitetural: Monólito Modular com Arquitetura Hexagonal (Ports & Adapters) e Vertical Slices

### Status, data e decisores

| Campo | Registro |
|---|---|
| Status | Aceito |
| Data | 2026-10-11 |
| Decisores | Arquitetura de Solução, responsáveis de Backend, Segurança, Dados e Operações |
| Escopo | Estrutura do Backend Core, fronteiras de módulo, dependências e evolução futura |

### Contexto e problema de negócio/técnico

`RNF-MANT-01` exige modularização por domínio; `RNF-MANT-02` exige padrão claro e documentado; `RNF-MANT-03` exige testes das regras críticas. `RNF-ESC-01` pede crescimento horizontal e `RNF-ESC-03` pede componentes assíncronos escaláveis independentemente.

O sistema possui nove domínios relacionados — CondominiumManagement, Booking, VisitorAccess, Incident, Communication, Document, Financial, ServiceProvider e Notification —, mas ainda não há evidência de que cada domínio exija deploy, banco, equipe e operação independentes. Uma arquitetura precisa preservar velocidade de evolução sem criar um monólito acoplado ao ORM ou uma rede de microsserviços prematura.

### Decisão técnica detalhada

1. O Backend Core é um **monólito modular stateless**, distribuído como um artefato de aplicação com múltiplas réplicas atrás de Load Balancer/Ingress.
2. Cada módulo organiza `domain`, `application`, `ports` e `adapters`. O domínio contém invariantes; aplicação orquestra casos de uso; portas descrevem dependências; adaptadores conectam HTTP, banco, cache, broker e fornecedores.
3. **Vertical Slices** organizam casos de uso completos, por exemplo `create-booking`, com DTO, política, autorização, use case, porta, adaptador e testes próximos.
4. Módulos só se comunicam pela camada de aplicação, portas ou eventos explícitos. É proibido acessar repositório, entidade ou adaptador interno de outro módulo.
5. Infraestrutura transversal — AuthGuard/RBAC, TenantResolver, ObservabilityInterceptor e AuditLogger — possui contratos claros e não incorpora regras específicas de reserva, ocorrência ou financeiro.
6. Notification já é isolado por fila e worker, preservando uma possível extração futura sem tornar a solução distribuída antes da necessidade.

### Alternativas consideradas e motivos de descarte

| Alternativa | Benefícios reconhecidos | Motivo de descarte |
|---|---|---|
| Microsserviços desde o Dia 1 | Deploy/escala/ownership potencialmente independentes. | Exigem consistência distribuída, contratos remotos, tracing, gateways, mais pipelines e maturidade operacional; custo não justificado para o estágio atual. |
| Arquitetura em camadas tradicional acoplada ao ORM | Menos código inicial e curva de entrada simples. | Regras de domínio e controllers passam a depender de persistência/framework; testes e troca de integração se tornam custosos; módulos se acoplam pelo banco/ORM. |
| Clean/Onion aplicadas de modo purista | Forte foco em dependências para dentro. | Podem criar indireção excessiva e duplicação de modelos em CRUDs simples; Hexagonal comunica melhor fronteiras de entrada/saída necessárias ao produto. |

### Consequências positivas

- Mantém transações locais simples para domínios estreitamente relacionados.
- Reduz custo de deploy e observabilidade no início, sem abandonar modularidade.
- Isola provedores externos por portas/adaptadores e facilita testes de domínio.
- Mantém casos de uso coesos e rastreáveis para `US01`–`US13`.
- Prepara Notification para escalar/extrair independentemente quando métricas justificarem.

### Consequências negativas, custos, riscos assumidos e mitigação

| Custo ou risco | Consequência possível | Estratégia de mitigação |
|---|---|---|
| Deploy único do core | Uma alteração exige build/deploy da aplicação inteira. | Pipeline rápido, testes seletivos/integração, rollout progressivo e separação prévia do worker. |
| Erosão de fronteiras de módulo | Dependências internas tornam extração futura cara. | Testes arquiteturais com dependency-cruiser/ts-arch, revisão de PR e regras de importação. |
| Portas demais em CRUD simples | Complexidade sem benefício proporcional. | Regra dos Três; porta somente onde há infraestrutura, variação ou necessidade clara de teste/substituição. |
| Escala desigual por módulo | Um domínio pesado pode consumir recursos do core. | Métricas por rota/módulo, Read Replica, cache, worker separado e gatilhos objetivos de extração. |

### Evidência de validação, critério de aceite e gatilhos para revisão futura

- **Aceite:** testes arquiteturais impedem dependência direta de módulos para repositórios/adaptadores de outros módulos; domínio não importa frameworks de HTTP, ORM ou SDKs externos.
- **Aceite:** regras de Booking e Incident são testadas sem banco; integrações são testadas por contrato/adaptador; a API mantém múltiplas réplicas sem sessão local.
- **Revisar:** extrair módulo quando houver métricas sustentadas de domínio dominante, cadence de deploy independente, requisito regulatório/segurança isolado ou ownership autônomo, e somente após contrato/eventos, dados, observabilidade e rollback estarem definidos.

## 11.5 ADR-005 — Estratégia Criteriosa de Padrões de Projeto (GoF) Guiada pela Regra dos Três

### Status, data e decisores

| Campo | Registro |
|---|---|
| Status | Aceito |
| Data | 2026-10-11 |
| Decisores | Arquitetura de Solução e responsáveis de Backend |
| Escopo | Padrões de projeto em regras variáveis, integrações e efeitos colaterais |

### Contexto e problema de negócio/técnico

`US01` e `US03` introduzem políticas de reserva configuráveis por condomínio e área. A solução integra cinco serviços externos e precisa suportar efeitos assíncronos para reservas, ocorrências e comunicados. `RNF-MANT-01`, `RNF-MANT-02` e `RNF-MANT-03` exigem modularidade, arquitetura clara e testes de regras críticas.

Sem critérios, padrões podem virar complexidade ornamental. Sem abstrações onde há variação real, condicionais, SDKs externos e chamadas laterais se espalham pelos casos de uso. A decisão precisa definir tanto onde aplicar padrões quanto onde deliberadamente não aplicá-los.

### Decisão técnica detalhada

1. **Strategy:** `BookingPolicy` encapsula pelo menos três variações reais: antecedência, limite mensal e cancelamento/elegibilidade por área/condomínio.
2. **Factory Method:** fábricas de eventos criam envelopes consistentes de reserva, ocorrência, comunicado e pagamento, com versão, tenant e correlação.
3. **Observer:** eventos de domínio, Outbox e handlers independentes notificam, auditam e atualizam métricas sem acoplamento dos agregados aos consumidores.
4. **Adapter:** portas `EmailSender`, `PushSender`, `FileStorage`, `PaymentGateway` e `SignatureProvider` isolam APIs/SDKs de fornecedor.
5. **Decorator:** timeout, retry, circuit breaker, telemetria e mascaramento decoram adaptadores externos sem poluir domínio ou casos de uso.
6. **Regra dos Três:** uma abstração é extraída quando há três variações reais, ou duas existentes e uma terceira aprovada e iminente. Exceções exigem risco concreto de segurança, consistência ou fornecedor externo que justifique isolar antes.
7. Todo padrão adotado requer: problema recorrente descrito, alternativa simples avaliada, diagrama de sequência quando altera interação relevante, testes de unidade/integração e registro de consequência.

### Alternativas consideradas e motivos de descarte

| Alternativa | Benefícios reconhecidos | Motivo de descarte |
|---|---|---|
| Não usar padrões e manter condicionais/SDKs nos serviços | Menos arquivos no curto prazo. | Cresce acoplamento, dificulta teste e torna mudança de regra/provedor arriscada após múltiplas variações. |
| Aplicar padrões a todo objeto e CRUD | Aparente uniformidade estrutural. | Overengineering, navegação difícil, indireção sem variação e menor produtividade. |
| Framework/DI como substituto de decisão de design | Automatiza wiring de dependências. | Contêiner DI não resolve fronteiras, invariantes ou critérios de abstração; pode esconder acoplamento. |

### Consequências positivas

- Reduz condicionais crescentes de política e acoplamento direto a fornecedores.
- Mantém regras críticas testáveis em isolamento.
- Habilita múltiplos observadores sem alterar agregados e casos de uso produtores.
- Cria linguagem comum para revisão de código e evolução arquitetural.

### Consequências negativas, custos, riscos assumidos e mitigação

| Custo ou risco | Consequência possível | Estratégia de mitigação |
|---|---|---|
| Mais contratos e classes | Curva de leitura e navegação maior. | Convenções de diretório/nome, documentação de nível 4 e abstrações pequenas por caso de uso. |
| Abstração prematura | Código genérico para uma necessidade inexistente. | Regra dos Três, ADR obrigatório para exceção e remoção de abstrações sem uso comprovado. |
| Decorators empilhados | Ordem de retry, telemetria e circuito pode gerar comportamento inesperado. | Ordem padronizada, testes de composição e métricas de cada camada. |
| Observer assíncrono | Consistência eventual e duplicidade. | Outbox, idempotência, DLQ, correlação e SLO de processamento. |

### Evidência de validação, critério de aceite e gatilhos para revisão futura

- **Aceite:** cada padrão possui teste e exemplo de pelo menos três variações/consumidores/integrações quando a Regra dos Três é a justificativa; os diagramas de sequência do DAS mostram o efeito arquitetural de Strategy, Factory, Observer, Adapter e Decorator.
- **Aceite:** testes arquiteturais confirmam que casos de uso dependem de portas e não de SDKs externos; políticas podem ser executadas sem infraestrutura.
- **Revisar:** simplificar/remover padrão se desaparecer a variação que o justifica; introduzir novo padrão somente com smell documentado, teste de caracterização, alternativa simples avaliada e impacto mensurável.

## 11.6 ADR-006 — Idempotência em Webhooks e Consumidores Assíncronos

### Status, data e decisores

| Campo | Registro |
|---|---|
| Status | Aceito |
| Data | 2026-10-11 |
| Decisores | Arquitetura de Solução, responsáveis de Backend, Segurança e Integrações |
| Escopo | Webhooks de pagamento/assinatura e consumidores de eventos de broker |

### Contexto e problema de negócio/técnico

`US12` depende de confirmação de pagamento por gateway. Serviços externos e brokers normalmente entregam ao-menos-uma-vez: um webhook pode ser reenviado após timeout, uma mensagem pode ser redeliverada depois de falha de worker e a resposta anterior pode ter sido perdida pelo remetente. Sem idempotência, um mesmo evento pode alterar repetidamente um lançamento financeiro, disparar várias notificações ou distorcer a auditoria.

`RNF-CONF-02` exige auditabilidade; `RNF-DISP-03` exige isolamento de falhas; `RNF-SEG-10` exige proteção de credenciais e integrações; `RNF-OBS-01` exige registros estruturados. A solução precisa tratar duplicidade como fluxo normal e validar a fronteira externa antes de produzir efeito interno.

### Decisão técnica detalhada

1. Todo webhook valida assinatura HMAC ou assimétrica, emissor, timestamp e janela contra replay antes de o payload ser tratado como confiável.
2. Cada evento externo usa `provider_event_id` estável; a tabela `processed_webhooks` possui unicidade por provedor/evento. O insert de deduplicação e a alteração do agregado ocorrem na mesma transação.
3. Cada consumidor de broker usa `event_id` e registra efeito/entrega por consumidor e destino quando necessário. Uma reentrega retorna sucesso após reconhecer que o efeito já existe.
4. Eventos de saída possuem `event_id`, versão, `tenant_id` e `correlation_id`; payload não carrega dados pessoais além do necessário.
5. Processamento inválido, não verificável ou com falha persistente é registrado para investigação, sem aceitar assinatura inválida como “sucesso”.

### Alternativas consideradas e motivos de descarte

| Alternativa | Benefícios reconhecidos | Motivo de descarte |
|---|---|---|
| Confiar que o provedor entrega uma vez | Menor armazenamento e código. | Não corresponde à semântica real de webhooks/brokers; falhas de rede causam reentrega legítima. |
| Deduplicar apenas em memória | Baixa latência. | Perde estado em restart e diverge entre réplicas; não suporta recuperação confiável. |
| Aceitar todo webhook e reconciliar manualmente depois | Implementação inicial curta. | Risco financeiro e operacional inaceitável; correção manual não garante consistência nem auditoria. |

### Consequências positivas

- Evita pagamento duplicado, notificações duplicadas e métricas inconsistentes.
- Permite retries e reentregas seguros.
- Mantém trilha auditável de recebimento e processamento.
- Protege a fronteira externa contra requisições forjadas e replay.

### Consequências negativas, custos, riscos assumidos e mitigação

| Custo ou risco | Consequência possível | Estratégia de mitigação |
|---|---|---|
| Tabelas de deduplicação crescem | Maior custo de storage e índice. | Retenção compatível com janela de reentrega/obrigações de auditoria; particionamento/arquivamento se necessário. |
| Identificador de provedor inadequado | Eventos distintos podem ser tratados como iguais ou vice-versa. | Contrato de integração documentado; chave composta por provedor/ambiente/evento; testes de contrato. |
| Falha após efeito externo e antes de registro | Reentrega pode gerar duplicidade. | Idempotency key propagada ao provedor quando suportada; transação local para efeito interno; reconciliação periódica. |

### Evidência de validação, critério de aceite e gatilhos para revisão futura

- **Aceite:** o mesmo webhook de pagamento é enviado repetidamente e altera o lançamento uma única vez; a mesma mensagem é redeliverada e não reenvia notificação ao mesmo destino.
- **Aceite:** assinatura inválida, timestamp expirado e replay fora da janela são rejeitados e registrados sem mudar dados financeiros.
- **Revisar:** alterar a estratégia se fornecedor não oferecer identificador estável ou se volume/retenção exigir armazenamento de deduplicação especializado; a nova solução deve manter evidência equivalente de idempotência.

## 11.7 Refatoração Dirigida por Padrões — Passo a Passo Prático

Esta seção operacionaliza a estratégia do ADR-005. O objetivo não é introduzir padrões por preferência estética, mas deixar que eles emerjam de código existente e de comportamento caracterizado.

1. **Identificar o code smell concreto.** Exemplos: `BookingService` com condicionais por condomínio, `FinancialService` acoplado ao SDK de pagamento ou `IncidentService` chamando auditoria, push e métrica em sequência.
2. **Criar testes de caracterização.** Antes de alterar estrutura, registrar resultados atuais válidos, entradas limites, erros e efeitos externos esperados. O teste protege comportamento, não a implementação acoplada.
3. **Executar extrações pequenas com a suíte verde.** Extrair interface/porta ou política, criar adaptador/fake, migrar um ramo por vez, manter contrato HTTP/evento e rodar testes após cada passo.
4. **Nomear o padrão somente quando ele emergir.** Se as abstrações revelarem seleção de algoritmo, é Strategy; se isolarem fornecedor, é Adapter; se desacoplarem múltiplas reações, é Observer.
5. **Registrar e validar.** Atualizar ADR, diagrama e convenções; adicionar teste arquitetural que impede retorno do acoplamento; medir ganho definido antes da refatoração.

| Code smell identificado | Teste de caracterização | Passos de refatoração | Padrão emergente | Ganho mensurável |
|---|---|---|---|---|
| `BookingService.create()` contém `if/else` por antecedência, limite mensal e política de cancelamento de cada condomínio. | Testes parametrizados cobrem três configurações reais, período inválido, limite excedido e cancelamento fora da janela. | Extrair cálculo de regra para funções puras; introduzir `BookingPolicy`; migrar uma política por vez; criar `BookingPolicyFactory`; remover condicionais antigos. | Strategy + Factory Method | Número de ramos condicionais no caso de uso diminui; nova política é adicionada com teste sem alterar fluxo transacional. |
| `FinancialService` chama SDK de pagamento, valida assinatura, atualiza lançamento e formata erro de fornecedor no mesmo método. | Testes de webhook válido, assinatura inválida, timeout, evento duplicado e lançamento já pago. | Introduzir porta `PaymentGateway`; mover SDK para `PaymentGatewayAdapter`; extrair `WebhookVerifier`; adicionar `processed_webhooks`; conservar contrato do controller. | Adapter + Decorator | Caso de uso testa com fake; troca de fornecedor não modifica domínio; testes de idempotência repetem evento sem alterar saldo. |
| `IncidentService` chama auditoria, e-mail, push e contador KPI após toda mudança de status. | Teste confirma transição e histórico mesmo com e-mail indisponível; testes verificam que cada reação recebe evento uma vez por mudança. | Primeiro publicar evento em memória no teste; separar handlers; persistir Outbox junto à transição; mover handlers para worker; implementar retry/DLQ. | Observer + Transactional Outbox | Transição não bloqueia por provedor; novos consumidores são adicionados sem mudar `IncidentService`; medição de idade/backlog torna falhas visíveis. |

### Critérios de saída de uma refatoração dirigida por padrões

- Todos os testes de caracterização e regressão permanecem verdes.
- O novo contrato possui teste unitário e, quando toca infraestrutura, teste de integração ou contrato.
- O comportamento de segurança, tenant, auditoria e idempotência não foi reduzido.
- Um teste arquitetural impede que o módulo volte a importar SDK/adaptador que a porta deveria encapsular.
- A motivação, alternativa simples e trade-off foram adicionados ou vinculados ao ADR aplicável.
- O ganho mensurável definido na tabela é observado ou, se não for, a abstração é reavaliada.
