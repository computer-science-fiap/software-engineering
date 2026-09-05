# 02 — Stakeholders

## 2.1 Visão Geral

O sistema possui 6 perfis de atores, cada um com necessidades, responsabilidades e níveis de acesso distintos. Essa segmentação é a base para o modelo de permissões (RBAC) que será detalhado na Visão Arquitetural.

## 2.2 Matriz de Stakeholders

### Morador
- **O que é:** Usuário final que reside na unidade condominial.
- **Necessidades:** Comunicação clara, agilidade em reservas, transparência financeira, facilidade para registrar/acompanhar ocorrências.
- **Responsabilidades no sistema:** Manter cadastro atualizado (dependentes, veículos, pets), respeitar regras de uso dos espaços comuns, cadastrar visitantes antecipadamente.
- **Nível de acesso:** Restrito à própria unidade/família. Não vê dados de outras unidades.
- **Dores atuais:** Não sabe se a reserva foi confirmada; não tem visibilidade do andamento de uma ocorrência; recebe avisos picados via grupos de WhatsApp.

### Síndico
- **O que é:** Representante legal do condomínio, eleito pelos condôminos.
- **Necessidades:** Visão consolidada da operação, agilidade para comunicar e decidir, prestação de contas facilitada.
- **Responsabilidades no sistema:** Aprovar/gerenciar ocorrências, emitir comunicados oficiais, supervisionar financeiro, gerenciar prestadores.
- **Nível de acesso:** Amplo — visão de todo o condomínio (todas as unidades, financeiro, ocorrências, documentos).
- **Dores atuais:** Sobrecarga de decisões operacionais repetitivas; dificuldade de manter histórico e provar decisões tomadas.

### Administrador (da plataforma/condomínio)
- **O que é:** Responsável técnico/operacional pela configuração do condomínio no sistema (pode ser a administradora contratada).
- **Necessidades:** Configurar rapidamente blocos, unidades, perfis e permissões; gerenciar múltiplos condomínios (se for administradora).
- **Responsabilidades no sistema:** Cadastro estrutural do condomínio, gestão de perfis e permissões, suporte técnico de primeiro nível.
- **Nível de acesso:** Administrativo total sobre configuração; pode ou não ter acesso operacional dependendo do modelo (síndico profissional vs. administradora terceirizada).
- **Dores atuais:** Gerencia múltiplos condomínios em planilhas/sistemas diferentes, sem padronização.

### Funcionário/Portaria
- **O que é:** Colaborador que opera a rotina do condomínio no local (portaria, zeladoria).
- **Necessidades:** Interface simples e rápida para uso operacional contínuo (ex: liberar visitante, registrar ocorrência reportada por morador presencialmente).
- **Responsabilidades no sistema:** Controle de entrada/saída de visitantes, registro inicial de ocorrências, apoio na comunicação.
- **Nível de acesso:** Operacional, limitado a visitantes e ocorrências — sem acesso a financeiro ou documentos sigilosos.
- **Dores atuais:** Depende de listas físicas ou grupos de WhatsApp para saber quem tem visita autorizada.

### Conselheiro
- **O que é:** Membro do conselho fiscal/consultivo do condomínio, eleito pelos condôminos.
- **Necessidades:** Fiscalizar a gestão do síndico com transparência.
- **Responsabilidades no sistema:** Acompanhar prestação de contas, acessar documentos e atas, opinar/aprovar decisões relevantes.
- **Nível de acesso:** Leitura ampla sobre financeiro e documentos; geralmente sem permissão de execução/alteração.
- **Dores atuais:** Acesso a informações financeiras depende da boa vontade/agenda do síndico.

### Prestador de Serviço
- **O que é:** Empresa ou profissional contratado para serviços no condomínio (manutenção, limpeza, jardinagem etc.).
- **Necessidades:** Saber o que foi solicitado, registrar execução do serviço.
- **Responsabilidades no sistema:** Atualizar status de serviços/contratos sob sua responsabilidade.
- **Nível de acesso:** Muito restrito — apenas aos próprios contratos/ordens de serviço.
- **Dores atuais:** Comunicação sobre demandas é informal, sem rastreabilidade de execução.

## 2.3 Matriz de Permissões

| Módulo | Morador | Síndico | Admin | Portaria | Conselheiro | Prestador |
|---|---|---|---|---|---|---|
| Reservas | Criar/ver próprias | Ver todas | Configurar espaços | — | Ver todas | — |
| Visitantes | Cadastrar próprios | Ver todos | — | Registrar entrada/saída | — | — |
| Ocorrências | Criar/ver próprias | Gerenciar todas | — | Registrar | Ver todas | Ver atribuídas |
| Comunicação | Ler | Emitir | Configurar segmentação | Ler | Ler | — |
| Documentos | Ler públicos | Gerenciar | Configurar acesso | — | Ler todos | — |
| Financeiro | Ver próprio | Gerenciar | — | — | Ver consolidado | — |
| Prestadores | — | Gerenciar | Cadastrar | — | Ver | Atualizar próprio status |

