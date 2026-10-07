# Plan — Zona Azul Digital

## 1. Objetivo

Definir a estratégia técnica para implementar a API de Zona Azul Digital conforme `constitution.md` e `spec.md`.

Este documento descreve decisões de implementação, sem fornecer código da solução.

## 2. Arquitetura

A aplicação DEVE ser organizada em responsabilidades separadas:

* **API:** recebe requisições, valida entradas e produz respostas HTTP.
* **Domínio:** contém regras de bilhetes, estados e cobrança.
* **Persistência:** mantém os bilhetes e seu histórico.
* **Relatórios:** calcula os agregados do relatório diário.

A regra de cobrança DEVE ficar centralizada para evitar cálculos diferentes entre endpoints.

## 3. Modelo de Dados

Cada bilhete DEVE armazenar, no mínimo:

* identificador;
* placa;
* entrada;
* saída, quando encerrado;
* duração em minutos, quando encerrado;
* valor em centavos, quando encerrado;
* estado atual.

O histórico NÃO DEVE ser apagado após encerramento ou cancelamento.

## 4. Estados

O domínio DEVE controlar explicitamente os estados:

`aberto` → `encerrado`

`aberto` → `cancelado`

Não devem existir outras transições.

A regra de unicidade DEVE impedir mais de um bilhete aberto para a mesma placa.

## 5. Validação

As validações devem ocorrer antes das regras de conflito.

Ordem esperada:

1. Validar parâmetros e corpo.
2. Verificar existência do bilhete.
3. Verificar estado atual.
4. Executar a operação.

As respostas devem utilizar exatamente os códigos definidos em `spec.md`.

## 6. Datas e Horários

As datas DEVEM ser tratadas com informação de fuso horário.

O fuso da variante é `-03:00`.

A duração de um bilhete DEVE ser calculada a partir da diferença entre `entrada` e `saida`.

O mecanismo de tempo deve permitir determinar a saída de forma controlada nos testes.

## 7. Estratégia de Cobrança

A cobrança DEVE utilizar somente números inteiros em centavos.

Parâmetros da variante:

* tarifa horária: `550` centavos;
* fração: `30` minutos;
* valor da fração: `275` centavos;
* teto: `7000` centavos;
* tolerância: `0` minutos.

A quantidade de frações deve ser arredondada para cima.

A tolerância deve ser aplicada antes da cobrança.

O teto deve ser aplicado ao valor calculado.

## 8. Ordenação

Consultas que retornam bilhetes DEVEM ordenar os resultados do mais novo para o mais antigo.

A ordenação deve utilizar uma informação temporal consistente.

## 9. Relatório Diário

O relatório deve:

1. Selecionar somente bilhetes encerrados na data solicitada.
2. Somar seus valores em centavos.
3. Calcular a média das durações.
4. Arredondar a média com 0,5 para cima.
5. Retornar zero quando não houver encerramentos.

Bilhetes abertos e cancelados devem ser ignorados.

## 10. Tratamento de Erros

Os erros devem ser centralizados para manter respostas consistentes.

Os códigos definidos no contrato são:

* `422` para dados inválidos;
* `404` para recurso inexistente;
* `409` para conflitos de estado ou unicidade.

O corpo de erro deve utilizar exatamente o campo `erro` e os códigos definidos na especificação.

## 11. Configuração

A aplicação deve iniciar utilizando a porta:

`8004`

A configuração deve permitir que a porta e os parâmetros da aplicação sejam controlados sem alterar as regras de domínio.

## 12. Testabilidade

As regras de negócio devem ser isoladas da camada HTTP sempre que possível.

A solução deve permitir testar separadamente:

* validação;
* transições de estado;
* cobrança;
* tolerância;
* teto;
* ordenação;
* histórico;
* relatório.

## 13. Ordem de Implementação

A implementação deve seguir esta ordem:

1. Estrutura da aplicação e configuração.
2. Modelo e persistência de bilhetes.
3. Validações.
4. Abertura de bilhetes.
5. Regra de unicidade por placa.
6. Cálculo de duração e cobrança.
7. Encerramento.
8. Cancelamento.
9. Listagem de ativos.
10. Histórico por placa.
11. Relatório diário.
12. Validação final de todos os critérios do `spec.md`.

## 14. Decisões Técnicas

### DT-01 — Valores em centavos

Valores monetários serão armazenados como inteiros.

**Motivo:** evita erros de precisão de ponto flutuante e corresponde diretamente ao contrato.

### DT-02 — Cobrança centralizada

O cálculo da cobrança será realizado por uma única regra de domínio.

**Motivo:** garante o mesmo comportamento em qualquer operação que necessite calcular valores.

### DT-03 — Estados explícitos

O estado do bilhete será controlado explicitamente.

**Motivo:** evita transições inválidas e simplifica as regras de encerramento e cancelamento.

### DT-04 — Validação antes de conflito

Dados inválidos serão rejeitados antes da verificação de conflitos.

**Motivo:** garante o comportamento definido pelo contrato.

### DT-05 — Histórico permanente

Bilhetes encerrados ou cancelados permanecerão armazenados.

**Motivo:** o histórico por placa e o relatório dependem desses registros.

## 15. Critério de Conclusão

A implementação estará tecnicamente concluída quando:

* todos os endpoints do `spec.md` estiverem implementados;
* as decisões deste plano forem respeitadas;
* os cenários de `tests.md` forem atendidos;
* a aplicação executar na porta `8004`;
* não houver comportamento divergente do contrato.
