# ADR-004: Pontuação como livro-razão append-only com saldo derivado

**Data:** 2026-09-12
**Status:** Aceito
**Decisores:** Arquitetura
**Consultados:** Produto

---

## Contexto

O módulo M4 atribui pontos que são trocados por benefícios reais no comércio local. Ainda que os pontos não tenham expressão monetária (RN-042.4), eles têm valor percebido, e erro de saldo destrói a confiança no produto inteiro. A RF-043 exige extrato com origem de cada movimento, e a seção 8 do PRD identifica fraude na pontuação como risco de impacto médio.

A pergunta é qual estrutura representa o saldo: um número que se altera, ou uma sequência de movimentos cuja soma é o saldo.

**Restrições conhecidas:**
- Extrato retido por no mínimo 24 meses (RN-043.2).
- Resgate precisa verificar saldo e debitar de forma atômica, resistindo a requisições concorrentes (FE-2 da RF-042).
- Crédito chega de forma assíncrona a partir da sincronização do catador, potencialmente em lote.
- Volume modesto: cerca de 1.300 usuários ativos, com dezenas de movimentos por usuário por ano.

**Critérios de decisão:**
1. Auditabilidade e capacidade de explicar qualquer saldo
2. Correção sob concorrência
3. Simplicidade de implementação
4. Performance de leitura

---

## Alternativas Consideradas

### Opção A: Campo de saldo mutável no usuário

**Descrição:** Coluna `saldo_pontos` em `usuarios`, incrementada e decrementada a cada evento, com tabela de histórico mantida em paralelo para exibição.

**Prós:**
- Leitura de saldo trivial.
- Implementação imediata.

**Contras:**
- Duas fontes de verdade que podem divergir. Quando divergem, não há como saber qual está certa.
- Qualquer bug de escrita corrompe o saldo de forma irreversível, sem trilha para reconstruí-lo.
- O histórico vira registro decorativo, não lastro.

**Avaliação pelos critérios:**
| Critério | Score (1-5) | Justificativa |
|---|---|---|
| Auditabilidade | 1 | Saldo não é explicável a partir dos movimentos |
| Correção sob concorrência | 2 | Exige lock e mesmo assim não é reconstituível |
| Simplicidade | 5 | A mais simples de escrever |
| Performance de leitura | 5 | Leitura direta |

---

### Opção B: Livro-razão append-only com saldo derivado por soma

**Descrição:** Tabela `lancamentos_pontos` recebe apenas inserções, nunca alteração nem exclusão. O saldo é a soma dos lançamentos do usuário. Correção se faz por lançamento de ajuste, nunca por edição.

**Prós:**
- Fonte de verdade única. Todo saldo é explicável lançamento a lançamento.
- Correção deixa rastro, atendendo à RN-033.4 e ao requisito de ajuste manual do Admin.
- Detecção de anomalia fica natural, bastando agregar lançamentos por janela de tempo.
- A RF-043 é satisfeita por construção, sem estrutura adicional.

**Contras:**
- Leitura de saldo exige agregação. Com dezenas de lançamentos por usuário, é irrelevante; com dezenas de milhares, deixaria de ser.
- Verificação de saldo no resgate exige lock explícito na linha do usuário.

**Avaliação pelos critérios:**
| Critério | Score (1-5) | Justificativa |
|---|---|---|
| Auditabilidade | 5 | Saldo é consequência dos movimentos |
| Correção sob concorrência | 5 | Lock na linha do usuário resolve o único ponto crítico |
| Simplicidade | 4 | Uma tabela, uma regra |
| Performance de leitura | 4 | Adequada nos volumes previstos |

---

### Opção C: Livro-razão com saldo materializado

**Descrição:** Como a Opção B, mais uma coluna de saldo atualizada na mesma transação do lançamento, com reconciliação periódica comparando soma e snapshot.

**Prós:**
- Auditabilidade da Opção B com leitura constante.
- Suporta volume muito maior sem degradação.

**Contras:**
- Reintroduz a redundância que a Opção B eliminou, com o trabalho extra de reconciliar e alertar divergência.
- Otimiza para um problema de escala que este sistema não tem, e que a seção 4.4 do PRD afirma que não terá.

