# Especificação Zona Azul Digital

## 1. Objetivo

Implementar uma API HTTP para gerenciamento de bilhetes de estacionamento da Zona Azul Digital.

A implementação DEVE obedecer exatamente aos casos de uso, regras de negócio, formatos de resposta, códigos HTTP e regras de validação definidos neste documento.

A API DEVE estar disponível em:

`http://localhost:{PORTA_SERVICO}`

A implementação NÃO DEVE criar comportamentos, endpoints ou regras que não estejam especificados neste documento.

---

# 2. Configuração da Variante

Os seguintes parâmetros fazem parte da configuração da variante da prova:

* `TARIFA_HORA_CENTAVOS`: valor da hora cheia em centavos.
* `FRACAO_MINUTOS`: duração de cada fração de cobrança.
* `TETO_DIARIO_CENTAVOS`: valor máximo cobrado por bilhete no dia.
* `PORTA_SERVICO`: porta HTTP da aplicação.
* `TOLERANCIA_MINUTOS`: quantidade inicial de minutos gratuitos.

Valores efetivos:

* `TARIFA_HORA_CENTAVOS`: entre `400` e `600`, em incrementos de `50`.
* `FRACAO_MINUTOS`: `30` ou `15`.
* `TETO_DIARIO_CENTAVOS`: entre `5000` e `8000`, em incrementos de `1000`.
* `PORTA_SERVICO`: entre `8001` e `8005`.
* `TOLERANCIA_MINUTOS`: `0`, `10` ou `15`.

A aplicação DEVE escutar em `PORTA_SERVICO`.

O ambiente da aplicação DEVE utilizar o fuso horário `-03:00`.

---

# 3. Modelo de Bilhete

Cada bilhete DEVE possuir, no mínimo:

* `id`
* `placa`
* `entrada`
* `status`

Bilhetes encerrados DEVEM possuir também:

* `saida`
* `minutos`
* `valor_centavos`

Bilhetes cancelados NÃO DEVEM possuir:

* `saida`
* `valor_centavos`

Os valores monetários DEVEM ser representados exclusivamente como inteiros em centavos.

A API NÃO DEVE retornar valores monetários em ponto flutuante.

Os estados possíveis de um bilhete são:

* `aberto`
* `encerrado`
* `cancelado`

---

# 4. Regras Gerais de Validação

## 4.1 Placa

Uma placa válida DEVE:

* possuir exatamente 7 caracteres;
* conter somente caracteres alfanuméricos;
* estar em letras maiúsculas.

Exemplo válido:

`ABC1D23`

Placa ausente ou inválida DEVE resultar em:

**HTTP 422**

```json
{"erro": "placa_invalida"}
```

A validação da placa DEVE ocorrer antes da validação de conflitos de estado.

---

## 4.2 Datas e horários

Valores de data e hora DEVEM utilizar ISO-8601 com indicação explícita do fuso horário `-03:00`.

Uma `entrada` inválida DEVE resultar em:

**HTTP 422**

```json
{"erro": "entrada_invalida"}
```

A ausência de `entrada` NÃO é erro.

Quando `entrada` não for informada, o sistema DEVE utilizar o instante atual.

---

## 4.3 Data do relatório

O parâmetro `data` DEVE estar no formato:

`AAAA-MM-DD`

Data ausente ou inválida DEVE resultar em:

**HTTP 422**

```json
{"erro": "data_invalida"}
```

---

## 4.4 Prioridade das validações

Validações de formato DEVEM ocorrer antes das validações de conflito de estado.

Exemplo:

Uma placa inválida que também possui um bilhete aberto DEVE retornar:

**422 `placa_invalida`**

e NÃO:

**409 `bilhete_em_aberto`**.

---

# 5. UC1 — Abrir Bilhete

## 5.1 Endpoint

`POST /bilhetes`

## 5.2 Entrada

O corpo DEVE aceitar:

```json
{
  "placa": "ABC1D23"
}
```

Também DEVE aceitar:

```json
{
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00"
}
```

`entrada` é opcional.

---

## 5.3 Regras

Ao abrir um bilhete:

1. A placa DEVE ser validada.
2. Caso exista bilhete aberto para a mesma placa, a operação DEVE ser rejeitada.
3. Caso `entrada` tenha sido informada, ela DEVE ser validada.
4. Caso `entrada` não tenha sido informada, o sistema DEVE utilizar o instante atual.
5. O novo bilhete DEVE iniciar com `status: "aberto"`.
6. O sistema DEVE gerar um identificador único para o bilhete.

---

## 5.4 Sucesso

