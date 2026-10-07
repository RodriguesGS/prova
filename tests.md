# Plano de Testes — Zona Azul Digital

## 1. Objetivo

Este documento define os cenários que devem ser utilizados para verificar a conformidade da implementação com o contrato da Zona Azul Digital.

Os testes DEVEM validar comportamento observável da API por meio de requisições HTTP e suas respectivas respostas.

Este documento especifica os testes, mas NÃO implementa os testes.

---

# 2. Princípios de Teste

Os testes devem:

* verificar os códigos HTTP;
* verificar os campos obrigatórios das respostas;
* verificar os valores retornados;
* verificar as regras de negócio;
* verificar as transições de estado;
* verificar casos de borda;
* utilizar `entrada` explícita quando for necessário controlar a duração;
* evitar dependência desnecessária do relógio real;
* verificar exatamente os identificadores de erro definidos no contrato.

Os testes NÃO devem depender de valores monetários em ponto flutuante.

---

# 3. Dados de Teste

Os cenários devem utilizar placas válidas no formato:

`ABC1D23`

Sempre que a duração precisar ser controlada, o campo `entrada` deve ser fornecido explicitamente.

Exemplo de referência:

`2026-10-05T10:00:00-03:00`

Os valores de tarifa, fração, teto, tolerância e porta devem ser obtidos da variante da prova.

Os testes NÃO devem assumir valores diferentes dos parâmetros efetivamente fornecidos pela variante.

---

# 4. UC1 — Abrir Bilhete

## TEST-UC1-01 — Abrir bilhete com placa válida

**Dado:** não existe bilhete aberto para a placa.

**Quando:** enviar `POST /bilhetes` com uma placa válida.

**Então:**

* HTTP deve ser `201`;
* resposta deve conter `id`;
* resposta deve conter a placa informada;
* resposta deve conter `entrada`;
* `entrada` deve possuir timezone `-03:00`;
* `status` deve ser `aberto`.

---

## TEST-UC1-02 — Abrir bilhete com entrada explícita

**Dado:** uma placa válida sem bilhete aberto.

**Quando:** enviar uma `entrada` válida no corpo.

**Então:**

* HTTP deve ser `201`;
* o valor de `entrada` retornado deve representar o instante informado;
* o status deve ser `aberto`.

Este teste deve comprovar que a aplicação não substitui uma `entrada` válida pelo horário atual.

---

## TEST-UC1-03 — Placa ausente

**Quando:** enviar requisição sem `placa`.

**Então:**

* HTTP deve ser `422`;
* body deve ser `{"erro": "placa_invalida"}`.

---

## TEST-UC1-04 — Placa com tamanho inválido

Devem ser testadas placas:

* com menos de 7 caracteres;
* com mais de 7 caracteres.

**Resultado esperado:** HTTP `422` com `placa_invalida`.

---

## TEST-UC1-05 — Placa com caracteres inválidos

**Quando:** enviar placa contendo caracteres que não sejam alfanuméricos.

**Então:** HTTP `422` com `placa_invalida`.

---

## TEST-UC1-06 — Placa em minúsculas

**Quando:** enviar uma placa que contenha letras minúsculas.

**Então:** HTTP `422` com `placa_invalida`.

---

## TEST-UC1-07 — Entrada inválida

**Quando:** enviar `entrada` que não represente um ISO-8601 válido com timezone.

**Então:**

* HTTP deve ser `422`;
* body deve ser `{"erro": "entrada_invalida"}`.

---

# 5. UC8 — Uma Vaga por Placa

## TEST-UC8-01 — Impedir segundo bilhete aberto

**Dado:**

* existe um bilhete aberto para uma placa.

**Quando:** tentar abrir outro bilhete para a mesma placa.

**Então:**

* HTTP deve ser `409`;
* body deve ser `{"erro": "bilhete_em_aberto"}`;
* o segundo bilhete não deve ser criado.

---

## TEST-UC8-02 — Permitir reabertura após encerramento

**Dado:**

* existe um bilhete aberto;
* o bilhete é encerrado.

**Quando:** abrir novamente um bilhete para a mesma placa.

**Então:**

* HTTP deve ser `201`;
* um novo `id` deve ser criado;
* o novo bilhete deve estar `aberto`.

---

## TEST-UC8-03 — Permitir reabertura após cancelamento

**Dado:**

* existe um bilhete aberto;
* o bilhete é cancelado.

