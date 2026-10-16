# 24. Checklists de Arquitetura e Padrões

Os checklists operacionais completos estão nos Anexos J, K e L. Nenhum item crítico pode ser marcado como concluído sem evidência objetiva: link de pipeline, teste, dashboard, relatório de restauração ou aprovação formal de risco.

| Controle de arquitetura | Evidência exigida | Referência |
|---|---|---|
| Isolamento de tenant | Teste RLS com dois tenants e `SET LOCAL app.current_tenant` | ADR-001, RNF-ESC-02, RNF-SEG-05 |
| Reserva concorrente | Teste de N requisições: uma confirmação e demais HTTP `409` | US01, RNF-CONF-01 |
| Modularidade | Teste de dependência impedindo acesso entre adaptadores internos de módulos | ADR-004, RNF-MANT-01/02 |
| Efeito assíncrono | Outbox, retry e DLQ exercitados com consumidor idempotente | ADR-003/006, RNF-DISP-03 |
| Uso de padrão | Smell, alternativa simples, teste e custo registrados | ADR-005 |

---

# Anexo A — Catálogo de APIs REST

| Método e rota | Operação | Papéis autorizados | Request resumido | Response resumido | Erros relevantes |
|---|---|---|---|---|---|
| `POST /v1/bookings` | Criar reserva | Morador | `commonAreaId`, `startAt`, `endAt` | `201 { id, status, period }` | `409 BOOKING_CONFLICT`, `422 BOOKING_POLICY_VIOLATION` |
| `DELETE /v1/bookings/{id}` | Cancelar reserva | Morador proprietário, Síndico | — | `204` | `403`, `422 CANCELLATION_WINDOW_EXPIRED` |
| `POST /v1/visitor-authorizations` | Pré-autorizar visitante | Morador | `visitorName`, `unitId`, `validFrom`, `validUntil` | `201 { id, validUntil }` | `403`, `422 INVALID_VALIDITY_PERIOD` |
| `POST /v1/access-logs/check-in` | Registrar entrada | Portaria | `visitorName`, `unitId`, `authorizationId?` | `201 { id, checkedInAt, source }` | `403`, `409 AUTHORIZATION_NOT_VALID` |
| `PATCH /v1/incidents/{id}/status` | Atualizar ocorrência | Síndico, Prestador atribuído | `status`, `version` | `200 { id, status, version }` | `409 VERSION_CONFLICT`, `422 INVALID_STATUS_TRANSITION` |
| `POST /v1/announcements` | Publicar comunicado | Síndico | `title`, `content`, `targetScope` | `201 { id, audienceEstimate }` | `403`, `422 INVALID_TARGET_SCOPE` |
| `POST /v1/documents/{id}/download-url` | Emitir URL assinada | Conforme `access_level` | — | `200 { url, expiresAt }` | `403 DOCUMENT_ACCESS_DENIED` |
| `GET /v1/financial/units/{unitId}/status` | Consultar taxa da unidade | Morador proprietário, Síndico | — | `200 { referencePeriod, paymentStatus }` | `403` |
| `POST /v1/webhooks/payment` | Receber webhook | Gateway autenticado | Cabeçalhos assinados + payload | `200 { accepted: true }` | `401 INVALID_SIGNATURE`, `422 INVALID_EVENT` |

Todos os endpoints usam JSON, `Authorization: Bearer`, `X-Correlation-Id` opcional/gerado e erros no formato:

```json
{
  "code": "BOOKING_CONFLICT",
  "message": "O horário solicitado não está disponível.",
  "correlationId": "cor-2f1d"
}
```

# Anexo B — Catálogo de Eventos

| Evento / versão | Produtor | Consumidores | Campos mínimos | Idempotência |
|---|---|---|---|---|
| `booking.confirmed.v1` | Booking | Notification, Audit, Metrics | `eventId`, `tenantId`, `bookingId`, `occurredAt`, `correlationId` | `eventId` + destinatário/canal |
| `booking.cancelled.v1` | Booking | Notification, Cache, Audit | `eventId`, `tenantId`, `bookingId` | `eventId` |
| `incident.status-changed.v1` | Incident | Notification, Audit, Metrics | `eventId`, `incidentId`, `previousStatus`, `newStatus` | `eventId` + handler |
| `communication.announcement-published.v1` | Communication | Notification, Audit | `eventId`, `announcementId`, `targetScope` | `eventId` |
| `visitor.checked-in.v1` | VisitorAccess | Audit, Metrics | `eventId`, `accessLogId`, `authorizationSource` | `eventId` |
| `financial.payment-confirmed.v1` | Financial | Notification, Audit, Metrics | `eventId`, `financialEntryId`, `providerEventId` | `providerEventId` + `eventId` |
| `document.signed.v1` | Document | Notification, Audit, Cache | `eventId`, `documentId`, `status` | `eventId` |

