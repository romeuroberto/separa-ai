# ADR-003: Offline-first por fila de operações idempotentes no app do catador

**Data:** 2026-09-12
**Status:** Aceito, cláusula de retenção local substituída pelo ADR-015
**Decisores:** Arquitetura
**Consultados:** a validar com a cooperativa

> **Nota de revisão (2026-09-13):** a cláusula "Retenção local" abaixo estabelece prazo único de 7 dias para a rota e suas operações. O ADR-015 a substitui por dois relógios, separando o dado pessoal da fila de operações. O texto original permanece inalterado como registro da decisão de 2026-09-12; onde ele e o ADR-015 divergirem, prevalece o ADR-015. O restante deste ADR segue integralmente válido.

---

## Contexto

A RN-033.1 torna o modo offline não negociável no app do catador. A questão aqui não é se haverá offline, mas qual o desenho e, sobretudo, qual o escopo, dado que sincronização é uma das áreas onde equipes pequenas mais se afogam.

O padrão de escrita do domínio é favorável e vale explicitar. Os dados fluem quase inteiramente numa direção: o Coordenador publica a rota, o Catador registra desfechos em campo, o servidor recebe. Não existe edição concorrente do mesmo registro por dois usuários. O único conflito real previsto é o da FE-2 da RF-033, quando o Coordenador altera no escritório uma parada que o catador já finalizou em campo.

**Restrições conhecidas:**
- Conectividade intermitente durante toda a jornada, podendo ser nula por horas.
- Registro de pesagem é imutável após sincronizado (RN-033.4) e lastreia crédito de pontos.
- Bateria é recurso escasso, o que limita sincronização agressiva em segundo plano.
- Motor construído sob medida, conforme aceito no ADR-002.

**Critérios de decisão:**
1. Garantia de não perder registro de campo
2. Ausência de crédito duplicado de pontos
3. Simplicidade de implementação e depuração
4. Consumo de bateria e dados

---

## Alternativas Consideradas

### Opção A: Cache de leitura com escrita exclusivamente online

**Descrição:** A rota é baixada e fica legível offline, mas o registro de desfecho exige rede no momento da ação.

**Prós:**
- Trivial de implementar, sem estado local mutável.
- Nenhuma possibilidade de conflito ou duplicação.

**Contras:**
- Inviabiliza a operação. O catador em zona sem sinal ficaria impedido de registrar, e registraria depois de memória ou no papel, que é exatamente o processo que a plataforma existe para substituir.

**Avaliação pelos critérios:**
| Critério | Score (1-5) | Justificativa |
|---|---|---|
| Não perder registro | 1 | Registro simplesmente não acontece sem rede |
| Sem duplicação | 5 | Não há fila |
| Simplicidade | 5 | A mais simples |
| Bateria e dados | 4 | Sem trabalho de fundo |

---

### Opção B: Fila de operações idempotentes com outbox local

**Descrição:** SQLite local guarda a rota e uma tabela de operações pendentes. Toda ação do catador grava localmente e enfileira uma operação com identificador UUID gerado no dispositivo. A sincronização é oportunista, disparada por retorno de conectividade, por abertura do app e por tentativa periódica com recuo exponencial. O servidor trata o UUID como chave de idempotência.

**Prós:**
- O registro nunca depende de rede no momento da ação.
- Idempotência por UUID de origem resolve reenvio, que é a principal causa de crédito duplicado.
- A fila é inspecionável, o que torna a depuração viável em campo. É possível mostrar ao catador quantas operações faltam sincronizar.
- Escopo contido: apenas a direção campo para servidor exige fila.

**Contras:**
- Exige disciplina de ordenação na fila, já que operações da mesma parada devem chegar em sequência.
- O estado local pode ficar defasado em relação ao servidor, exigindo política explícita de conflito.

**Avaliação pelos critérios:**
| Critério | Score (1-5) | Justificativa |
|---|---|---|
| Não perder registro | 5 | Persistência local imediata, independente de rede |
| Sem duplicação | 5 | Idempotência por chave de origem |
| Simplicidade | 4 | Padrão bem conhecido, escopo unidirecional |
| Bateria e dados | 4 | Sincronização oportunista, com payload pequeno |

---

### Opção C: Motor de sincronização bidirecional genérico com resolução automática

**Descrição:** Replicação completa do subconjunto de dados relevante, com CRDT ou vetor de versões resolvendo conflitos automaticamente.

**Prós:**
- Resolve qualquer padrão de escrita, inclusive os que não existem hoje.
- Robusto a cenários de edição concorrente.

**Contras:**
- Resolve um problema que este domínio não tem. Não há edição concorrente do mesmo registro.
- Complexidade desproporcional ao volume e à equipe. É a categoria de decisão que consome um trimestre e atrasa o piloto inteiro.
- Depuração difícil justamente quando algo dá errado em campo, que é o momento de maior custo.

