# 03 — Cenário As-Is

## 3.1 Visão Geral do Processo Atual

Atualmente, a operação do condomínio depende de canais fragmentados e não integrados: WhatsApp, planilhas, e-mail, avisos físicos no mural e controle manual em papel/caderno na portaria.

## 3.2 Fluxo Atual — Reserva de Espaço Comum

```mermaid
flowchart TD
    A[Morador quer reservar espaço] --> B[Envia mensagem no WhatsApp do síndico/zelador]
    B --> C{Síndico/zelador consulta caderno ou planilha}
    C -->|Horário livre| D[Confirma manualmente por mensagem]
    C -->|Conflito não percebido| E[Reserva duplicada]
    D --> F[Anota na planilha/caderno]
    E --> G[Conflito descoberto só no dia do evento]
```

**Gargalos:** dependência de resposta manual do síndico/zelador; sem verificação automática de conflito; risco de duplicidade só descoberto na hora do uso.

## 3.3 Fluxo Atual — Controle de Visitantes

```mermaid
flowchart TD
    A[Morador avisa portaria sobre visitante] --> B{Como avisa?}
    B -->|Verbal/telefone| C[Porteiro anota em caderno]
    B -->|Esquece de avisar| D[Visitante chega sem autorização prévia]
    D --> E[Porteiro tenta ligar para morador]
    E -->|Sem resposta| F[Visitante aguarda ou é barrado]
```

**Gargalos:** ausência de pré-autorização confiável; dependência de contato telefônico no momento da chegada; sem histórico digital de quem entrou/saiu.

## 3.4 Fluxo Atual — Ocorrências

```mermaid
flowchart TD
    A[Morador percebe problema, ex: elevador com defeito] --> B[Avisa grupo de WhatsApp ou porteiro]
    B --> C[Síndico toma conhecimento, às vezes com atraso]
    C --> D[Síndico aciona prestador informalmente]
    D --> E[Execução do serviço]
    E --> F{Morador é informado da resolução?}
    F -->|Nem sempre| G[Falta de retorno formal]
```

**Gargalos:** sem status visível (aberta/em análise/resolvida); informação se perde em grupos de WhatsApp; morador não tem retorno formal.

## 3.5 Fluxo Atual — Comunicação Geral

```mermaid
flowchart TD
    A[Síndico tem um comunicado] --> B{Canal usado}
    B --> C[Aviso físico no mural]
    B --> D[Grupo de WhatsApp]
    B --> E[E-mail avulso]
    C --> F[Nem todo morador vê]
    D --> G[Mensagem se perde no histórico do grupo]
    E --> H[Baixa taxa de leitura]
```

**Gargalos:** nenhum canal garante alcance nem confirmação de leitura; comunicados importantes se perdem entre mensagens informais.

## 3.6 Problemas Existentes

| Categoria | Problema |
|---|---|
| Comunicação | Fragmentada entre WhatsApp, mural físico e e-mail; sem confirmação de leitura |
| Reservas | Conflitos de horário descobertos tardiamente |
| Visitantes | Sem pré-autorização confiável; dependência de contato telefônico |
| Ocorrências | Sem status visível; retorno ao morador inconsistente |
| Documentos | Difícil acesso (pasta física, e-mail perdido) |
| Financeiro | Prestação de contas manual, pouco transparente |
| Governança | Dependência excessiva da memória e disponibilidade do síndico |

## 3.7 Custos Operacionais Atuais 

- Tempo do síndico/zelador gasto respondendo mensagens repetitivas (reservas, dúvidas, autorizações);
- Retrabalho por conflitos de reserva e falta de registro único;
- Custo de impressão/afixação de avisos físicos;
- Tempo da portaria tentando confirmar visitantes por telefone.

## 3.8 Riscos do Modelo Atual

- **Risco operacional:** decisões e histórico dependem de uma pessoa (síndico), sem continuidade formal se ela sair ou estiver indisponível;
- **Risco de segurança:** controle de visitantes frágil, sem rastreabilidade;
- **Risco jurídico/administrativo:** falta de registro formal de decisões e prestação de contas pode gerar contestação de moradores;
- **Risco de reputação/satisfação:** comunicação falha gera insatisfação e desconfiança na gestão.