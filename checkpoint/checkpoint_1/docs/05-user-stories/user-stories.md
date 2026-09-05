# 05 — User Stories

## 5.1 Módulo: Reservas

### US01 — Reservar espaço comum
**Como** morador
**Quero** reservar um espaço comum (salão, churrasqueira, quadra, academia)
**Para** garantir o uso do espaço sem depender de contato manual com o síndico

**Critérios de aceitação:**
- O sistema deve exibir apenas horários disponíveis para o espaço selecionado;
- Ao confirmar, a reserva deve ficar visível para o morador e para o síndico/portaria;
- O morador deve receber notificação de confirmação.

**Regras de negócio:**
- Não pode haver duas reservas confirmadas com sobreposição de horário para o mesmo espaço;
- Cada unidade pode ter um limite configurável de reservas simultâneas/mês (definido pelo síndico).



---

### US02 — Cancelar reserva
**Como** morador
**Quero** cancelar uma reserva que fiz
**Para** liberar o horário para outro morador caso eu não vá mais usar o espaço

**Critérios de aceitação:**
- Cancelamento deve liberar o horário imediatamente para nova reserva;
- Sistema deve notificar a portaria sobre o cancelamento.

**Regras de negócio:**
- Pode existir prazo mínimo de antecedência para cancelamento (configurável pelo síndico).

**Prioridade:** Média
**Dependências:** US01

---

### US03 — Configurar espaços e regras de reserva
**Como** síndico
**Quero** configurar quais espaços existem, seus horários de funcionamento e regras de uso
**Para** adaptar o sistema às regras específicas do meu condomínio

**Critérios de aceitação:**
- Síndico pode cadastrar/editar espaços, horário de funcionamento e limite de reservas.

**Prioridade:** Alta
**Dependências:** Nenhuma (pré-requisito de US01)

## 5.2 Módulo: Visitantes

### US04 — Pré-cadastrar visitante
**Como** morador
**Quero** cadastrar antecipadamente um visitante esperado
**Para** agilizar a entrada dele na portaria

**Critérios de aceitação:**
- Sistema gera um registro de autorização vinculado à unidade e a uma data/período de validade;
- Portaria consegue consultar essa autorização pelo nome do visitante.

**Regras de negócio:**
- Autorização expira automaticamente após a data/período informado.

**Prioridade:** Alta
**Dependências:** Cadastro de moradores/unidades

---

### US05 — Registrar entrada/saída de visitante
**Como** funcionário da portaria
**Quero** registrar a entrada e saída de um visitante
**Para** manter um histórico confiável de acesso ao condomínio

**Critérios de aceitação:**
- Sistema deve permitir consulta rápida por nome/unidade;
- Deve registrar timestamp de entrada e de saída.

**Regras de negócio:**
- Visitante sem autorização prévia deve poder ser registrado mediante contato com o morador (fluxo de exceção).

**Prioridade:** Alta
**Dependências:** US04

## 5.3 Módulo: Ocorrências

### US06 — Registrar ocorrência
**Como** morador
**Quero** registrar um problema ou ocorrência (ex: manutenção, barulho, dano)
**Para** que o síndico tome conhecimento formalmente e possa agir

**Critérios de aceitação:**
- Ocorrência criada com status inicial "Aberta";
- Morador pode acompanhar o status a qualquer momento.

**Prioridade:** Alta
**Dependências:** Nenhuma

---

### US07 — Acompanhar e atualizar status de ocorrência
**Como** síndico
**Quero** classificar e atualizar o status de uma ocorrência
**Para** dar andamento formal até a resolução

**Critérios de aceitação:**
- Status deve seguir o fluxo: Aberta → Em análise → Em atendimento → Resolvida;
- Morador deve ser notificado a cada mudança de status.

**Regras de negócio:**
- Não é possível pular diretamente de "Aberta" para "Resolvida" sem passar pelos status intermediários (garante rastreabilidade).

**Prioridade:** Alta
**Dependências:** US06

---

### US08 — Atualizar status de execução de serviço
**Como** prestador de serviço
**Quero** atualizar o status de uma ocorrência/ordem de serviço atribuída a mim
**Para** manter o síndico e o morador informados sobre o andamento

**Prioridade:** Média
**Dependências:** US07, Módulo Prestadores

## 5.4 Módulo: Comunicação

### US09 — Enviar comunicado segmentado
**Como** síndico
**Quero** enviar um comunicado para todos os moradores ou apenas para um bloco/unidade específica
**Para** garantir que a informação certa chegue ao público certo

**Critérios de aceitação:**
- Síndico pode escolher o alcance do comunicado (todos, bloco, unidade);
- Sistema envia notificação (push/e-mail) e registra o comunicado no histórico.

**Prioridade:** Alta
**Dependências:** Cadastro de blocos/unidades

---

### US10 — Confirmar leitura de comunicado
**Como** síndico
**Quero** visualizar quais moradores já leram um comunicado
**Para** medir o alcance da comunicação e cobrar formalmente quando necessário

**Prioridade:** Média
**Dependências:** US09

## 5.5 Módulo: Documentos

### US11 — Acessar documentos do condomínio
**Como** morador
**Quero** acessar atas, convenção e regimento interno
**Para** me informar sem precisar solicitar diretamente ao síndico

**Regras de negócio:**
- Documentos sigilosos (ex: contratos com prestadores) só ficam visíveis para síndico/conselheiro.

**Prioridade:** Média
**Dependências:** Módulo Gestão do Condomínio (perfis e permissões)

## 5.6 Módulo: Financeiro (MVP controlado)

### US12 — Consultar status de pagamento da própria unidade
**Como** morador
**Quero** ver se estou em dia com a taxa condominial
**Para** ter transparência sobre minha situação financeira

**Prioridade:** Média
**Dependências:** Cadastro de unidades

---

### US13 — Consultar prestação de contas consolidada
**Como** conselheiro
**Quero** visualizar relatórios financeiros consolidados do condomínio
**Para** exercer a fiscalização da gestão

**Prioridade:** Média
**Dependências:** US12