Em caso de sucesso, retornar:

**HTTP 201**

```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00",
  "status": "aberto"
}
```

---

## 5.5 Erros

Placa inválida:

**422**

```json
{"erro": "placa_invalida"}
```

Entrada inválida:

**422**

```json
{"erro": "entrada_invalida"}
```

Placa já possui bilhete aberto:

**409**

```json
{"erro": "bilhete_em_aberto"}
```

---

## 5.6 Critérios de aceitação

* Uma placa válida sem bilhete aberto DEVE gerar um novo bilhete.
* O novo bilhete DEVE possuir status `aberto`.
* A resposta DEVE possuir HTTP 201.
* Uma placa inválida DEVE resultar em HTTP 422.
* Uma `entrada` inválida DEVE resultar em HTTP 422.
* Uma placa com bilhete aberto DEVE resultar em HTTP 409.
* Após encerramento ou cancelamento, a mesma placa DEVE poder abrir outro bilhete.

---

# 6. UC2 — Encerrar Bilhete

## 6.1 Endpoint

`POST /bilhetes/{id}/encerramento`

## 6.2 Regras

O sistema DEVE:

1. localizar o bilhete pelo `id`;
2. verificar se o bilhete existe;
3. verificar se ainda está aberto;
4. determinar o instante de saída;
5. calcular a duração em minutos;
6. calcular o valor devido;
7. aplicar a tolerância;
8. aplicar o teto diário;
9. alterar o status para `encerrado`.

---

## 6.3 Sucesso

Em caso de sucesso:

**HTTP 200**

```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00",
  "saida": "2026-10-05T11:35:00-03:00",
  "minutos": 95,
  "valor_centavos": 1250
}
```

---

## 6.4 Bilhete inexistente

Quando o `id` não existir:

**HTTP 404**

```json
{"erro": "bilhete_nao_encontrado"}
```

---

## 6.5 Bilhete já encerrado

Quando o bilhete já estiver encerrado:

**HTTP 409**

```json
{"erro": "bilhete_ja_encerrado"}
```

---

# 7. Regra de Cálculo da Cobrança

A cobrança DEVE ser realizada em frações de `FRACAO_MINUTOS`.

A quantidade de frações DEVE ser arredondada sempre para cima.

Matematicamente:

`frações = ceil(minutos / FRACAO_MINUTOS)`

O valor de cada fração DEVE ser calculado por:

`valor_fração = TARIFA_HORA_CENTAVOS / (60 / FRACAO_MINUTOS)`

O valor final DEVE ser:

`valor = frações × valor_fração`

O resultado DEVE permanecer em centavos inteiros.

---

## 7.1 Regra de arredondamento da fração

Uma duração exatamente igual ao tamanho da fração DEVE cobrar somente uma fração.

Exemplo para `FRACAO_MINUTOS = 30`:

* 30 minutos → 1 fração
* 31 minutos → 2 frações
* 60 minutos → 2 frações
* 61 minutos → 3 frações

A implementação NÃO DEVE arredondar a duração para baixo.

---

# 8. UC7 — Tolerância Gratuita

A quantidade de minutos gratuitos é definida por:

`TOLERANCIA_MINUTOS`

Quando:

`minutos <= TOLERANCIA_MINUTOS`

o valor DEVE ser:

`valor_centavos = 0`

Quando:

`minutos > TOLERANCIA_MINUTOS`

a cobrança DEVE ser realizada integralmente desde o primeiro minuto.

A tolerância NÃO DEVE ser descontada da duração antes do cálculo.

---

## 8.1 Exemplos

Para `TOLERANCIA_MINUTOS = 10`:

* 0 minutos → R$ 0,00
* 5 minutos → R$ 0,00
* 10 minutos → R$ 0,00
* 11 minutos → cobrança integral de 11 minutos

A diferença entre 10 e 11 minutos DEVE ser observável no valor cobrado.

---

# 9. Teto Diário

O valor final cobrado por um bilhete NÃO PODE ultrapassar:

`TETO_DIARIO_CENTAVOS`

Portanto:

`valor_centavos <= TETO_DIARIO_CENTAVOS`

Caso o cálculo da cobrança ultrapasse o teto, o valor retornado DEVE ser limitado ao teto.

---

# 10. UC3 — Listar Bilhetes Ativos

## 10.1 Endpoint

`GET /bilhetes/ativos`

## 10.2 Comportamento

A resposta DEVE conter somente bilhetes cujo status seja:

`aberto`

Bilhetes encerrados ou cancelados NÃO DEVEM aparecer.