**Quando:** abrir novamente um bilhete para a mesma placa.

**Então:**

* HTTP deve ser `201`;
* um novo bilhete deve ser criado;
* o novo bilhete deve estar `aberto`.

---

# 6. UC2 — Encerrar Bilhete

## TEST-UC2-01 — Encerrar bilhete existente

**Dado:** existe um bilhete aberto.

**Quando:** solicitar encerramento.

**Então:**

* HTTP deve ser `200`;
* o status lógico do bilhete deve passar para `encerrado`;
* resposta deve conter `id`;
* resposta deve conter `placa`;
* resposta deve conter `entrada`;
* resposta deve conter `saida`;
* resposta deve conter `minutos`;
* resposta deve conter `valor_centavos`.

---

## TEST-UC2-02 — Encerrar bilhete inexistente

**Quando:** solicitar encerramento de um `id` inexistente.

**Então:**

* HTTP deve ser `404`;
* body deve ser `{"erro": "bilhete_nao_encontrado"}`.

---

## TEST-UC2-03 — Encerrar bilhete já encerrado

**Dado:** um bilhete já foi encerrado.

**Quando:** solicitar novo encerramento.

**Então:**

* HTTP deve ser `409`;
* body deve ser `{"erro": "bilhete_ja_encerrado"}`.

---

# 7. Testes de Fração

## TEST-FRAC-01 — Duração exatamente igual à fração

**Dado:** `FRACAO_MINUTOS = F`.

**Quando:** encerrar um bilhete após exatamente `F` minutos.

**Então:** deve ser cobrada exatamente uma fração.

---

## TEST-FRAC-02 — Um minuto acima da fração

**Dado:** `FRACAO_MINUTOS = F`.

**Quando:** encerrar após `F + 1` minutos.

**Então:** devem ser cobradas duas frações.

---

## TEST-FRAC-03 — Duração exatamente igual a uma hora

**Dado:** duração de 60 minutos.

**Então:** o valor cobrado deve corresponder exatamente a uma `TARIFA_HORA_CENTAVOS`.

---

## TEST-FRAC-04 — Um minuto acima de uma hora

**Dado:** duração de 61 minutos.

**Então:** a cobrança deve considerar a fração seguinte, conforme `FRACAO_MINUTOS`.

---

## TEST-FRAC-05 — Valor em centavos

Para qualquer encerramento:

* `valor_centavos` deve existir quando houver cobrança;
* o valor deve ser inteiro;
* a API não deve retornar valor monetário como ponto flutuante.

---

# 8. UC7 — Tolerância

## TEST-TOL-01 — Duração abaixo da tolerância

**Dado:** `TOLERANCIA_MINUTOS = T`.

**Quando:** duração for menor que `T`.

**Então:** `valor_centavos` deve ser `0`.

---

## TEST-TOL-02 — Duração exatamente igual à tolerância

**Dado:** `TOLERANCIA_MINUTOS = T`.

**Quando:** duração for exatamente `T`.

**Então:** `valor_centavos` deve ser `0`.

---

## TEST-TOL-03 — Um minuto acima da tolerância

**Dado:** `TOLERANCIA_MINUTOS = T`.

**Quando:** duração for `T + 1`.

**Então:**

* deve existir cobrança;
* a cobrança deve considerar a duração integral;
* os `T` minutos de tolerância NÃO devem ser descontados.

Este é um teste obrigatório de fronteira.

---

## TEST-TOL-04 — Tolerância igual a zero

**Dado:** `TOLERANCIA_MINUTOS = 0`.

**Quando:** houver duração positiva.

**Então:** a cobrança deve seguir normalmente a regra de frações.

---

# 9. Testes de Teto Diário

## TEST-TETO-01 — Valor abaixo do teto

**Quando:** o valor calculado for menor que `TETO_DIARIO_CENTAVOS`.

**Então:** o valor calculado deve ser preservado.

---

## TEST-TETO-02 — Valor exatamente igual ao teto

**Quando:** o valor calculado atingir exatamente o teto.

**Então:** `valor_centavos` deve ser exatamente igual a `TETO_DIARIO_CENTAVOS`.

---

## TEST-TETO-03 — Valor acima do teto

**Quando:** o cálculo normal ultrapassar `TETO_DIARIO_CENTAVOS`.

**Então:** `valor_centavos` deve ser limitado exatamente ao teto.

---