**Avaliação pelos critérios:**
| Critério | Score (1-5) | Justificativa |
|---|---|---|
| Não perder registro | 5 | Robusto |
| Sem duplicação | 4 | Depende da correção da implementação, que é sofisticada |
| Simplicidade | 1 | Desproporcional ao problema |
| Bateria e dados | 2 | Replicação contínua consome mais |

---

## Decisão

**Escolhemos: Opção B — fila de operações idempotentes com outbox local.**

A Opção A é eliminada pelo critério principal. A Opção C é eliminada por desproporção: paga-se o custo de resolver edição concorrente num domínio onde ela não ocorre.

O desenho fica assim especificado:

- **Persistência local:** SQLite guarda a rota publicada, incluindo endereços, referências e coordenadas, além da tabela `operacoes_pendentes`.
- **Chave de idempotência:** todo desfecho de parada gera um UUID v4 no dispositivo, enviado ao servidor e persistido junto ao evento de coleta. Reenvio com UUID já conhecido retorna o resultado original sem novo processamento e sem novo crédito de pontos, atendendo à FE-1 da RF-040.
- **Ordenação:** a fila é processada em ordem de criação. Operações da mesma parada são serializadas; falha em uma bloqueia as seguintes daquela parada, não as de outras.
- **Gatilhos de sincronização:** retorno de conectividade, abertura do app, conclusão de parada e tentativa periódica com recuo exponencial limitado a 15 minutos.
- **Resolução de conflito:** o registro de campo prevalece sobre alteração feita no escritório, conforme FE-2 da RF-033. O catador esteve no local; o coordenador não. Toda prevalência é registrada em log de auditoria e sinalizada ao coordenador.
- **Retenção local:** a rota e suas operações permanecem no aparelho por 7 dias após a sincronização, então são removidas. Isso limita a exposição de endereços de terceiros, atendendo à seção 4.3 do PRD.
- **Visibilidade:** o app exibe permanentemente o número de operações pendentes e o horário da última sincronização bem-sucedida.

Fora de escopo deste desenho: sincronização bidirecional genérica, edição offline de dados do servidor e resolução automática de conflitos além da regra única acima.

---

## Consequências

### Positivas
- Registro de pesagem sobrevive a jornada inteira sem rede.
- Crédito duplicado de pontos fica estruturalmente impedido, não apenas improvável.
- A fila visível dá ao catador e ao coordenador um diagnóstico imediato quando algo não chega.
- Retenção de 7 dias limita a janela de exposição de endereços em aparelho perdido ou roubado.

### Negativas / Trade-offs aceitos
- Dados do servidor alterados após a publicação da rota não chegam ao catador até a próxima sincronização. Aceito: alteração de rota em andamento é exceção operacional e deve ser tratada por telefone.
- Nenhuma sincronização em segundo plano agressiva, para preservar bateria. Consequência: o crédito de pontos do morador pode demorar horas. Mitigação: comunicar ao morador que o crédito ocorre após o fechamento da rota, ajustando a expectativa em vez do sistema.

### Riscos e Mitigações
| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Fila travada por operação com erro permanente | Média | Alto | Após 5 tentativas, a operação é isolada em quarentena com alerta ao coordenador; a fila segue |
| Desinstalação do app com fila pendente | Baixa | Alto | Aviso persistente de pendência; orientação operacional de sincronizar antes de fechar a jornada |
| Relógio do dispositivo incorreto afetando ordenação | Média | Médio | Ordenação por sequencial local monotônico, não por timestamp; o servidor carimba o horário de recebimento |

---

## Implementação

**Impacta:**
- [x] Infraestrutura
- [x] Backend
- [x] Frontend
- [x] Banco de dados
- [ ] Integrações externas

**Tarefas necessárias para implementar esta decisão:**
1. Modelar `operacoes_pendentes` no SQLite local, com sequencial monotônico, UUID, payload, tentativas e status.
2. Adicionar coluna de chave de idempotência em `eventos_coleta`, com índice único.
3. Implementar o processador de fila com recuo exponencial e quarentena.
4. Implementar no servidor o tratamento idempotente do endpoint de registro de coleta.
5. Implementar o log de auditoria de prevalência de campo.
6. Implementar a rotina de expurgo local após 7 dias. **Revisto pelo ADR-015:** são duas rotinas com gatilhos distintos, não uma. Esta tarefa está substituída pelas tarefas 2 e 3 do ADR-015 e não deve ser implementada como escrita.
7. Construir cenário de teste com rede intermitente simulada.

**Métricas de validação:**
Taxa de operações em quarentena abaixo de 0,5%. Zero eventos de coleta duplicados em produção. P95 do tempo entre registro em campo e sincronização abaixo de 4h.

