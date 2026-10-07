# Plano de Tarefas — Zona Azul Digital

## 1. Objetivo

Este documento decompõe a implementação da Zona Azul Digital em tarefas executáveis.

Cada tarefa deve possuir um objetivo claro e produzir uma alteração verificável no projeto.

As tarefas devem seguir as regras estabelecidas em:

* `constitution.md`
* `spec.md`
* `plan.md`
* `tests.md`

Em caso de conflito, as regras do contrato e de `spec.md` possuem prioridade.

---

# 2. Ordem de Implementação

As tarefas devem ser executadas preferencialmente na ordem apresentada.

A implementação deve começar pela infraestrutura mínima e pelo modelo de domínio, avançando posteriormente para os casos de uso e, por fim, para a validação completa.

---

# 3. Tarefas

## TASK-001 — Estruturar a aplicação

### Objetivo

Criar a estrutura mínima da aplicação HTTP.

### Deve incluir

* inicialização da aplicação;
* configuração da porta;
* definição da URL base;
* mecanismo para receber requisições HTTP;
* mecanismo para retornar respostas HTTP;
* configuração dos parâmetros da variante.

### Resultado esperado

A aplicação deve iniciar corretamente e estar preparada para responder na `PORTA_SERVICO`.

### Critério de conclusão

A aplicação deve iniciar sem interação manual e escutar na porta definida pela variante.

---

## TASK-002 — Implementar o modelo de bilhete

### Objetivo

Criar a representação interna de um bilhete.

### Deve suportar

* `id`;
* `placa`;
* `entrada`;
* `status`;
* `saida`;
* `minutos`;
* `valor_centavos`.

### Regras

Os campos relacionados ao encerramento devem permanecer ausentes ou não preenchidos enquanto o bilhete estiver aberto.

Bilhetes cancelados não devem possuir cobrança ou horário de saída.

### Critério de conclusão

O modelo deve representar corretamente os estados `aberto`, `encerrado` e `cancelado`.

---

## TASK-003 — Implementar persistência dos bilhetes

### Objetivo

Criar a camada responsável por armazenar e consultar bilhetes.

### Deve permitir

* criar bilhete;
* buscar por `id`;
* buscar por placa;
* buscar bilhete aberto por placa;
* listar bilhetes abertos;
* buscar bilhetes encerrados em uma data;
* preservar histórico.

### Critério de conclusão

Os registros devem permanecer disponíveis durante o ciclo de execução da aplicação e permitir todas as consultas necessárias pelos casos de uso.

---

## TASK-004 — Implementar validação de placa

### Objetivo

Centralizar a validação da placa.

### Regras

A validação deve garantir:

* exatamente 7 caracteres;
* somente caracteres alfanuméricos;
* letras maiúsculas;
* placa obrigatória quando exigida pelo endpoint.

### Critério de conclusão

Entradas inválidas devem resultar no erro:

`placa_invalida`

com HTTP `422`.

---

## TASK-005 — Implementar validação de data e hora

### Objetivo

Centralizar a validação dos valores temporais utilizados pela API.

### Deve validar

* `entrada` em ISO-8601 com timezone;
* `data` no formato `AAAA-MM-DD`;
* timezone `-03:00`.

### Critério de conclusão

Os erros devem utilizar exatamente:

* `entrada_invalida`;
* `data_invalida`.

Ambos com HTTP `422`.

---

## TASK-006 — Implementar UC1 — Abrir bilhete

### Objetivo

Implementar `POST /bilhetes`.

### Deve permitir

* abertura com placa válida;
* abertura com `entrada` explícita;
* abertura utilizando horário atual quando `entrada` não for informada;
* geração de `id`;
* criação do estado `aberto`.

### Deve rejeitar

* placa inválida;
* entrada inválida;
* placa que já possui bilhete aberto.

### Critério de conclusão

O endpoint deve obedecer integralmente aos códigos HTTP e formatos de resposta definidos no `spec.md`.

---

## TASK-007 — Implementar regra de unicidade por placa

### Objetivo

Garantir que uma placa possua no máximo um bilhete aberto.

### Regras

Antes de criar um bilhete, verificar se existe outro bilhete da mesma placa com status `aberto`.

Quando existir:

* HTTP `409`;
* erro `bilhete_em_aberto`.

Após encerramento ou cancelamento, a placa deve poder abrir novamente.

### Critério de conclusão

A regra deve funcionar tanto para novas aberturas quanto para reaberturas após encerramento ou cancelamento.

---

## TASK-008 — Implementar cálculo de duração

### Objetivo

Calcular a duração de estacionamento entre `entrada` e `saida`.

### Deve considerar

* timezone;
* instante de entrada;
* instante de saída;
* duração em minutos.

### Critério de conclusão

O resultado deve ser determinístico quando uma `entrada` explícita for utilizada.

---

## TASK-009 — Implementar cálculo de cobrança

### Objetivo

Criar a regra central de cálculo do valor do estacionamento.