# 10. UC3 — Listar Ativos

## TEST-UC3-01 — Listar bilhetes abertos

**Dado:** existem vários bilhetes abertos.

**Quando:** solicitar `GET /bilhetes/ativos`.

**Então:**

* HTTP deve ser `200`;
* somente bilhetes `aberto` devem aparecer;
* os mais recentes devem aparecer primeiro.

---

## TEST-UC3-02 — Excluir encerrados da lista de ativos

**Dado:** existem bilhetes abertos e encerrados.

**Quando:** consultar os ativos.

**Então:** bilhetes encerrados não devem aparecer.

---

## TEST-UC3-03 — Excluir cancelados da lista de ativos

**Dado:** existem bilhetes abertos e cancelados.

**Quando:** consultar os ativos.

**Então:** bilhetes cancelados não devem aparecer.

---

## TEST-UC3-04 — Nenhum ativo

**Dado:** não existem bilhetes abertos.

**Quando:** consultar os ativos.

**Então:**

* HTTP deve ser `200`;
* resposta deve ser um array vazio.

---

# 11. UC5 — Cancelar Bilhete

## TEST-UC5-01 — Cancelar bilhete aberto

**Dado:** existe um bilhete aberto.

**Quando:** solicitar cancelamento.

**Então:**

* HTTP deve ser `200`;
* status deve ser `cancelado`;
* não deve existir `saida`;
* não deve existir `valor_centavos`.

---

## TEST-UC5-02 — Cancelar bilhete inexistente

**Quando:** solicitar cancelamento de `id` inexistente.

**Então:** HTTP `404` com `bilhete_nao_encontrado`.

---

## TEST-UC5-03 — Cancelar bilhete encerrado

**Dado:** o bilhete já foi encerrado.

**Quando:** solicitar cancelamento.

**Então:** HTTP `409` com `bilhete_nao_aberto`.

---

## TEST-UC5-04 — Cancelar bilhete já cancelado

**Dado:** o bilhete já foi cancelado.

**Quando:** solicitar novo cancelamento.

**Então:** HTTP `409` com `bilhete_nao_aberto`.

---

# 12. UC6 — Histórico por Placa

## TEST-UC6-01 — Histórico completo

**Dado:** uma placa possui:

* um bilhete encerrado;
* um bilhete cancelado;
* um bilhete atualmente aberto.

**Quando:** consultar o histórico da placa.

**Então:**

* HTTP deve ser `200`;
* os três bilhetes devem aparecer;
* os registros devem estar ordenados do mais recente para o mais antigo.

---

## TEST-UC6-02 — Placa sem histórico

**Quando:** consultar uma placa válida que nunca estacionou.

**Então:**

* HTTP deve ser `200`;
* resposta deve ser `[]`.

---

## TEST-UC6-03 — Placa inválida

**Quando:** consultar histórico sem placa ou com placa inválida.

**Então:** HTTP `422` com `placa_invalida`.

---

# 13. UC4 — Relatório Diário

## TEST-UC4-01 — Relatório com bilhetes encerrados

**Dado:** existem bilhetes encerrados na data consultada.

**Quando:** solicitar o relatório.

**Então:**

* HTTP deve ser `200`;
* `data` deve corresponder à data solicitada;
* `total_bilhetes` deve representar os bilhetes encerrados no dia;
* `faturamento_centavos` deve representar a soma dos valores;
* `tempo_medio_minutos` deve representar a média das durações.

---

## TEST-UC4-02 — Bilhete aberto não entra no relatório

**Dado:** existe bilhete aberto na data consultada.

**Então:** esse bilhete não deve contribuir para o total, faturamento ou tempo médio.

---

## TEST-UC4-03 — Bilhete cancelado não entra no relatório

**Dado:** existe bilhete cancelado na data consultada.

**Então:** esse bilhete não deve contribuir para o total, faturamento ou tempo médio.

---

## TEST-UC4-04 — Nenhum encerramento no dia

**Dado:** não existem bilhetes encerrados na data consultada.

**Então:**

* HTTP deve ser `200`;
* `total_bilhetes` deve ser `0`;
* `faturamento_centavos` deve ser `0`;
* o comportamento de `tempo_medio_minutos` deve permanecer consistente com o contrato da API.

---

## TEST-UC4-05 — Data ausente

**Quando:** consultar o relatório sem o parâmetro `data`.

**Então:** HTTP `422` com `data_invalida`.