Envelope padrão: `{ eventId, type, version, occurredAt, tenantId, aggregateId, correlationId, payload }`. Eventos são gravados em `outbox_events` antes de publicação pelo relay.

# Anexo C — Matriz de Rastreabilidade

| Origem | RAS / decisão | Implementação / evidência |
|---|---|---|
| US01, RNF-CONF-01, RNF-PERF-02 | ADR-001, Booking, GiST | Constraint de exclusão, teste concorrente, HTTP `409` |
| US04, US05, RNF-DISP-02 | ADR-002, VisitorAccess | Cache/snapshot criptografado, exercício de contingência |
| US07, RNF-CONF-02 | Incident + ADR-001/003 | State machine, histórico append-only, Outbox |
| US09, RNF-PERF-03, RNF-ESC-03 | ADR-003 | RabbitMQ, Worker, retry/DLQ, métrica de backlog |
| US11, RNF-SEG-09/10 | Document + FileStorage | Policy de acesso, URL assinada, TLS/AES-256 |
| US12, US13, RNF-SEG-09 | Financial + ADR-006 | Webhook HMAC, deduplicação, RLS/RBAC |
| RNF-MANT-01/02/03 | ADR-004/005 | Testes arquiteturais, módulos, portas e políticas testáveis |
| RNF-OBS-01/02/03 | Seção 13 | Logs JSON, OTel, dashboards e alertas |
| RNF-CONF-03 | Seção 21 e Anexo I | PITR, teste de restore, RTO/RPO |

# Anexo D — Modelo Físico de Dados e DDL SQL

O DDL completo deve ser executado por migrações versionadas. O modelo segue o ADR-001: `condominium_id` obrigatório, RLS forçada, históricos append-only, Outbox transacional e GiST para reserva.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE booking (
  id uuid PRIMARY KEY,
  condominium_id uuid NOT NULL,
  common_area_id uuid NOT NULL,
  unit_id uuid NOT NULL,
  start_datetime timestamptz NOT NULL,
  end_datetime timestamptz NOT NULL,
  status text NOT NULL CHECK (status IN ('Confirmed', 'Cancelled')),
  created_at timestamptz NOT NULL DEFAULT now(),
  CHECK (start_datetime < end_datetime),
  EXCLUDE USING gist (
    common_area_id WITH =,
    tstzrange(start_datetime, end_datetime, '[)') WITH &&
  ) WHERE (status = 'Confirmed')
);

ALTER TABLE booking ENABLE ROW LEVEL SECURITY;
ALTER TABLE booking FORCE ROW LEVEL SECURITY;
CREATE POLICY booking_tenant_policy ON booking
  USING (condominium_id::text = current_setting('app.current_tenant', true))
  WITH CHECK (condominium_id::text = current_setting('app.current_tenant', true));

