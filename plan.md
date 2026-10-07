# Plano de Implementação — Zona Azul Digital

## 1. Objetivo

Definir a estratégia técnica para implementar a API da Zona Azul Digital de acordo com o contrato funcional estabelecido em `spec.md`.

Este documento descreve as principais decisões arquiteturais, estruturas de dados, regras de implementação e estratégias necessárias para garantir que a implementação seja compatível com os testes automatizados.

O plano NÃO deve alterar ou redefinir regras estabelecidas em `spec.md`.

---

# 2. Princípios de Implementação

A implementação deve seguir os seguintes princípios:

1. Priorizar aderência exata ao contrato.
2. Manter as regras de negócio separadas da camada HTTP.
3. Evitar lógica duplicada entre endpoints.
4. Utilizar valores monetários exclusivamente em centavos inteiros.
5. Centralizar as regras de cálculo de cobrança.
6. Centralizar validações reutilizadas por múltiplos casos de uso.
7. Manter o comportamento determinístico sempre que possível.
8. Não adicionar comportamentos não especificados no contrato.

---

# 3. Arquitetura

A aplicação deve ser organizada em camadas simples:

```text
HTTP/API
   ↓
Validação
   ↓
Serviço de Negócio
   ↓
Persistência
```

## 3.1 Camada HTTP

Responsável por:

* receber requisições;
* interpretar parâmetros;
* validar formato básico da entrada;
* chamar os serviços de negócio;
* transformar resultados em respostas HTTP;
* retornar os códigos HTTP definidos no contrato.

A camada HTTP NÃO deve conter toda a lógica de cobrança ou regras de estado.

---

## 3.2 Camada de Domínio/Serviço

Responsável pelas regras de negócio:

* abertura de bilhete;
* encerramento;
* cancelamento;
* cálculo da cobrança;
* aplicação da tolerância;
* aplicação do teto;
* validação de transições de estado;
* verificação de bilhete aberto por placa;
* geração dos dados do relatório.

Essa camada deve concentrar as regras que precisam ser consistentes independentemente do endpoint utilizado.

---

## 3.3 Camada de Persistência

Responsável por armazenar e recuperar os bilhetes.

A persistência deve permitir:

* localizar bilhete por `id`;
* localizar bilhetes por placa;
* localizar bilhetes abertos por placa;
* listar bilhetes abertos;
* recuperar bilhetes encerrados em determinada data;
* preservar o histórico dos bilhetes.

A escolha da tecnologia de persistência deve priorizar simplicidade e confiabilidade para o escopo da prova.

---

# 4. Modelo de Dados

O conceito central da aplicação será o `Bilhete`.

Estrutura lógica:

```text
Bilhete
├── id
├── placa
├── entrada
├── status
├── saida
├── minutos
└── valor_centavos
```

Os campos `saida`, `minutos` e `valor_centavos` devem ser preenchidos somente quando o bilhete for encerrado.

Bilhetes cancelados devem permanecer armazenados, porém sem cobrança e sem `saida`.

---

# 5. Máquina de Estados

O ciclo de vida do bilhete deve ser tratado como uma máquina de estados.

Estados:

```text
ABERTO
  ├──→ ENCERRADO
  │
  └──→ CANCELADO
```

Não devem existir transições de retorno.

### Decisão técnica

A validação das transições deve ser centralizada em uma regra de domínio.

### Justificativa

Centralizar essa regra evita que cada endpoint implemente sua própria interpretação dos estados e reduz o risco de comportamentos inconsistentes.

---

# 6. Validação

As validações devem ser separadas em:

## 6.1 Validação de formato

Responsável por verificar:

* formato da placa;
* formato de `entrada`;
* formato da data;
* campos obrigatórios.

Essas validações devem ocorrer antes das regras de conflito.

## 6.2 Validação de negócio

Responsável por verificar:

* placa já possui bilhete aberto;
* bilhete existe;
* bilhete está aberto;
* bilhete já foi encerrado;
* transição de estado permitida.

