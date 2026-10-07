# Tasks — Zona Azul Digital

## 1. Objetivo

Decompor a implementação da Zona Azul Digital em tarefas pequenas e verificáveis, seguindo `constitution.md`, `spec.md`, `plan.md` e `tests.md`.

## 2. Tarefas

### TASK-01 — Estrutura e configuração

Criar a estrutura da aplicação, configuração da API e inicialização na porta `8004`.

**Concluído quando:**

* A aplicação inicia corretamente.
* A API responde na porta `8004`.

### TASK-02 — Modelo e persistência

Implementar o armazenamento dos bilhetes e seus campos obrigatórios.

**Concluído quando:**

* Bilhetes podem ser armazenados.
* Bilhetes encerrados e cancelados permanecem no histórico.

### TASK-03 — Validação de entradas

Implementar as validações de placa, entrada e data.

**Concluído quando:**

* Entradas inválidas retornam `422`.
* Os códigos de erro correspondem ao contrato.
* Validações acontecem antes dos conflitos de negócio.

### TASK-04 — Abertura de bilhete

Implementar `POST /bilhetes`.

**Concluído quando:**

* Placas válidas criam bilhetes abertos.
* `entrada` pode ser informada ou gerada automaticamente.
* A resposta segue o contrato do UC1.

### TASK-05 — Unicidade por placa

Garantir somente um bilhete aberto por placa.

**Concluído quando:**

* Segunda abertura para placa ocupada retorna `409`.
* A placa pode ser reutilizada após encerramento.
* A placa pode ser reutilizada após cancelamento.

### TASK-06 — Regra de cobrança

Implementar o cálculo de duração e cobrança.

**Concluído quando:**

* A tarifa utilizada é `550` centavos por hora.
* A fração utilizada é de `30` minutos.
* Frações incompletas são arredondadas para cima.
* O teto de `7000` centavos é respeitado.
* Valores são tratados como inteiros.

### TASK-07 — Encerramento

Implementar `POST /bilhetes/{id}/encerramento`.

**Concluído quando:**

* Bilhetes abertos podem ser encerrados.
* Bilhetes inexistentes retornam `404`.
* Bilhetes já encerrados retornam `409`.
* A resposta contém os campos definidos no contrato.

### TASK-08 — Cancelamento

Implementar `POST /bilhetes/{id}/cancelamento`.

**Concluído quando:**

* Somente bilhetes abertos podem ser cancelados.
* Bilhetes inexistentes retornam `404`.
* Bilhetes não abertos retornam `409`.
* Cancelamento não gera cobrança.

### TASK-09 — Consultas

Implementar:

* `GET /bilhetes/ativos`
* `GET /bilhetes?placa=...`

**Concluído quando:**

* Ativos retornam somente bilhetes abertos.
* Histórico retorna todos os estados.
* Resultados são ordenados do mais novo para o mais antigo.
* Placas sem histórico retornam `[]`.

### TASK-10 — Relatório diário

Implementar `GET /relatorios/diario?data=AAAA-MM-DD`.

**Concluído quando:**

* Somente encerramentos da data são considerados.
* Faturamento é calculado em centavos.
* Tempo médio considera somente bilhetes encerrados.
* Média é arredondada com 0,5 para cima.
* Data inválida retorna `422`.

### TASK-11 — Testes e casos-limite

Validar os cenários definidos em `tests.md`.

**Concluído quando:**

* Todas as regras de cobrança são verificadas.
* Tolerância é verificada.
* Teto é verificado.
* Estados inválidos são rejeitados.
* Erros e status HTTP correspondem ao contrato.

### TASK-12 — Validação final do contrato

Executar uma revisão final de toda a API.

**Concluído quando:**

* Todos os endpoints existem.
* Todos os campos possuem os nomes esperados.
* Todos os status HTTP estão corretos.
* Os códigos de erro estão corretos.
* A aplicação executa na porta `8004`.
* Não existem comportamentos fora do contrato.

## 3. Dependências

A ordem recomendada é:

`TASK-01 → TASK-02 → TASK-03 → TASK-04 → TASK-05 → TASK-06 → TASK-07/TASK-08 → TASK-09 → TASK-10 → TASK-11 → TASK-12`

TASK-07 e TASK-08 podem ser desenvolvidas em paralelo após a conclusão das regras de cobrança e do modelo de estados.

## 4. Critério Geral de Conclusão

O trabalho estará concluído quando todas as tarefas forem atendidas e a implementação estiver em conformidade com `constitution.md`, `spec.md`, `plan.md` e `tests.md`.
