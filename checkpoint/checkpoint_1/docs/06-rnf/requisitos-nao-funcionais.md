# 06 — Requisitos Não Funcionais (RNF)

## 6.1 Segurança

### Autenticação
- **RNF-SEG-01:** O sistema deve exigir autenticação para qualquer acesso além de páginas públicas (ex: tela de login).
- **RNF-SEG-02:** Autenticação deve suportar recuperação de senha segura (token com expiração curta).
- **RNF-SEG-03:** Sessões devem expirar por inatividade (tempo configurável).

### Autorização
- **RNF-SEG-04:** O sistema deve implementar controle de acesso baseado em papéis (RBAC), refletindo os 6 perfis definidos em Stakeholders (Morador, Síndico, Administrador, Portaria, Conselheiro, Prestador).
- **RNF-SEG-05:** Um morador só pode acessar dados da própria unidade (US01, US04, US06, US12) — nunca de outra unidade.
- **RNF-SEG-06:** Ações sensíveis (aprovar ocorrência, emitir comunicado, ver financeiro consolidado) devem ser restritas por papel, validadas no backend.

### Proteção de Dados / LGPD
- **RNF-SEG-07:** Dados pessoais de moradores, dependentes, visitantes e prestadores devem ser tratados conforme a LGPD — coleta com finalidade específica e informada.
- **RNF-SEG-08:** O sistema deve permitir que o morador solicite acesso, correção ou exclusão de seus dados pessoais (direito do titular).
- **RNF-SEG-09:** Dados financeiros e documentos sigilosos (contratos, atas restritas) devem ter controle de acesso por perfil (ligado a RNF-SEG-04).
- **RNF-SEG-10:** Dados sensíveis em trânsito e em repouso devem ser criptografados (TLS em trânsito; criptografia no banco para campos sensíveis).
- **RNF-SEG-11:** Logs de acesso a dados pessoais (quem viu o quê) devem ser mantidos para fins de auditoria/LGPD.

## 6.2 Performance

- **RNF-PERF-01:** Operações de consulta comuns (ver reservas, ver ocorrências, ver comunicados) devem responder em até 1-2 segundos sob carga normal.
- **RNF-PERF-02:** A verificação de conflito de reserva (US01) deve ser síncrona e retornar resultado em tempo real, para evitar dupla reserva por concorrência.
- **RNF-PERF-03:** Notificações (push/e-mail) podem ser assíncronas — não devem bloquear a ação principal do usuário (ex: confirmar reserva não espera o envio do e-mail).

## 6.3 Escalabilidade

- **RNF-ESC-01:** A arquitetura deve suportar crescimento horizontal do número de condomínios (multi-tenant), sem exigir nova instância de aplicação por cliente.
- **RNF-ESC-02:** O modelo de dados deve isolar logicamente os dados de cada condomínio (tenant), permitindo crescer em número de unidades/moradores sem degradar performance de outros tenants.
- **RNF-ESC-03:** Componentes de alto volume e assíncronos (notificações, e-mails) devem poder escalar independentemente do core da aplicação (fila/mensageria dedicada).

## 6.4 Disponibilidade

- **RNF-DISP-01:** O sistema deve ter meta de disponibilidade (SLA) compatível com uso diário essencial (ex: 99,5% mensal), já que envolve controle de acesso de visitantes (relevante para segurança física).
- **RNF-DISP-02:** Funcionalidades críticas de portaria (consulta de autorização de visitante) devem ter estratégia de degradação graciosa (ex: cache local/offline mínimo) para não travar o acesso ao condomínio em caso de instabilidade momentânea.
- **RNF-DISP-03:** Falhas em serviços não-críticos (ex: envio de notificação) não podem indisponibilizar funcionalidades críticas (reserva, autorização de visitante).

## 6.5 Confiabilidade

- **RNF-CONF-01:** O sistema deve garantir integridade transacional na criação de reservas — duas requisições simultâneas para o mesmo horário não podem resultar em duas reservas confirmadas (relacionado à regra de negócio de US01).
- **RNF-CONF-02:** Mudanças de status de ocorrência (US07) devem ser auditáveis — manter histórico de quem alterou e quando, sem permitir edição retroativa silenciosa.
- **RNF-CONF-03:** O sistema deve ter rotina de backup do banco de dados, com procedimento de recuperação testável (RPO/RTO definidos).

## 6.6 Observabilidade

- **RNF-OBS-01:** O sistema deve registrar logs estruturados de operações críticas (reservas, ocorrências, autorizações de visitante, alterações financeiras).
- **RNF-OBS-02:** Devem existir métricas técnicas (tempo de resposta, taxa de erro, throughput) e métricas de negócio (nº de reservas, tempo médio de resolução de ocorrência — ligado aos KPIs da Visão de Negócio).
- **RNF-OBS-03:** Deve existir monitoramento com alertas para falhas em componentes críticos (ex: fila de notificações parada, banco indisponível).

## 6.7 Manutenibilidade

- **RNF-MANT-01:** O backend deve ser modularizado por domínio (Reservas, Visitantes, Ocorrências, Comunicação, Documentos, Financeiro, Prestadores), permitindo evolução independente de cada módulo.
- **RNF-MANT-02:** O sistema deve seguir um padrão arquitetural claro e documentado (a definir na Visão Arquitetural), evitando acoplamento direto entre módulos de domínio.
- **RNF-MANT-03:** Regras de negócio críticas (ex: conflito de reserva, transições de status de ocorrência) devem ser cobertas por testes automatizados, dado seu impacto direto na confiabilidade percebida pelo usuário.
- **RNF-MANT-04:** Código e documentação técnica devem ser versionados no GitHub, com histórico de decisões arquiteturais (ADR) para justificar escolhas técnicas relevantes.

## 6.8 Rastreabilidade com User Stories (amostra)

| RNF | User Story relacionada |
|---|---|
| RNF-SEG-04/05/06 | US01, US04, US06, US12 (acesso restrito por perfil/unidade) |
| RNF-PERF-02, RNF-CONF-01 | US01 (conflito de reserva) |
| RNF-SEG-07/08/09 | US04, US11, US12, US13 (dados pessoais e financeiros) |
| RNF-OBS-02 | Todos os KPIs definidos na Visão de Negócio |