### Decisão técnica

A ordem de validação deve ser:

```text
Entrada da requisição
        ↓
Validação de formato
        ↓
Validação de existência
        ↓
Validação de estado
        ↓
Regra de negócio
        ↓
Persistência
```

### Justificativa

O contrato determina que erros de formato devem possuir prioridade sobre conflitos de estado.

---

# 7. Estratégia de Datas e Horários

Todos os horários devem ser tratados como valores de data/hora com timezone.

O sistema deve trabalhar com o fuso `-03:00`.

A entrada fornecida pelo cliente deve ser preservada como um instante temporal válido.

Quando `entrada` não for informada, deve ser utilizado o instante atual.

No encerramento, `saida` deve representar o instante do encerramento.

A duração deve ser calculada a partir de:

```text
saida - entrada
```

e convertida para minutos conforme definido pelo contrato.

---

# 8. Estratégia de Cobrança

A lógica de cobrança deve existir em uma única função ou serviço de domínio.

Ela deve receber, conceitualmente:

```text
duração
tarifa por hora
tamanho da fração
tolerância
teto diário
```

e produzir:

```text
valor_centavos
```

---

## 8.1 Ordem do cálculo

A ordem deve ser:

```text
Duração
   ↓
Verificar tolerância
   ↓
Calcular quantidade de frações
   ↓
Calcular valor
   ↓
Aplicar teto
   ↓
Retornar centavos inteiros
```

---

## 8.2 Tolerância

Primeiramente deve ser verificado:

```text
duração <= TOLERANCIA_MINUTOS
```

Quando verdadeiro:

```text
valor = 0
```

Quando falso, a cobrança deve ser calculada sobre a duração completa.

A tolerância NÃO deve ser subtraída da duração.

### Justificativa

Essa decisão evita implementar incorretamente a regra como uma franquia descontável.

---

# 9. Cálculo das Frações

A quantidade de frações deve utilizar arredondamento para cima.

Conceitualmente:

```text
frações = ceil(minutos / FRACAO_MINUTOS)
```

O valor de uma fração deve ser obtido a partir da tarifa horária:

```text
valor_fração =
TARIFA_HORA_CENTAVOS / (60 / FRACAO_MINUTOS)
```

O resultado final deve permanecer como inteiro em centavos.

---

# 10. Valores Monetários

Nenhum valor monetário deve ser representado como `float`.

### Decisão técnica

A aplicação deve trabalhar exclusivamente com inteiros em centavos.

Exemplo conceitual:

```text
R$ 12,50
↓
1250 centavos
```

### Justificativa

Isso elimina erros de precisão de ponto flutuante e está diretamente alinhado ao contrato da API.

---

# 11. Aplicação do Teto

Depois do cálculo normal da cobrança, o valor deve ser limitado por:

```text
TETO_DIARIO_CENTAVOS
```

Conceitualmente:

```text
valor_final = min(valor_calculado, TETO_DIARIO_CENTAVOS)
```

A regra deve ser aplicada antes da resposta HTTP.

---

# 12. Unicidade de Bilhete Aberto

Antes de criar um novo bilhete, o sistema deve verificar se existe outro bilhete da mesma placa com:

```text
status = aberto
```

Caso exista, a operação deve retornar:

```text
409
bilhete_em_aberto
```

### Decisão técnica

A verificação deve ficar no serviço responsável pela abertura do bilhete e não somente na camada HTTP.

### Justificativa

A regra pertence ao domínio e precisa ser respeitada independentemente de como a operação for chamada.

---

# 13. Ordenação

As consultas que retornam listas devem utilizar ordenação explícita.

Para:

* `GET /bilhetes/ativos`
* `GET /bilhetes?placa=...`

a ordenação deve ser:

```text
mais recente → mais antigo
```

A ordenação deve ser realizada de maneira determinística utilizando o instante de entrada e, quando necessário, o identificador como critério secundário.

---

# 14. Relatório Diário

O relatório deve utilizar como referência a data de encerramento (`saida`).