Os bilhetes DEVEM ser retornados do mais recente para o mais antigo.

---

## 10.3 Sucesso

**HTTP 200**

A resposta DEVE ser um array.

Quando não houver bilhetes ativos, o sistema DEVE retornar um array vazio.

---

## 10.4 Critérios de aceitação

* Bilhetes abertos aparecem na resposta.
* Bilhetes encerrados não aparecem.
* Bilhetes cancelados não aparecem.
* A ordenação DEVE ser do mais recente para o mais antigo.
* Nenhum bilhete ativo deve aparecer duplicado.

---

# 11. UC4 — Relatório Diário

## 11.1 Endpoint

`GET /relatorios/diario?data=AAAA-MM-DD`

## 11.2 Entrada

O parâmetro `data` é obrigatório.

Quando ausente ou inválido:

**HTTP 422**

```json
{"erro": "data_invalida"}
```

---

## 11.3 Resposta

Em caso de sucesso:

**HTTP 200**

```json
{
  "data": "2026-10-05",
  "total_bilhetes": 12,
  "faturamento_centavos": 8400,
  "tempo_medio_minutos": 47
}
```

---

## 11.4 Total de bilhetes

`total_bilhetes` DEVE representar a quantidade de bilhetes encerrados no dia consultado.

Bilhetes abertos ou cancelados NÃO DEVEM contribuir para o faturamento ou tempo médio do relatório.

---

## 11.5 Faturamento

`faturamento_centavos` DEVE ser a soma dos valores dos bilhetes encerrados no dia.

O valor DEVE ser representado em centavos inteiros.

---

## 11.6 Tempo médio

`tempo_medio_minutos` DEVE considerar somente bilhetes encerrados no dia consultado.

A média DEVE ser arredondada utilizando a regra:

**0,5 para cima.**

Exemplo:

* média 46,4 → 46
* média 46,5 → 47
* média 46,6 → 47

---

# 12. UC5 — Cancelar Bilhete

## 12.1 Endpoint

`POST /bilhetes/{id}/cancelamento`

## 12.2 Regras

Somente bilhetes com status `aberto` podem ser cancelados.

O cancelamento:

* NÃO gera cobrança;
* NÃO gera `saida`;
* NÃO gera `valor_centavos`;
* altera o status para `cancelado`.

---

## 12.3 Sucesso

**HTTP 200**

A resposta DEVE possuir:

```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00",
  "status": "cancelado"
}
```

---

## 12.4 Erros

Bilhete inexistente:

**404**

```json
{"erro": "bilhete_nao_encontrado"}
```

Bilhete não está aberto:

**409**

```json
{"erro": "bilhete_nao_aberto"}
```

---

# 13. UC6 — Histórico por Placa

## 13.1 Endpoint

`GET /bilhetes?placa=ABC1D23`

## 13.2 Regras

O parâmetro `placa` é obrigatório.

A placa DEVE obedecer às mesmas regras de validação utilizadas no UC1.

A resposta DEVE conter todos os bilhetes associados à placa, independentemente do status:

* `aberto`
* `encerrado`
* `cancelado`

Os registros DEVEM ser retornados do mais recente para o mais antigo.

---

## 13.3 Placa sem histórico

Quando a placa for válida, mas nunca possuir bilhetes:

**HTTP 200**

```json
[]
```

---

## 13.4 Placa inválida

**HTTP 422**

```json
{"erro": "placa_invalida"}
```

---

# 14. UC8 — Uma Vaga por Placa

O sistema DEVE permitir no máximo um bilhete `aberto` para cada placa.

Ao tentar abrir um novo bilhete para uma placa que já possui um bilhete aberto:

**HTTP 409**

```json
{"erro": "bilhete_em_aberto"}
```

Após o bilhete ser:

* encerrado; ou
* cancelado;

a placa DEVE poder receber um novo bilhete.

A regra de unicidade NÃO DEVE impedir o histórico de múltiplos bilhetes da mesma placa.

---

# 15. Estados e Transições

As transições permitidas são:

`aberto → encerrado`

`aberto → cancelado`

As seguintes operações NÃO DEVEM ser permitidas:

`encerrado → encerrado`

`encerrado → cancelado`

`cancelado → cancelado`

`cancelado → encerrado`

Uma tentativa de encerramento de bilhete já encerrado DEVE retornar:

**409 `bilhete_ja_encerrado`**

Uma tentativa de cancelamento de qualquer bilhete que não esteja aberto DEVE retornar:

**409 `bilhete_nao_aberto`**

---

# 16. Ordenação

Sempre que uma lista de bilhetes for retornada:

