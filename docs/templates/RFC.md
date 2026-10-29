# RFC-[Número] — [Título da Proposta ou Sistema]

| Campo | Valor |
|---|---|
| **Autor** | [Nome do Autor] |
| **Status** | [Rascunho / Em revisão / Aprovado / Rejeitado] |
| **Data** | [AAAA-MM-DD] |
| **Revisores** | [Nome dos Revisores e papéis] |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) |

> [Breve sumário ou contexto sobre o momento da RFC, quem participou das discussões iniciais e para quem a proposta está submetida.]

---

## 1. TL;DR
[Resumo executivo de até 1 parágrafo contendo a essência da proposta, a abordagem técnica escolhida, as garantias principais e o impacto geral, permitindo uma leitura rápida para decisores.]

---

## 2. Contexto e problema
[Descreva o problema de negócio ou técnico que motivou a proposta, quais clientes ou áreas internas são afetadas, prazos críticos ou restrições, e os requisitos fundamentais que a solução deve atender.]

*Exemplo:*
> Clientes pediram formalmente para ser notificados quando o status dos seus pedidos muda. Hoje eles fazem polling periódico, gerando gargalos. Precisamos de uma solução assíncrona, segura e que caiba no prazo de 3 sprints sem criar nova infraestrutura.

---

## 3. Proposta técnica

### 3.1 Visão geral
[Apresente um diagrama conceitual, fluxo textual ou arquitetural demonstrando como a solução opera de ponta a ponta.]

### 3.2 Componentes
| Componente | Responsabilidade | Decisão / Referência |
|---|---|---|
| [Ex: Outbox] | [Ex: Registro atômico do evento] | [ADR-001](adrs/...) |

### 3.3 Superfície pública (resumo)
[Descreva resumidamente os endpoints, contratos ou interfaces expostas para consumo interno ou externo.]

---

## 4. Alternativas consideradas
| # | Alternativa | Por que foi descartada (trade-off) |
|---|---|---|
| ALT-01 | [Ex: Chamada síncrona] | [Ex: Clientes lentos travam o sistema principal] |

---

## 5. Impacto
| Área | Impacto |
|---|---|
| **Código existente** | [Ex: Novos módulos, alteração de serviços pontuais] |
| **Banco de dados** | [Ex: Inclusão de novas tabelas ou índices] |
| **Operação** | [Ex: Novos processos em background para deployar] |
| **Prazos** | [Ex: Estimativa de entrega] |

---

## 6. Riscos

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| [Ex: Cliente lento] | [Média] | [Alto] | [Ex: Paralelismo controlado e timeout] |

---

## 7. Questões em aberto
| # | Questão | Situação na discussão | Proposta para resolução |
|---|---|---|---|
| Q-01 | [Ex: Rate limiting] | [Ex: Fora do escopo atual] | [Ex: Monitorar e reabrir se necessário] |

**Explicitamente fora de escopo:** [Liste o que foi deliberadamente cortado desta versão para evitar escopo infinito.]

---

## 8. Decisões relacionadas
* [ADR-001 — Título da Decisão Arquitetural](adrs/ADR-001-exemplo.md)