O processamento deve:

1. selecionar bilhetes encerrados no dia solicitado;
2. calcular a quantidade;
3. somar `valor_centavos`;
4. calcular a média de `minutos`;
5. arredondar a média conforme a regra do contrato.

Bilhetes abertos e cancelados não devem contribuir para o relatório de faturamento ou tempo médio.

---

# 15. Cálculo da Média

O tempo médio deve ser calculado somente sobre os bilhetes encerrados no dia.

A média deve ser arredondada com a regra de:

```text
0,5 → arredonda para cima
```

### Decisão técnica

A implementação não deve depender de um arredondamento padrão que possa utilizar comportamento de `banker's rounding`.

### Justificativa

O contrato exige explicitamente que valores terminados em `0,5` sejam arredondados para cima.

---

# 16. Endpoints

A API deve implementar os seguintes recursos:

| UC  | Método | Endpoint                             |
| --- | ------ | ------------------------------------ |
| UC1 | POST   | `/bilhetes`                          |
| UC2 | POST   | `/bilhetes/{id}/encerramento`        |
| UC3 | GET    | `/bilhetes/ativos`                   |
| UC4 | GET    | `/relatorios/diario?data=AAAA-MM-DD` |
| UC5 | POST   | `/bilhetes/{id}/cancelamento`        |
| UC6 | GET    | `/bilhetes?placa=ABC1D23`            |

A ordem específica das rotas deve ser configurada de forma que `/bilhetes/ativos` não seja interpretado como uma consulta de histórico por placa ou como um identificador.

---

# 17. Tratamento de Erros

Os erros de domínio devem ser convertidos para as respostas HTTP definidas no contrato.

A camada de negócio deve representar situações como:

```text
bilhete não encontrado
bilhete já encerrado
bilhete não aberto
bilhete em aberto para a placa
```

A camada HTTP deve convertê-las para os respectivos códigos:

```text
404
409
```

Erros de validação devem resultar em:

```text
422
```

O corpo do erro deve utilizar exatamente o campo:

```text
erro
```

---

# 18. Testabilidade

A implementação deve permitir controlar o instante de entrada por meio do campo opcional `entrada`.

Isso permite testar:

* duração exata de uma fração;
* duração uma unidade acima da fração;
* tolerância;
* teto;
* média diária;
* diferentes durações;

sem depender da passagem de tempo real.

Os testes devem preferencialmente utilizar valores de `entrada` determinados explicitamente.

---

# 19. Configuração

Os parâmetros da variante devem ser tratados como configuração da aplicação:

```text
TARIFA_HORA_CENTAVOS
FRACAO_MINUTOS
TETO_DIARIO_CENTAVOS
PORTA_SERVICO
TOLERANCIA_MINUTOS
```

A aplicação deve utilizar os valores definidos para a variante da prova.

A porta HTTP deve ser configurável e a aplicação deve escutar em `PORTA_SERVICO`.

---

# 20. Execução

A aplicação deve:

1. iniciar sem necessidade de interação manual;
2. escutar em `PORTA_SERVICO`;
3. responder às requisições HTTP;
4. funcionar dentro do ambiente de execução fornecido pela correção;
5. não depender de serviços externos que não estejam previstos pelo contrato.

A URL base esperada pela suíte será:

`http://localhost:{PORTA_SERVICO}`

---

# 21. Estratégia de Implementação por Caso de Uso

A implementação deve ser desenvolvida na seguinte ordem:

### Etapa 1 — Infraestrutura básica

* configurar aplicação HTTP;
* configurar porta;
* criar modelo de bilhete;
* implementar persistência;
* implementar tratamento básico de erros.

### Etapa 2 — UC1

* validação de placa;
* validação de `entrada`;
* criação de bilhete;
* geração de `id`;
* prevenção de placa duplicada aberta.

### Etapa 3 — UC2

* localização do bilhete;
* validação de estado;
* cálculo de duração;
* cálculo de frações;
* tolerância;
* teto;
* encerramento.