* os registros mais recentes DEVEM aparecer primeiro;
* a ordenação DEVE ser determinística.

Essa regra se aplica ao:

* UC3 — bilhetes ativos;
* UC6 — histórico por placa.

---

# 17. Persistência e Identificação

Cada bilhete DEVE possuir um `id` único.

O `id` DEVE permanecer associado ao bilhete durante todo o seu ciclo de vida.

O encerramento ou cancelamento NÃO DEVE criar um novo bilhete.

O histórico de uma placa DEVE preservar os registros anteriores.

---

# 18. Regras de Erro

A API DEVE utilizar exatamente os códigos HTTP e identificadores de erro definidos abaixo:

| Situação                       | HTTP | Erro                     |
| ------------------------------ | ---: | ------------------------ |
| Placa ausente ou inválida      |  422 | `placa_invalida`         |
| Entrada inválida               |  422 | `entrada_invalida`       |
| Data inválida                  |  422 | `data_invalida`          |
| Bilhete inexistente            |  404 | `bilhete_nao_encontrado` |
| Encerrar bilhete já encerrado  |  409 | `bilhete_ja_encerrado`   |
| Cancelar bilhete não aberto    |  409 | `bilhete_nao_aberto`     |
| Abrir placa com bilhete aberto |  409 | `bilhete_em_aberto`      |

A API NÃO DEVE substituir os identificadores acima por mensagens diferentes quando a situação correspondente ocorrer.

---

# 19. Critérios Globais de Aceitação

A implementação será considerada aderente quando:

1. Todos os endpoints definidos neste documento estiverem disponíveis.
2. Cada endpoint retornar os códigos HTTP especificados.
3. Os nomes dos campos das respostas forem exatamente os especificados.
4. Os erros utilizarem exatamente os identificadores definidos.
5. Valores monetários forem sempre inteiros em centavos.
6. A cobrança respeitar `FRACAO_MINUTOS`.
7. A cobrança utilizar arredondamento para cima das frações.
8. O teto diário nunca for ultrapassado.
9. A tolerância respeitar a regra de gratuidade e cobrança integral.
10. Apenas um bilhete aberto existir por placa.
11. O histórico preservar bilhetes de qualquer status.
12. Os bilhetes das listas forem ordenados do mais recente para o mais antigo.
13. O relatório diário considerar somente bilhetes encerrados no dia.
14. O tempo médio utilizar arredondamento de 0,5 para cima.
15. Bilhetes cancelados não gerarem cobrança.
16. Datas e horários utilizarem ISO-8601 com fuso `-03:00`.
17. A aplicação executar na `PORTA_SERVICO` definida pela variante.
18. A implementação funcionar dentro do ambiente de execução fornecido pela prova.

---

# 20. Casos de Borda Obrigatórios

A implementação DEVE tratar corretamente, no mínimo:

### Validação

* placa ausente;
* placa com menos de 7 caracteres;
* placa com mais de 7 caracteres;
* placa contendo caracteres inválidos;
* placa em minúsculas;
* `entrada` inválida;
* `data` inválida;
* data ausente.

### Frações

* duração exatamente igual a uma fração;
* duração uma unidade acima de uma fração;
* duração exatamente igual a uma hora;
* duração uma unidade acima de uma hora.

### Tolerância

* duração zero;
* duração exatamente igual à tolerância;
* duração uma unidade acima da tolerância;
* tolerância igual a zero.

### Estados

* encerramento de bilhete inexistente;
* encerramento de bilhete já encerrado;
* cancelamento de bilhete inexistente;
* cancelamento de bilhete encerrado;
* cancelamento de bilhete já cancelado;
* abertura de placa com bilhete aberto;
* abertura da mesma placa após encerramento;
* abertura da mesma placa após cancelamento.

### Consultas

* lista de ativos vazia;
* histórico de placa sem registros;
* histórico contendo bilhetes de diferentes estados.

### Relatório

* relatório sem bilhetes encerrados;
* relatório com um único bilhete;
* média com resultado inteiro;
* média terminando em `.5`;
* bilhetes abertos no dia;
* bilhetes cancelados no dia.

---

# 21. Observação sobre Exemplos Antigos

Qualquer exemplo externo ou exemplo de documentação anterior que contradiga este contrato DEVE ser ignorado.

Em particular, respostas contendo campos como:

```json
{
  "valor": 12.50
}
```

NÃO representam o contrato atual.

O campo correto é:

`valor_centavos`

e DEVE ser um inteiro.

O contrato definido neste documento possui prioridade sobre exemplos antigos, ilustrativos ou conflitantes.
