# ADR-[Número] — [Título da Decisão Arquitetural]

- **Status:** [Proposto / Aceito / Rejeitado / Substituído]
- **Data da decisão:** [Data ou período da discussão]
- **Decisores:** [Nomes e papéis dos decisores]
- **Relacionados:** [Links para outras ADRs, RFCs ou seções de FDD]

## Contexto

[Descreva o problema técnico, a funcionalidade ou restrição de sistema que exige uma decisão arquitetural, citando os componentes de software envolvidos e as forças em jogo (requisitos de negócio, limitações de equipe, infraestrutura existente).]

*Exemplo:*
> Um microsserviço precisa registrar eventos de forma atômica junto com operações transacionais no banco, sem acoplar a latência de chamadas externas ao fluxo principal e sem adicionar nova infraestrutura.

## Decisão

[Descreva claramente a decisão arquitetural tomada, enumerando os pontos principais de implementação ou diretrizes técnicas estabelecidas.]

1. [Ponto principal 1 da decisão arquitetural]
2. [Ponto principal 2 da decisão arquitetural]
3. [Exclusões ou limites explícitos acordados]

## Alternativas consideradas

### A1. [Nome da alternativa descartada 1]
- **Prós:** [Vantagens técnicas ou operacionais]
- **Contras:** [Desvantagens ou riscos]
- **Motivo do descarte:** [Por que a opção foi rejeitada]

### A2. [Nome da alternativa descartada 2]
- **Prós:** [...]
- **Contras:** [...]
- **Motivo do descarte:** [...]

## Consequências

### Positivas
- [Benefício arquitetural ou operacional 1 - ex: Consistência forte e zero infra nova]
- [Benefício 2]

### Negativas
- [Custo ou ônus técnico 1 - ex: Aumento da carga no banco transacional]
- [Custo 2]

### Trade-off explícito
[Descreva o compromisso central assumido pela equipe: o que foi sacrificado em troca do ganho principal.]