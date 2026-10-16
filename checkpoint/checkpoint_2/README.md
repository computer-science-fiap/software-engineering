# Checkpoint 2 — Documento de Arquitetura da Solução

Esta pasta contém o DAS do **Sistema SaaS Multi-tenant de Gestão Condominial**. O documento aprofunda tecnicamente os requisitos e decisões levantados no [Checkpoint 1](../checkpoint_1/README.md), preservando seus IDs e limites de escopo.

## Equipe do Projeto

| Integrante | RM |
|---|---:|
| Ana Carolina Pereira Lopez | 568401 |
| Danilo Roberto Anterio dos Santos | 566966 |
| Enzo Hitoshi Oda | 567313 |
| Henrique Fessel Trench | 567513 |
| José Ribeiro dos Santos Neto | 567692 |

## Como navegar

1. Comece pelo [DAS.md](DAS.md), que consolida a referência arquitetural e o índice.
2. Leia [Fundamentos e RAS](docs/01-fundamentos-e-direcionadores/direcionadores-e-ras.md) para entender o vínculo entre negócio, requisitos e aceite.
3. Consulte [Arquitetura C4](docs/02-arquitetura-c4/estilo-e-diagramas-c4.md) para contexto, containers, integrações e estilo.
4. Consulte [Interações, GoF e DDD](docs/03-interacoes-padroes-e-dominio/sequencia-gof-ddd.md) para componentes, código de referência e sequências.
5. Consulte [ADRs](docs/04-adrs/catalogo-adrs.md) para decisões, trade-offs e gatilhos de revisão.
6. Consulte [Segurança, Operação e SRE](docs/05-seguranca-operacao-sre/seguranca-k8s-confiabilidade.md) para controles e prontidão operacional.
7. Consulte [Anexos e Checklists](docs/06-anexos-e-checklists/anexos-a-ate-l.md) para contratos técnicos e evidências.

## Estrutura

```text
checkpoint_2/
├── DAS.md
├── README.md
└── docs/
    ├── 01-fundamentos-e-direcionadores/
    ├── 02-arquitetura-c4/
    ├── 03-interacoes-padroes-e-dominio/
    ├── 04-adrs/
    ├── 05-seguranca-operacao-sre/
    └── 06-anexos-e-checklists/
```

## Convenções

- IDs `US01`–`US13` e `RNF-*` são provenientes do Checkpoint 1 e não podem ser renomeados.
- Diagramas usam Mermaid e devem descrever fronteiras técnicas, não redefinir processos de negócio.
- PostgreSQL é a fonte de verdade; Redis é cache/sessão/lock auxiliar; eventos externos passam por Outbox e RabbitMQ.
- Alterações arquiteturais relevantes exigem ADR, teste e atualização da matriz de rastreabilidade.