### Deve considerar

* `FRACAO_MINUTOS`;
* `TARIFA_HORA_CENTAVOS`;
* arredondamento para cima;
* duração exata de uma fração;
* duração uma unidade acima da fração;
* valores inteiros em centavos.

### Critério de conclusão

A cobrança deve produzir exatamente os valores definidos pelas regras do contrato.

---

## TASK-010 — Implementar tolerância gratuita

### Objetivo

Adicionar a regra de `TOLERANCIA_MINUTOS`.

### Regras

Quando:

`duração <= tolerância`

o valor deve ser `0`.

Quando:

`duração > tolerância`

a cobrança deve considerar a duração integral.

A tolerância não deve ser descontada da duração.

### Critério de conclusão

Os casos de fronteira `tolerância` e `tolerância + 1` devem produzir comportamentos diferentes conforme definido no contrato.

---

## TASK-011 — Implementar teto diário

### Objetivo

Limitar o valor máximo cobrado.

### Regra

O valor final não pode ultrapassar:

`TETO_DIARIO_CENTAVOS`.

### Critério de conclusão

Qualquer cálculo acima do teto deve retornar exatamente o valor do teto.

---

## TASK-012 — Implementar UC2 — Encerrar bilhete

### Objetivo

Implementar:

`POST /bilhetes/{id}/encerramento`

### Deve

* localizar o bilhete;
* validar existência;
* validar estado;
* gerar `saida`;
* calcular `minutos`;
* calcular `valor_centavos`;
* alterar o status para encerrado;
* preservar o registro no histórico.

### Deve retornar

* `200` em caso de sucesso;
* `404 bilhete_nao_encontrado` quando o bilhete não existir;
* `409 bilhete_ja_encerrado` quando já estiver encerrado.

### Critério de conclusão

O encerramento deve integrar corretamente duração, fração, tolerância e teto.

---

## TASK-013 — Implementar UC5 — Cancelar bilhete

### Objetivo

Implementar:

`POST /bilhetes/{id}/cancelamento`

### Deve

* localizar o bilhete;
* verificar se está aberto;
* alterar status para `cancelado`;
* preservar o histórico.

### Não deve

* gerar cobrança;
* gerar `saida`;
* gerar `valor_centavos`.

### Critério de conclusão

Bilhetes inexistentes devem retornar `404`.

Bilhetes que não estejam abertos devem retornar `409 bilhete_nao_aberto`.

---

## TASK-014 — Implementar UC3 — Listar ativos

### Objetivo

Implementar:

`GET /bilhetes/ativos`

### Deve

* retornar somente bilhetes `aberto`;
* excluir encerrados;
* excluir cancelados;
* ordenar do mais recente para o mais antigo.

### Critério de conclusão

Quando não existirem bilhetes abertos, retornar `200` com array vazio.

---

## TASK-015 — Implementar UC6 — Histórico por placa

### Objetivo

Implementar:

`GET /bilhetes?placa=...`

### Deve

* validar a placa;
* localizar todos os bilhetes da placa;
* incluir qualquer status;
* ordenar do mais recente para o mais antigo;
* retornar array vazio quando não houver histórico.

### Critério de conclusão

O histórico deve preservar registros abertos, encerrados e cancelados.

---

## TASK-016 — Implementar UC4 — Relatório diário

### Objetivo

Implementar:

`GET /relatorios/diario?data=AAAA-MM-DD`

### Deve calcular

* `data`;
* `total_bilhetes`;
* `faturamento_centavos`;
* `tempo_medio_minutos`.

### Regras

Somente bilhetes encerrados no dia devem contribuir para:

* quantidade;
* faturamento;
* tempo médio.

### Critério de conclusão

O relatório deve respeitar o filtro pela data de encerramento.

---

## TASK-017 — Implementar arredondamento da média

### Objetivo

Implementar especificamente a regra de arredondamento do tempo médio.

### Regra

Valores com parte decimal:

* menor que `0,5` → arredondar para baixo;
* exatamente `0,5` → arredondar para cima;
* maior que `0,5` → arredondar para cima.

### Critério de conclusão

O resultado de `tempo_medio_minutos` deve seguir a regra de `0,5 para cima`.

---

## TASK-018 — Implementar tratamento centralizado de erros

### Objetivo

Garantir respostas consistentes para erros de validação e regras de negócio.

### Deve suportar

* `placa_invalida`;
* `entrada_invalida`;
* `data_invalida`;
* `bilhete_nao_encontrado`;
* `bilhete_ja_encerrado`;
* `bilhete_nao_aberto`;
* `bilhete_em_aberto`.

### Critério de conclusão

Cada situação deve retornar exatamente o código HTTP e o corpo definidos no contrato.

---

## TASK-019 — Implementar ordenação determinística

### Objetivo

Garantir ordenação consistente nas consultas de bilhetes.

### Deve aplicar-se a

* bilhetes ativos;
* histórico por placa.

