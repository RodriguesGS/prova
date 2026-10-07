# Spec — Zona Azul Digital

## 1. Objetivo

Especificar o comportamento funcional da API de Zona Azul Digital para a variante desta prova.

A implementação DEVE seguir exatamente os endpoints, campos, regras de negócio, status HTTP e erros definidos neste documento.

## 2. Parâmetros da Variante

| Parâmetro              |    Valor |
| ---------------------- | -------: |
| `TARIFA_HORA_CENTAVOS` |    `550` |
| `FRACAO_MINUTOS`       |     `30` |
| `TETO_DIARIO_CENTAVOS` |   `7000` |
| `PORTA_SERVICO`        |   `8004` |
| `TOLERANCIA_MINUTOS`   |      `0` |
| Fuso horário           | `-03:00` |

## 3. Bilhete

Um bilhete possui:

* `id`
* `placa`
* `entrada`
* `saida`, quando encerrado
* `minutos`, quando encerrado
* `valor_centavos`, quando encerrado
* `status`

Os estados possíveis são `aberto`, `encerrado` e `cancelado`.

Bilhetes encerrados ou cancelados DEVEM permanecer no histórico.

---

## 4. UC1 — Abrir Bilhete

### Endpoint

`POST /bilhetes`

### Entrada

O campo `placa` é obrigatório e DEVE possuir exatamente 7 caracteres alfanuméricos em letras maiúsculas.

O campo `entrada` é opcional e, quando informado, DEVE ser uma data ISO-8601 com fuso horário.

Quando `entrada` não for informado, a aplicação DEVE utilizar o momento atual.

### Sucesso

Retornar HTTP `201` contendo:

* `id`
* `placa`
* `entrada`
* `status: "aberto"`

### Erros

* Placa ausente ou inválida → `422` com `{"erro":"placa_invalida"}`
* Entrada inválida → `422` com `{"erro":"entrada_invalida"}`
* Já existe bilhete aberto para a placa → `409` com `{"erro":"bilhete_em_aberto"}`

### Critérios de aceite

* Uma placa válida cria exatamente um bilhete aberto.
* Uma placa inválida nunca cria bilhete.
* Uma entrada inválida nunca cria bilhete.
* Uma placa com bilhete aberto não pode abrir outro bilhete.

---

## 5. UC2 — Encerrar Bilhete

### Endpoint

`POST /bilhetes/{id}/encerramento`

O bilhete DEVE estar aberto.

### Sucesso

Retornar HTTP `200` contendo:

* `id`
* `placa`
* `entrada`
* `saida`
* `minutos`
* `valor_centavos`

O estado do bilhete passa para `encerrado`.

### Erros

* Bilhete inexistente → `404` com `{"erro":"bilhete_nao_encontrado"}`
* Bilhete já encerrado → `409` com `{"erro":"bilhete_ja_encerrado"}`

### Critérios de aceite

* Um bilhete aberto pode ser encerrado uma única vez.
* O tempo deve corresponder à diferença entre entrada e saída.
* O valor deve ser calculado em centavos inteiros.
* O encerramento deve respeitar a fração, a tolerância e o teto diário.

---

## 6. Regra de Cobrança

A tarifa horária é de `550` centavos.

A fração é de `30` minutos.

Portanto, cada fração de 30 minutos custa `275` centavos.

A quantidade de frações cobradas DEVE ser arredondada para cima.

Exemplos:

|  Tempo | Frações | Valor |
| -----: | ------: | ----: |
| 30 min |       1 |   275 |
| 31 min |       2 |   550 |
| 60 min |       2 |   550 |
| 61 min |       3 |   825 |

O valor final NÃO PODE ultrapassar `7000` centavos por bilhete/dia conforme a regra de teto definida para a variante.

---

## 7. UC7 — Tolerância

A tolerância desta variante é `0` minutos.

Portanto, qualquer duração positiva DEVE ser cobrada normalmente.

A regra geral de tolerância é:

* duração igual ou inferior à tolerância → `valor_centavos = 0`
* duração superior à tolerância → cobrança integral desde o minuto `0`

A tolerância NÃO DEVE ser descontada do tempo cobrado.

### Critérios de aceite

* Com tolerância `0`, a cobrança normal permanece ativa.
* Não deve existir desconto automático de minutos.

---

## 8. UC3 — Listar Bilhetes Ativos

### Endpoint

`GET /bilhetes/ativos`

### Sucesso

Retornar HTTP `200` com um array contendo somente bilhetes `aberto`.

A ordenação DEVE ser do mais novo para o mais antigo.

### Critérios de aceite

* Bilhetes encerrados não aparecem.
* Bilhetes cancelados não aparecem.
* Quando não houver bilhetes abertos, retornar `[]`.