### Etapa 4 — UC5

* localização do bilhete;
* validação de estado;
* cancelamento;
* preservação do histórico.

### Etapa 5 — UC3 e UC6

* listagem de ativos;
* histórico por placa;
* ordenação.

### Etapa 6 — UC4

* filtro por data de encerramento;
* faturamento;
* quantidade;
* média;
* arredondamento.

### Etapa 7 — Validação final

* verificar todos os códigos HTTP;
* verificar todos os corpos de erro;
* verificar formatos de resposta;
* verificar regras de arredondamento;
* verificar transições de estado;
* verificar casos de borda.

---

# 22. Estratégia de Verificação

Antes da entrega, a implementação deve ser verificada contra cada critério de `spec.md`.

A verificação deve cobrir obrigatoriamente:

* abertura válida;
* abertura inválida;
* entrada personalizada;
* entrada inválida;
* placa duplicada;
* encerramento válido;
* encerramento inexistente;
* encerramento duplicado;
* cobrança por fração;
* tolerância;
* tolerância + 1 minuto;
* teto;
* cancelamento;
* cancelamento de bilhete não aberto;
* ativos;
* histórico;
* relatório diário;
* média com `.5`;
* valores monetários em centavos;
* ordenação;
* funcionamento na porta configurada.

---

# 23. Decisões Técnicas Fundamentais

## DT-01 — Dinheiro em centavos inteiros

**Decisão:** utilizar somente inteiros para valores monetários.

**Justificativa:** elimina problemas de precisão de ponto flutuante e corresponde exatamente ao contrato.

---

## DT-02 — Regra de cobrança centralizada

**Decisão:** concentrar o cálculo de cobrança em uma única unidade de domínio.

**Justificativa:** evita divergência entre diferentes operações e facilita a validação das regras de fração, tolerância e teto.

---

## DT-03 — Validação antes de conflito

**Decisão:** validar formato antes de verificar conflitos de negócio.

**Justificativa:** o contrato determina prioridade de erros `422` sobre conflitos `409`.

---

## DT-04 — Estados explícitos

**Decisão:** representar explicitamente os estados `aberto`, `encerrado` e `cancelado`.

**Justificativa:** simplifica a validação das transições e evita inferir o estado pela presença ou ausência de campos.

---

## DT-05 — Tempo controlável

**Decisão:** aceitar `entrada` fornecida pelo cliente.

**Justificativa:** permite testes determinísticos de duração, cobrança, tolerância e relatório sem depender do relógio real.

---

## DT-06 — Histórico preservado

**Decisão:** não excluir bilhetes após encerramento ou cancelamento.

**Justificativa:** o UC6 exige histórico completo da placa.

---

# 24. Restrições de Implementação

A implementação:

* NÃO DEVE utilizar valores monetários em `float`;
* NÃO DEVE ignorar o timezone;
* NÃO DEVE descontar a tolerância da duração quando houver cobrança;
* NÃO DEVE arredondar frações para baixo;
* NÃO DEVE permitir dois bilhetes abertos para a mesma placa;
* NÃO DEVE excluir bilhetes cancelados do histórico;
* NÃO DEVE considerar bilhetes abertos ou cancelados como encerrados no relatório;
* NÃO DEVE retornar campos incompatíveis com o contrato;
* NÃO DEVE criar endpoints adicionais como parte necessária da solução;
* NÃO DEVE depender de espera em tempo real para testar duração.

---

# 25. Resultado Esperado

Ao final da implementação, a API deve fornecer uma implementação determinística e testável dos oito casos de uso definidos no contrato.

A implementação deve priorizar:

1. compatibilidade com a especificação;
2. comportamento determinístico;
3. regras de negócio centralizadas;
4. valores monetários exatos;
5. tratamento explícito de estados;
6. facilidade de teste;
7. simplicidade arquitetural.

Em caso de conflito entre uma decisão deste plano e uma regra de `spec.md`, a regra de `spec.md` deve prevalecer.