CREATE TABLE incident_status_history (
  id uuid PRIMARY KEY,
  condominium_id uuid NOT NULL,
  incident_id uuid NOT NULL,
  previous_status text NOT NULL,
  new_status text NOT NULL,
  changed_by uuid NOT NULL,
  changed_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE outbox_events (
  id uuid PRIMARY KEY,
  condominium_id uuid NOT NULL,
  type text NOT NULL,
  version integer NOT NULL,
  aggregate_id uuid NOT NULL,
  correlation_id text NOT NULL,
  payload jsonb NOT NULL,
  status text NOT NULL DEFAULT 'PENDING',
  attempts integer NOT NULL DEFAULT 0,
  created_at timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz
);
CREATE INDEX outbox_pending_idx ON outbox_events (status, created_at)
  WHERE status = 'PENDING';
```

Permissões operacionais negam `UPDATE` e `DELETE` em históricos. Todas as tabelas multi-tenant recebem política RLS equivalente e índice iniciado por `condominium_id`.

# Anexo E — Matriz de Permissões RBAC

| Recurso | Morador | Síndico | Administrador | Portaria | Conselheiro | Prestador |
|---|---|---|---|---|---|---|
| Reserva | Criar/ver própria | Ver todas | Configurar | — | Ver todas | — |
| Visitante | Criar próprio | Ver todos | — | Consultar/check-in/out | — | — |
| Ocorrência | Criar/ver própria | Gerenciar | — | Registrar | Ver todas | Atualizar atribuída |
| Comunicação | Ler audiência | Publicar | Configurar | Ler | Ler | — |
| Documento | Ler público | Gerenciar | Configurar acesso | — | Ler permitido | — |
| Financeiro | Ver própria unidade | Gerenciar | — | — | Ver consolidado | — |
| Prestador | — | Gerenciar | Cadastrar | — | Ver | Atualizar próprio |

Toda permissão é ainda limitada por tenant; Morador por unidade; Prestador por atribuição; Documento por `access_level`.

# Anexo F — Padrões de Desenvolvimento e Erros

- Rotas são versionadas em `/v1`; escrita aceita `Idempotency-Key` quando a repetição puder causar efeito financeiro ou externo.
- DTO valida forma; Policy/entidade valida regra de domínio; repositório não decide autorização.
- Erros usam `code` estável, mensagem segura, HTTP coerente e `correlationId`; não retornam stack trace, SQL, PII ou segredo.
- Código segue Monólito Modular + Ports & Adapters + Vertical Slices; módulos não importam adaptadores ou repositórios internos de outros módulos.
- Testes mínimos: unitário para invariantes, integração para Postgres/RLS/GiST/Outbox, contrato para adaptadores e arquitetura para dependências.

# Anexo G — Segurança, Classificação e Criptografia

| Classificação | Exemplos | Controles |
|---|---|---|
| Pública | Comunicados públicos, regras publicadas | Integridade, controle de publicação |
| Interna | Configuração de áreas, métricas agregadas | RBAC, RLS, logs sem PII |
| Confidencial | Moradores, visitantes, acesso, ocorrências | Menor privilégio, RLS, auditoria, mascaramento |
| Restrita | Financeiro, contratos, credenciais, tokens | RBAC estrito, TLS 1.3, AES-256, Secrets Manager, auditoria reforçada |

TLS 1.3 protege o trânsito; AES-256 protege banco, backups e objetos em repouso. Segredos nunca entram em Git, imagens, logs ou ConfigMaps.

# Anexo H — Plano de Capacidade de 3 Anos

| Horizonte | Condomínios | Unidades | Usuários | Estratégia de capacidade |
|---|---:|---:|---:|---|
| Ano 1 | 100 | 15.000 | 35.000 | 2–10 APIs, 2–20 workers, Primary/Replica, Redis e RabbitMQ monitorados |
| Ano 2 | 160 | 24.000 | 56.000 | Ajustar HPA/KEDA, pool/índices, storage e filas; teste de carga trimestral |
| Ano 3 | 256 | 38.400 | 89.600 | Revisar particionamento/read model e gatilhos de extração de Notification/Financial |

O crescimento assume 60% ao ano. Escala é guiada por p95/p99, saturação, backlog, replica lag, custo e SLO, não somente por contagem de usuários.

# Anexo I — Plano de Backup Full/Incremental

| Elemento | Política | Evidência |
|---|---|---|
| PostgreSQL | Backup full diário; WAL contínuo para PITR; cópia criptografada e segregada | Restore trimestral com ponto no tempo e medição RTO/RPO |
| Object Storage | Versionamento, replicação/retention conforme classificação | Recuperação amostral de documento e validação de acesso |
| Configuração/IaC | Versionada em Git e backup de estado protegido | Recriação de ambiente de homologação |
| Retenção | Conforme política legal e classificação | Registro de retenção/eliminação auditável |

Meta: RPO <= 15 minutos e RTO <= 2 horas. Backup sem exercício de restauração não é evidência suficiente.

# Anexo J — Checklist de Arquitetura

- [ ] RAS possui decisão, componente e teste rastreável.
- [ ] Tenant está presente em dados, cache, logs e eventos.
- [ ] GiST/RLS/append-only são validados por integração.
- [ ] Módulos obedecem portas e limites de importação.
- [ ] Integrações possuem timeout, retry, circuito, telemetria e idempotência.
- [ ] ADR foi criado/atualizado para decisão estrutural.

# Anexo K — Checklist de Go Live

- [ ] Segurança, RBAC, RLS, MFA, segredos e webhooks foram homologados.
- [ ] Backup/PITR, restore, alertas e runbooks foram exercitados.
- [ ] HPA/KEDA, PDB, probes e NetworkPolicies foram aplicados.
- [ ] k6, concorrência de reserva, contingência de portaria e DLQ foram aprovados.
- [ ] Rollback foi testado e responsáveis de plantão foram definidos.

# Anexo L — Checklist de Revisão do DAS

- [ ] IDs `US*` e `RNF-*` coincidem com o Checkpoint 1.
- [ ] Diagramas C4 e sequência correspondem aos ADRs e ao modelo físico.
- [ ] Trade-offs, custos e gatilhos de revisão estão descritos.
- [ ] Referências Markdown estão funcionais e relativas.
- [ ] Métricas, SLO, RTO/RPO e critérios de aceite são mensuráveis.
- [ ] Não existem decisões sem dono, evidência ou plano de revisão.