# Constitution — Zona Azul Digital

## 1. Objetivo

Estabelecer as regras globais que devem ser obedecidas durante a implementação da Zona Azul Digital.

A especificação funcional detalhada está definida em `spec.md` e prevalece sobre interpretações ou comportamentos não documentados.

## 2. Fonte de Verdade

* O contrato definido na documentação DEVE ser seguido exatamente.
* Nenhum comportamento adicional DEVE ser inventado quando não estiver especificado.
* Nomes de endpoints, campos, status HTTP e códigos de erro DEVEM respeitar o contrato.
* Em caso de conflito, a regra mais específica do contrato prevalece.

## 3. Variante da Prova

Esta execução utiliza os seguintes parâmetros:

* `TARIFA_HORA_CENTAVOS`: **550**
* `FRACAO_MINUTOS`: **30**
* `TETO_DIARIO_CENTAVOS`: **7000**
* `PORTA_SERVICO`: **8004**
* `TOLERANCIA_MINUTOS`: **0**

A aplicação DEVE utilizar esses valores na execução desta variante.

## 4. Validação

* Dados de entrada inválidos DEVEM ser rejeitados com HTTP `422`.
* Conflitos de estado DEVEM retornar HTTP `409`.
* A validação de formato DEVE ocorrer antes da verificação de conflitos de negócio.
* Recursos inexistentes DEVEM retornar HTTP `404`.
* As respostas de erro DEVEM utilizar exatamente os códigos definidos na especificação.

## 5. Valores Monetários

* Valores monetários DEVEM ser representados exclusivamente em centavos inteiros.
* Valores monetários NÃO DEVEM utilizar ponto flutuante.
* O valor da tarifa por hora DEVE ser `550` centavos.
* O teto diário DEVE ser `7000` centavos.

## 6. Tempo e Arredondamento

* Datas e horários DEVEM utilizar ISO-8601 com informação de fuso horário.
* A referência temporal da aplicação DEVE utilizar o fuso `-03:00`.
* A cobrança DEVE respeitar a fração de `30` minutos.
* Frações incompletas DEVEM ser arredondadas para cima.
* O arredondamento da média do relatório DEVE utilizar 0,5 como ponto de arredondamento para cima.

## 7. Estado dos Bilhetes

Um bilhete pode estar nos estados:

* `aberto`
* `encerrado`
* `cancelado`

As transições de estado DEVEM respeitar as regras definidas em `spec.md`.

Um bilhete encerrado ou cancelado NÃO DEVE voltar ao estado aberto.

## 8. Unicidade por Placa

* Uma placa PODE possuir histórico de múltiplos bilhetes.
* Uma placa NÃO PODE possuir mais de um bilhete aberto simultaneamente.
* Após encerramento ou cancelamento, a placa PODE receber um novo bilhete.

## 9. Persistência e Histórico

* Bilhetes encerrados ou cancelados NÃO DEVEM ser removidos do histórico.
* Consultas de histórico DEVEM considerar todos os estados.
* Listagens DEVEM respeitar a ordenação definida na especificação.

## 10. Relatórios

O relatório diário DEVE considerar somente bilhetes encerrados na data consultada.

Bilhetes abertos e cancelados NÃO DEVEM compor faturamento nem tempo médio.

## 11. Execução

* A aplicação DEVE disponibilizar a API na porta `8004`.
* A API DEVE estar acessível pela base definida pelo contrato da prova.
* A aplicação DEVE ser executável no ambiente de validação automatizada.

## 12. Restrição da Prova

A documentação desta prova DEVE especificar o comportamento esperado, não fornecer a implementação.

Arquivos `.md` NÃO DEVEM conter blocos de código com mais de 20 linhas.

## 13. Critério de Conformidade

A implementação será considerada conforme somente quando respeitar simultaneamente:

1. O contrato funcional de `spec.md`.
2. As regras globais desta constituição.
3. Os cenários definidos em `tests.md`.
4. As decisões técnicas estabelecidas em `plan.md`.
5. Os requisitos de execução e entrega da prova.