---

## TEST-UC4-06 — Data inválida

**Quando:** utilizar uma data fora do formato `AAAA-MM-DD`.

**Então:** HTTP `422` com `data_invalida`.

---

# 14. Testes de Média

## TEST-MEDIA-01 — Média inteira

**Dado:** durações cuja média seja um número inteiro.

**Então:** `tempo_medio_minutos` deve ser exatamente esse número.

---

## TEST-MEDIA-02 — Média abaixo de 0,5

**Dado:** durações cuja média possua parte decimal menor que `0,5`.

**Então:** a média deve ser arredondada para baixo.

---

## TEST-MEDIA-03 — Média exatamente em 0,5

**Dado:** durações cuja média termine exatamente em `.5`.

**Então:** a média deve ser arredondada para cima.

---

## TEST-MEDIA-04 — Média acima de 0,5

**Dado:** durações cuja média possua parte decimal maior que `0,5`.

**Então:** a média deve ser arredondada para cima.

---

# 15. Testes de Estado

## TEST-STATE-01 — Aberto para encerrado

A transição deve ser permitida.

---

## TEST-STATE-02 — Aberto para cancelado

A transição deve ser permitida.

---

## TEST-STATE-03 — Encerrado para encerrado

A transição deve ser rejeitada com `409 bilhete_ja_encerrado`.

---

## TEST-STATE-04 — Encerrado para cancelado

A transição deve ser rejeitada com `409 bilhete_nao_aberto`.

---

## TEST-STATE-05 — Cancelado para cancelado

A transição deve ser rejeitada com `409 bilhete_nao_aberto`.

---

## TEST-STATE-06 — Cancelado para encerrado

A operação deve ser rejeitada, pois o bilhete não está aberto.

---

# 16. Prioridade de Validação

## TEST-VALID-01 — Formato inválido possui prioridade sobre conflito

**Dado:** uma placa que possui bilhete aberto.

**Quando:** enviar uma placa em formato inválido.

**Então:** a resposta deve ser `422 placa_invalida`, e não `409 bilhete_em_aberto`.

---

## TEST-VALID-02 — Entrada inválida não deve criar bilhete

**Quando:** enviar uma placa válida com `entrada` inválida.

**Então:**

* HTTP deve ser `422`;
* nenhum bilhete deve ser criado.

---

# 17. Consistência do Histórico

## TEST-HIST-01 — Encerramento preserva histórico

Após encerrar um bilhete, ele deve continuar aparecendo no histórico da respectiva placa com status encerrado.

---

## TEST-HIST-02 — Cancelamento preserva histórico

Após cancelar um bilhete, ele deve continuar aparecendo no histórico da respectiva placa com status cancelado.

---

## TEST-HIST-03 — Novo estacionamento cria novo histórico

Após encerrar ou cancelar um bilhete, uma nova abertura para a mesma placa deve criar um novo registro sem alterar os registros anteriores.

---

# 18. Critérios de Aceitação Globais

A implementação será considerada conforme quando todos os seguintes pontos forem atendidos:

* todos os endpoints definidos no contrato responderem corretamente;
* os códigos HTTP forem exatamente os especificados;
* os identificadores de erro forem exatamente os especificados;
* os campos das respostas respeitarem o contrato;
* valores monetários forem inteiros em centavos;
* frações forem arredondadas para cima;
* tolerância respeitar a regra de cobrança integral após o limite;
* teto nunca for ultrapassado;
* somente uma vaga aberta existir por placa;
* cancelamentos não gerarem cobrança;
* histórico preservar todos os estados;
* ativos exibirem somente bilhetes abertos;
* relatórios considerarem somente encerramentos do dia;
* médias utilizarem arredondamento de `0,5` para cima;
* listas forem ordenadas do mais recente para o mais antigo;
* datas e horários respeitarem ISO-8601 com timezone `-03:00`.

---

# 19. Regra de Ouro dos Testes

Os testes devem verificar **comportamento**, não detalhes internos da implementação.

Não deve ser exigida uma tecnologia, classe, função, banco de dados ou estrutura interna específica quando isso não estiver definido pelo contrato.

A implementação é livre para escolher sua estrutura interna desde que o comportamento observável seja compatível com `spec.md`.

**Especifique; não implemente.**

Este documento descreve o que deve ser comprovado pela suíte de testes. A implementação dos testes deve permanecer fora dos arquivos de especificação.