**Avaliação pelos critérios:**
| Critério | Score (1-5) | Justificativa |
|---|---|---|
| Auditabilidade | 5 | Ledger permanece a fonte de verdade |
| Correção sob concorrência | 5 | Atualização na mesma transação |
| Simplicidade | 3 | Exige rotina de reconciliação e tratamento de divergência |
| Performance de leitura | 5 | Constante |

---

## Decisão

**Escolhemos: Opção B — livro-razão append-only com saldo derivado por soma.**

A Opção A é descartada por razão de princípio. Num sistema em que pontos viram benefício real, o saldo precisa ser explicável, e a Opção A não consegue explicar o próprio número.

A escolha relevante é entre B e C, e é uma escolha sobre disciplina de escopo. A Opção C é a resposta correta em escala, mas a seção 4.4 do PRD é explícita ao afirmar que o gargalo do piloto é operacional e não computacional. Um usuário deste sistema acumula algo na faixa de dezenas de lançamentos por ano. Somar essas linhas é irrelevante para qualquer banco relacional, e adotar C agora significa construir e manter uma rotina de reconciliação para resolver um problema que ainda não existe.

O ponto de correção concorrente, que é o argumento mais forte a favor de C, é resolvido em B com lock na linha do usuário durante a transação de resgate: adquire-se o lock, soma-se o saldo, valida-se e insere-se o débito. Uma única linha sob contenção, com requisições de resgate de baixa frequência.

A migração de B para C é aditiva e barata: acrescentar coluna, preencher por soma e passar a atualizar na transação. Não há quebra de contrato nem de dados. Por isso o gatilho de revisão fica registrado explicitamente, em vez de adotarmos C preventivamente.

**Gatilho de revisão:** quando qualquer usuário ultrapassar 5.000 lançamentos, ou quando o P95 da consulta de saldo passar de 100ms.

---

## Consequências

### Positivas
- Qualquer contestação de saldo é respondida mostrando os lançamentos, sem investigação.
- Ajuste manual do Admin nasce auditável, com autor e motivo.
- Detecção de anomalia da seção 4.5 do PRD se implementa como agregação simples sobre o ledger.
- Estorno de resgate vira um lançamento de crédito, não uma reversão de estado.

### Negativas / Trade-offs aceitos
- Consulta de saldo tem custo proporcional ao histórico. Aceito, com gatilho de revisão documentado.
- Resgate serializa por usuário. Aceito, já que resgates concorrentes do mesmo usuário são patológicos, não normais.
- O ledger cresce indefinidamente. Mitigação: a retenção de 24 meses da RN-043.2 permite consolidar lançamentos mais antigos em um único lançamento de abertura, se necessário.

### Riscos e Mitigações
| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Alguém adicionar UPDATE no ledger no futuro | Média | Alto | Revogar UPDATE e DELETE na tabela em nível de banco para o usuário da aplicação |
| Lançamento sem referência de origem | Baixa | Médio | Coluna de origem obrigatória, com restrição de integridade contra a entidade correspondente |
| Crédito em lote na sincronização gerar contenção | Baixa | Baixo | Créditos são inserções, sem lock de leitura de saldo |

---

## Implementação

**Impacta:**
- [ ] Infraestrutura
- [x] Backend
- [ ] Frontend
- [x] Banco de dados
- [ ] Integrações externas

**Tarefas necessárias para implementar esta decisão:**
1. Modelar `lancamentos_pontos` com tipo, valor com sinal, origem polimórfica e chave de idempotência.
2. Revogar UPDATE e DELETE na tabela para o papel da aplicação.
3. Implementar a transação de resgate com lock na linha do usuário.
4. Implementar a agregação de anomalia diária prevista na seção 4.5.
5. Definir os tipos de lançamento: crédito de coleta, crédito de check-in, débito de resgate, ajuste de admin, estorno.

**Métricas de validação:**
Zero divergência entre saldo exibido e soma do ledger. Resgates contestados abaixo de 1% conforme meta do PRD. P95 da consulta de saldo monitorado contra o gatilho de revisão.