### Regra

Os registros devem aparecer do mais recente para o mais antigo.

### Critério de conclusão

Consultas repetidas sobre o mesmo conjunto de dados devem produzir a mesma ordem.

---

## TASK-020 — Validar prioridade das validações

### Objetivo

Garantir que erros de formato tenham prioridade sobre conflitos de negócio.

### Deve verificar

* placa inválida versus placa já ocupada;
* entrada inválida versus criação de bilhete;
* data inválida no relatório.

### Critério de conclusão

Erros `422` devem ser retornados antes de conflitos `409` quando ambos poderiam ser identificados na mesma requisição.

---

## TASK-021 — Validar transições de estado

### Objetivo

Verificar todas as transições permitidas e proibidas.

### Permitidas

* `aberto → encerrado`;
* `aberto → cancelado`.

### Proibidas

* `encerrado → encerrado`;
* `encerrado → cancelado`;
* `cancelado → encerrado`;
* `cancelado → cancelado`.

### Critério de conclusão

Cada transição inválida deve produzir o erro correspondente definido pelo contrato.

---

## TASK-022 — Validar casos de borda

### Objetivo

Executar os cenários de borda definidos em `tests.md`.

### Deve incluir

* fração exata;
* fração + 1 minuto;
* hora exata;
* hora + 1 minuto;
* tolerância exata;
* tolerância + 1;
* tolerância zero;
* teto exato;
* valor acima do teto;
* lista vazia;
* histórico vazio;
* média exata;
* média terminando em `.5`;
* bilhete inexistente;
* estados inválidos.

### Critério de conclusão

Todos os comportamentos de fronteira devem estar de acordo com `spec.md`.

---

## TASK-023 — Validar contrato HTTP

### Objetivo

Verificar a interface pública completa da API.

### Deve validar

* métodos HTTP;
* caminhos;
* parâmetros;
* corpos;
* códigos HTTP;
* nomes dos campos;
* tipos dos valores;
* mensagens de erro.

### Critério de conclusão

Nenhum endpoint deve retornar estrutura incompatível com o contrato.

---

## TASK-024 — Validar execução na porta da variante

### Objetivo

Garantir que a aplicação funcione utilizando a `PORTA_SERVICO` definida pela variante.

### Critério de conclusão

A aplicação deve estar acessível através de:

`http://localhost:{PORTA_SERVICO}`

e responder aos endpoints definidos.

---

## TASK-025 — Validação final contra o contrato

### Objetivo

Realizar uma verificação final de conformidade.

### Deve verificar

* `constitution.md`;
* `spec.md`;
* `plan.md`;
* `tests.md`;
* todos os casos de uso;
* todas as regras de negócio;
* todos os códigos de erro;
* casos de borda;
* execução da aplicação.

### Critério de conclusão

A implementação deve estar pronta para ser submetida à suíte automatizada sem depender de comportamento não especificado.

---

# 4. Dependências Entre Tarefas

A ordem lógica principal é:

```text id="q2z2b7"
TASK-001
   ↓
TASK-002
   ↓
TASK-003
   ↓
TASK-004 + TASK-005
   ↓
TASK-006 + TASK-007
   ↓
TASK-008
   ↓
TASK-009 + TASK-010 + TASK-011
   ↓
TASK-012 + TASK-013
   ↓
TASK-014 + TASK-015
   ↓
TASK-016 + TASK-017
   ↓
TASK-018 + TASK-019 + TASK-020 + TASK-021
   ↓
TASK-022 + TASK-023 + TASK-024
   ↓
TASK-025
```

As tarefas de validação e tratamento de erros podem ser implementadas progressivamente durante as etapas anteriores, mas devem estar concluídas antes da validação final.

---

# 5. Critérios Gerais de Conclusão

O trabalho será considerado concluído quando:

1. Todos os casos de uso estiverem implementados.
2. Todas as regras de negócio do `spec.md` forem atendidas.
3. Todos os cenários relevantes de `tests.md` forem cobertos.
4. Todos os erros possuírem o código HTTP correto.
5. Todos os valores monetários forem inteiros em centavos.
6. Todas as transições de estado forem controladas.
7. A regra de uma vaga aberta por placa for respeitada.
8. O histórico for preservado.
9. O relatório diário estiver correto.
10. A aplicação executar na porta da variante.
11. A API estiver acessível pela Base URL definida no contrato.
12. Nenhum comportamento obrigatório depender de interpretação não especificada.

---

# 6. Regra de Implementação

As tarefas deste documento descrevem **o trabalho que deve ser realizado**.

Elas NÃO devem ser interpretadas como autorização para alterar o contrato.

Quando houver dúvida durante a implementação:

1. consultar o contrato;
2. consultar `spec.md`;
3. consultar `constitution.md`;
4. consultar `plan.md`;
5. consultar `tests.md`.

A implementação deve sempre priorizar o comportamento especificado pelo contrato.

**Especifique; não implemente.**
