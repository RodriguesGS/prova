# Tests — Zona Azul Digital

## 1. Objetivo

Definir os cenários necessários para validar a implementação conforme `constitution.md` e `spec.md`.

Os testes DEVEM verificar comportamento observável da API, sem depender de detalhes internos da implementação.

## 2. Dados da Variante

| Parâmetro       |         Valor |
| --------------- | ------------: |
| Tarifa por hora |  550 centavos |
| Fração          |    30 minutos |
| Valor da fração |  275 centavos |
| Teto diário     | 7000 centavos |
| Tolerância      |     0 minutos |
| Porta           |          8004 |

## 3. Abertura de Bilhete — UC1

| ID  | Cenário                          | Resultado esperado               |
| --- | -------------------------------- | -------------------------------- |
| T01 | Placa válida                     | `201` e bilhete `aberto`         |
| T02 | Placa ausente                    | `422` `placa_invalida`           |
| T03 | Placa com tamanho diferente de 7 | `422` `placa_invalida`           |
| T04 | Placa com caracteres inválidos   | `422` `placa_invalida`           |
| T05 | Placa em minúsculas              | `422` `placa_invalida`           |
| T06 | Entrada inválida                 | `422` `entrada_invalida`         |
| T07 | Entrada omitida                  | `201` utilizando o momento atual |

## 4. Unicidade por Placa — UC8

| ID  | Cenário                                  | Resultado esperado        |
| --- | ---------------------------------------- | ------------------------- |
| T08 | Abrir segunda sessão com placa já aberta | `409` `bilhete_em_aberto` |
| T09 | Reabrir placa após encerramento          | `201`                     |
| T10 | Reabrir placa após cancelamento          | `201`                     |

## 5. Encerramento — UC2

| ID  | Cenário                       | Resultado esperado             |
| --- | ----------------------------- | ------------------------------ |
| T11 | Encerrar bilhete aberto       | `200` e estado `encerrado`     |
| T12 | Encerrar bilhete inexistente  | `404` `bilhete_nao_encontrado` |
| T13 | Encerrar bilhete já encerrado | `409` `bilhete_ja_encerrado`   |

## 6. Cobrança por Fração

Com fração de 30 minutos e tarifa de 550 centavos:

| ID  | Duração | Frações | Valor |
| --- | ------: | ------: | ----: |
| T14 |  30 min |       1 |   275 |
| T15 |  31 min |       2 |   550 |
| T16 |  60 min |       2 |   550 |
| T17 |  61 min |       3 |   825 |

A cobrança DEVE utilizar arredondamento para cima e valores inteiros em centavos.

## 7. Tolerância — UC7

A variante possui tolerância de `0` minutos.

| ID  | Cenário           | Resultado esperado     |
| --- | ----------------- | ---------------------- |
| T18 | Duração de 0 min  | valor `0`              |
| T19 | Duração de 1 min  | cobrança integral      |
| T20 | Duração de 30 min | cobrança de uma fração |

## 8. Teto Diário

| ID  | Cenário                       | Resultado esperado |
| --- | ----------------------------- | ------------------ |
| T21 | Valor abaixo de 7000          | valor calculado    |
| T22 | Valor exatamente 7000         | `7000`             |
| T23 | Valor calculado acima de 7000 | limitado a `7000`  |

## 9. Bilhetes Ativos — UC3

| ID  | Cenário                      | Resultado esperado         |
| --- | ---------------------------- | -------------------------- |
| T24 | Existem bilhetes abertos     | Retorna somente os abertos |
| T25 | Existem encerrados e abertos | Encerrados não aparecem    |
| T26 | Existem cancelados e abertos | Cancelados não aparecem    |
| T27 | Nenhum bilhete aberto        | Retorna `[]`               |
| T28 | Vários bilhetes abertos      | Mais novo primeiro         |

## 10. Cancelamento — UC5

| ID  | Cenário                       | Resultado esperado             |
| --- | ----------------------------- | ------------------------------ |
| T29 | Cancelar bilhete aberto       | `200` e estado `cancelado`     |
| T30 | Cancelar bilhete inexistente  | `404` `bilhete_nao_encontrado` |
| T31 | Cancelar bilhete encerrado    | `409` `bilhete_nao_aberto`     |
| T32 | Cancelar bilhete já cancelado | `409` `bilhete_nao_aberto`     |

O bilhete cancelado não deve possuir `saida` ou `valor_centavos`.

## 11. Histórico por Placa — UC6

| ID  | Cenário                       | Resultado esperado       |
| --- | ----------------------------- | ------------------------ |
| T33 | Placa com histórico           | Retorna todos os estados |
| T34 | Placa sem histórico           | Retorna `[]`             |
| T35 | Placa inválida                | `422` `placa_invalida`   |
| T36 | Histórico com vários bilhetes | Mais novo primeiro       |

## 12. Relatório Diário — UC4

| ID  | Cenário                          | Resultado esperado          |
| --- | -------------------------------- | --------------------------- |
| T37 | Existem encerramentos na data    | Calcula total e faturamento |
| T38 | Existem bilhetes abertos na data | Não entram no relatório     |
| T39 | Existem bilhetes cancelados      | Não entram no relatório     |
| T40 | Nenhum encerramento              | Agregados zerados           |
| T41 | Data ausente                     | `422` `data_invalida`       |
| T42 | Data inválida                    | `422` `data_invalida`       |

## 13. Média do Relatório

A média deve considerar somente bilhetes encerrados na data consultada.

| ID  | Cenário              | Resultado esperado   |
| --- | -------------------- | -------------------- |
| T43 | Média inteira        | Mantém o valor       |
| T44 | Média abaixo de 0,5  | Arredonda para baixo |
| T45 | Média exatamente 0,5 | Arredonda para cima  |
| T46 | Média acima de 0,5   | Arredonda para cima  |

## 14. Prioridade de Validação

| ID  | Cenário                             | Resultado esperado       |
| --- | ----------------------------------- | ------------------------ |
| T47 | Placa inválida com placa já aberta  | `422` `placa_invalida`   |
| T48 | Entrada inválida em nova abertura   | `422` `entrada_invalida` |
| T49 | Bilhete inexistente no encerramento | `404`                    |
| T50 | Bilhete inexistente no cancelamento | `404`                    |

Dados inválidos DEVEM ser rejeitados antes de conflitos de estado.

## 15. Consistência dos Estados

| ID  | Cenário                    | Resultado esperado            |
| --- | -------------------------- | ----------------------------- |
| T51 | Encerrar bilhete aberto    | Estado passa para `encerrado` |
| T52 | Cancelar bilhete aberto    | Estado passa para `cancelado` |
| T53 | Encerrar bilhete cancelado | Operação rejeitada            |
| T54 | Cancelar bilhete encerrado | Operação rejeitada            |
| T55 | Encerrar duas vezes        | Segunda operação rejeitada    |
| T56 | Cancelar duas vezes        | Segunda operação rejeitada    |

## 16. Critérios Finais

A implementação deve passar pelos cenários acima e também respeitar:

* valores monetários exclusivamente em centavos inteiros;
* datas em ISO-8601 com fuso horário;
* ordenação do mais novo para o mais antigo;
* preservação do histórico;
* somente um bilhete aberto por placa;
* execução na porta `8004`.

Os testes DEVEM validar o contrato externo da API e NÃO DEVEM exigir uma implementação interna específica.