---

## 9. UC4 — Relatório Diário

### Endpoint

`GET /relatorios/diario?data=AAAA-MM-DD`

O parâmetro `data` é obrigatório.

### Sucesso

Retornar HTTP `200` contendo:

* `data`
* `total_bilhetes`
* `faturamento_centavos`
* `tempo_medio_minutos`

O relatório DEVE considerar somente bilhetes encerrados na data informada.

Bilhetes abertos e cancelados DEVEM ser ignorados.

O tempo médio DEVE ser calculado somente entre os bilhetes encerrados naquele dia.

O resultado da média DEVE ser arredondado com 0,5 para cima.

### Erros

* Data ausente ou inválida → `422` com `{"erro":"data_invalida"}`

### Critérios de aceite

* O faturamento considera somente encerramentos da data consultada.
* Bilhetes abertos não entram no relatório.
* Bilhetes cancelados não entram no relatório.
* Sem encerramentos, o relatório deve retornar valores zerados para os agregados.

---

## 10. UC5 — Cancelar Bilhete

### Endpoint

`POST /bilhetes/{id}/cancelamento`

Somente bilhetes `aberto` podem ser cancelados.

### Sucesso

Retornar HTTP `200` contendo:

* `id`
* `placa`
* `entrada`
* `status: "cancelado"`

O retorno NÃO DEVE possuir `saida` ou `valor_centavos`.

### Erros

* Bilhete inexistente → `404` com `{"erro":"bilhete_nao_encontrado"}`
* Bilhete encerrado ou cancelado → `409` com `{"erro":"bilhete_nao_aberto"}`

### Critérios de aceite

* Um bilhete aberto pode ser cancelado.
* Um bilhete cancelado não pode ser cancelado novamente.
* Um bilhete encerrado não pode ser cancelado.
* Cancelamento não gera cobrança.

---

## 11. UC6 — Histórico por Placa

### Endpoint

`GET /bilhetes?placa=...`

A placa é obrigatória e deve seguir a mesma validação do UC1.

### Sucesso

Retornar HTTP `200` com todos os bilhetes associados à placa, independentemente do estado.

A ordenação DEVE ser do mais novo para o mais antigo.

### Erros

Placa ausente ou inválida → `422` com `{"erro":"placa_invalida"}`

### Critérios de aceite

* O histórico inclui bilhetes abertos, encerrados e cancelados.
* A ordem é do mais novo para o mais antigo.
* Uma placa sem histórico retorna `[]`.

---

## 12. Unicidade de Bilhete Aberto

Uma placa pode possuir vários registros históricos, mas somente um bilhete `aberto` simultaneamente.

Após encerramento ou cancelamento, a mesma placa pode receber um novo bilhete.

### Critérios de aceite

* Segundo bilhete aberto para a mesma placa → `409` `bilhete_em_aberto`.
* Após encerramento, nova abertura é permitida.
* Após cancelamento, nova abertura é permitida.

---

## 13. Validação e Prioridade dos Erros

A validação de formato DEVE ocorrer antes das validações de estado.

Exemplo: uma requisição com placa inválida não pode retornar `409` por já existir um bilhete aberto.

A ordem geral é:

1. Validar formato e dados obrigatórios.
2. Verificar existência do recurso.
3. Verificar conflito de estado.
4. Executar a operação.

---

## 14. Regras de Consistência

* `valor_centavos` DEVE ser inteiro.
* Valores monetários NÃO DEVEM utilizar ponto flutuante.
* Datas e horários DEVEM manter informação de fuso horário.
* Bilhetes encerrados DEVEM possuir `saida`, `minutos` e `valor_centavos`.
* Bilhetes cancelados NÃO DEVEM possuir `saida` nem `valor_centavos`.
* Bilhetes abertos não possuem informações de encerramento.
* O histórico não pode perder bilhetes após encerramento ou cancelamento.

---

## 15. Critérios Globais de Aceite

A implementação será aceita quando:

1. Todos os endpoints definidos neste documento estiverem disponíveis.
2. Os status HTTP forem exatamente os especificados.
3. Os códigos de erro forem exatamente os especificados.
4. A cobrança utilizar `550` centavos por hora e frações de 30 minutos.
5. O teto de `7000` centavos for respeitado.
6. A tolerância de `0` minuto for respeitada.
7. Apenas uma sessão aberta existir por placa.
8. Histórico e ordenação funcionarem conforme especificado.
9. O relatório diário considerar somente encerramentos.
10. A aplicação estiver disponível na porta `8004`.
11. Todos os valores monetários forem representados em centavos inteiros.
12. A implementação respeitar as regras de validação e transição de estados.
