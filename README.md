# Separa aí

> **In English:** Separa aí is a municipal recycling platform built solo, in TypeScript end
> to end. It is made of a resident app, an offline-first waste-picker app, a web dashboard
> for the cooperative and the city, and an API with a background worker. Residents find
> drop-off points, learn how to sort waste, schedule home pickups and earn points. Points
> are credited only against a weight recorded in the field. Businesses are settled in money
> on the same field operation. Built spec-first with GitHub Spec Kit, under a written
> constitution, a canonical data model and OpenAPI contracts enforced by a CI gate.

Plataforma municipal de coleta seletiva para cidades de pequeno e médio porte. Liga o
morador, a cooperativa de catadores, as empresas geradoras e a prefeitura numa mesma
operação de campo.

## Status

_Atualizado em 10/11/2026._

**Em piloto.** A plataforma roda numa VM própria, com publicação por imagem versionada. Das
17 features planejadas, 14 estão entregues, 1 está em fechamento e 2 ainda não começaram.

| # | Feature | Estado |
|---|---|---|
| 000 | Fundação: monorepo, schema base, CI e publicação | Entregue |
| 001 | Conta e identidade: cadastro, verificação por SMS, sessão, endereços | Entregue |
| 002 | Guia de separação, offline, com curadoria | Entregue |
| 003 | Mapa de PEVs, offline, com report do morador e QR rotativo | Entregue |
| 004 | Solicitação de coleta e triagem | Entregue |
| 005 | Rota e execução offline do catador, com sincronização idempotente | Entregue |
| 006 | Pontuação: crédito por pesagem, check-in em PEV, extrato | Entregue |
| 007 | Recompensas: parceiros, catálogo, resgate e cupom | Não iniciada |
| 008 | Empresa e homologação | Entregue |
| 009 | Coleta recorrente da empresa | Entregue |
| 010 | Liquidação financeira e comprovante em PDF | Entregue |
| 011 | Indicadores do município, com exportação em CSV | Entregue |
| 012 | Armazenamento local do app do catador, com expurgo | Entregue |
| 013 | Exclusão de conta com anonimização | Não iniciada |
| 014 | Painel da cooperativa | Entregue |
| 015 | App do morador: conta e coleta | Entregue |
| 016 | Identidade visual "Separa aí" nos três clientes | Em fechamento: faltam os APKs de release e a publicação |

**Pendências conhecidas do piloto**

- O SMS de verificação ainda passa por um operador: falta escolher o provedor.
- Os achados abertos (bugs, lacunas e débitos) ficam num registro próprio, com evidência
  em `arquivo:linha`.

**Qualidade:** cerca de 1.850 testes automatizados (Vitest), que rodam contra PostgreSQL e
Redis reais. Um portão de CI recusa qualquer mudança que fuja do contrato OpenAPI, do modelo
de dados canônico ou das fronteiras entre pacotes.

## O que tem aqui

| Componente | Pasta | Tecnologia |
|---|---|---|
| API e worker | apps/api | Fastify 5, PostgreSQL 16 + PostGIS, BullMQ sobre Redis |
| App do morador | apps/morador | Expo / React Native, mapa e guia offline |
| App do catador | apps/catador | Expo / React Native, offline-first com fila em SQLite |
| Painel web | apps/painel | React + Vite, MapLibre com mapa self-hosted (PMTiles) |
| Domínio e contratos | packages/core | Tipos gerados do OpenAPI |
| Banco | packages/db | Migrações com node-pg-migrate |
| Marca | packages/marca | Fonte única de cores, tipografia e símbolo |
| Portão de CI | tools/portao | Contratos, ERD, escopo, fronteiras e testes |

## Como o projeto é feito

O projeto segue especificação antes de código, com o
[GitHub Spec Kit](https://github.com/github/spec-kit). Cada feature tem spec, plano, tarefas e
roteiro de validação. Elas obedecem a artefatos que vieram antes de todas:

- Constitution: nove princípios inegociáveis. Por exemplo,
  o registro de campo é imutável, a idempotência vem do identificador de origem, ponto e
  dinheiro nunca compartilham estrutura e regra crítica vive no banco.
- PRD: requisitos (RF) e regras de negócio (RN) numerados.
- ADRs: 23 decisões de arquitetura, com o raciocínio de cada uma. Dois estão publicados
  aqui: [ADR-003, offline-first no app do catador](docs/adr/003-offline-first-catador.md) e
  [ADR-004, pontuação como livro-razão](docs/adr/004-livro-razao-pontos.md).
- C4: contexto e contêineres. [Publicado aqui](docs/c4.md).
- ERD: o modelo de dados canônico.
- Contratos OpenAPI: um por módulo, do M0 ao M6.

## Código

O código está disponível para avaliação técnica mediante solicitação. O desenvolvimento
acontece num repositório privado, com o histórico completo de commits, specs e testes. Este
repositório é a vitrine: traz o README e uma amostra da documentação de arquitetura. Para
pedir acesso ou uma demonstração, use o contato do meu perfil.
