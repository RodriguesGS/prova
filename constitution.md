# Constitution — Zona Azul Digital

## 1. Propósito

Esse documento estabele regras globais para o repositório que devem ser estritamente obedecidas por toda a implantação do sistema. Sempre utilizar esse documento antes de considerar qualquer decisão de implementação

---

## 2. Princípios Fundamentais

### 2.1 Contrato

O contrato da aplicação é a fonte de verdade para o comportamento externo do sistema.

A implementação:

* Deve respeitar exatamente os endpoints definidos;
* Deve respeitar os métodos HTTP definidos;
* Deve respeitar os códigos HTTP definidos;
* Deve respeitar os nomes dos campos definidos;
* Deve respeitar os formatos de entrada e saída definidos;
* Não Deve substituir campos ou códigos de erro por equivalentes;
* Não Deve adicionar comportamentos que alterem o contrato.

### 2.2 Simplicidade

A implementação deve utilizar a solução mais simples que satisfaça integralmente o contrato.

Abstrações, dependências e componentes adicionais somente devem ser introduzidos quando houver justificativa técnica clara.

---

## 3. Separação de Responsabilidades

A implementação deve manter separação entre:

* camada responsável pela comunicação HTTP;
* validação das entradas;
* regras de negócio;
* persistência e recuperação dos dados.

A camada HTTP NÃO DEVE concentrar regras complexas de negócio.

Cálculos de cobrança, controle de estados, tolerância, arredondamento e teto diário DEVEM permanecer isolados da lógica específica de transporte HTTP.

A persistência NÃO DEVE definir regras de negócio.

---

## 4. Compatibilidade da API

### 4.1 Contrato HTTP

Todos os endpoints DEVEM utilizar exatamente os métodos e caminhos definidos na especificação.

Os nomes dos campos de requisição e resposta DEVEM ser preservados exatamente como especificados.

Campos adicionais não devem ser adicionados às respostas quando não estiverem previstos pelo contrato.

### 4.2 Códigos HTTP

Os códigos HTTP definidos pelo contrato são obrigatórios.

A implementação NÃO DEVE substituir um código especificado por outro semanticamente semelhante.

Exemplos:

* sucesso na criação → `201`;
* sucesso na operação → `200`;
* recurso inexistente → `404`;
* conflito de estado → `409`;
* entrada inválida → `422`.

---

## 5. Ordem de Validação

A validação estrutural e de formato DEVE ocorrer antes da avaliação das regras de negócio.

Uma entrada inválida NÃO DEVE chegar às regras de conflito ou alteração de estado.

Quando uma requisição apresentar simultaneamente:

1. erro de formato; e
2. conflito de regra de negócio;

o erro de formato DEVE prevalecer e resultar em `422`.

Um payload inválido NÃO DEVE produzir alteração no estado do sistema.

---

## 6. Validação de Dados

### 6.1 Placa

A placa DEVE obedecer exatamente ao formato definido pelo contrato:

* 7 caracteres;
* caracteres alfanuméricos;
* letras maiúsculas.

Uma placa ausente ou inválida DEVE resultar em `422` com o erro correspondente definido no contrato.

### 6.2 Datas e horários

Datas e horários fornecidos pela API DEVEM utilizar ISO-8601 com informação de fuso horário.

O fuso horário operacional definido para o projeto é `-03:00`.

Quando uma data/hora de entrada for fornecida explicitamente, ela DEVE ser utilizada para os cálculos e regras de negócio correspondentes.

Quando o horário de entrada não for fornecido, o sistema DEVE utilizar o horário atual.

### 6.3 Data de relatório

Datas utilizadas para consulta do relatório diário DEVEM obedecer ao formato `AAAA-MM-DD`.

Datas fora desse formato DEVEM resultar em `422`.

---

## 7. Representação Monetária

Valores monetários DEVEM ser representados exclusivamente em centavos.

Os valores monetários:

* DEVEM ser inteiros;
* NÃO DEVEM utilizar ponto flutuante;
* NÃO DEVEM ser retornados como valores decimais;
* DEVEM utilizar o campo `valor_centavos` quando especificado pelo contrato.

A implementação NÃO DEVE depender de operações de ponto flutuante para determinar valores de cobrança.

A representação monetária deve evitar erros de precisão associados a operações como `0.1 + 0.2`.

---

## 8. Regras de Tempo e Arredondamento

A cobrança DEVE utilizar a granularidade definida pela variante do projeto.

A duração utilizada para cobrança deve ser convertida em minutos.

Quando a duração não for múltipla da fração de cobrança, o sistema DEVE arredondar a quantidade de frações para cima.

Uma duração exatamente igual a uma fração DEVE resultar em uma única fração cobrada.

Uma duração que exceda uma fração em apenas um minuto DEVE resultar na cobrança da próxima fração.

O arredondamento de cobrança DEVE ocorrer antes da aplicação do valor final da tarifa.

---

## 9. Tolerância

A tolerância inicial é definida pela variante do projeto.

Quando a duração for menor ou igual à tolerância, o valor cobrado DEVE ser `0` centavos.

Quando a duração ultrapassar a tolerância, mesmo que por apenas um minuto, a cobrança DEVE ser realizada integralmente desde o primeiro minuto.

A tolerância NÃO DEVE ser subtraída da duração utilizada para cobrança após ser ultrapassada.

Quando a tolerância for `0`, toda duração deve ser considerada para cobrança.

---

## 10. Teto Diário

O valor cobrado por bilhete NUNCA DEVE ultrapassar o `TETO_DIARIO_CENTAVOS` definido pela variante.

Após o cálculo da cobrança, o valor final DEVE ser limitado ao teto configurado.

O valor retornado pela API DEVE permanecer em centavos inteiros.

---

## 11. Estados dos Bilhetes

Os estados válidos de um bilhete são:

* `aberto`;
* `encerrado`;
* `cancelado`.

As transições permitidas são:

| Estado atual | Operação     | Estado resultante |
| ------------ | ------------ | ----------------- |
| `aberto`     | encerramento | `encerrado`       |
| `aberto`     | cancelamento | `cancelado`       |

Bilhetes `encerrados` ou `cancelados` NÃO DEVEM retornar ao estado `aberto`.

Um bilhete encerrado NÃO DEVE ser encerrado novamente.

Um bilhete que não esteja aberto NÃO DEVE ser cancelado.

---

## 12. Unicidade de Bilhete Aberto

Uma placa pode possuir no máximo um bilhete aberto simultaneamente.

Uma tentativa de abertura de novo bilhete para uma placa que já possui um bilhete aberto DEVE resultar em `409` com o erro definido pelo contrato.

Após o encerramento ou cancelamento do bilhete existente, a placa DEVE poder iniciar um novo bilhete.

Essa regra DEVE ser preservada mesmo quando houver requisições concorrentes.

---

## 13. Cancelamento

O cancelamento representa desistência antes do encerramento do estacionamento.

Um bilhete somente pode ser cancelado enquanto estiver `aberto`.

O cancelamento:

* DEVE alterar o estado para `cancelado`;
* NÃO DEVE gerar cobrança;
* NÃO DEVE possuir horário de saída;
* NÃO DEVE possuir `valor_centavos`;
* DEVE permanecer disponível no histórico da placa.

---

## 14. Identidade e Persistência

Cada bilhete criado DEVE possuir um identificador único.

O identificador de um bilhete DEVE permanecer estável durante todo o seu ciclo de vida.

O sistema DEVE preservar os dados necessários para:

* consultar um bilhete pelo identificador;
* listar bilhetes ativos;
* consultar o histórico de uma placa;
* gerar o relatório diário;
* determinar o estado atual do bilhete.

O histórico DEVE preservar bilhetes independentemente de seu estado atual.

---

## 15. Ordenação

Quando o contrato determinar ordenação por recência, os resultados DEVEM ser apresentados do mais recente para o mais antigo.

A mesma regra deve ser aplicada de forma consistente em:

* bilhetes ativos;
* histórico de uma placa.

A ordenação NÃO DEVE depender da ordem interna de armazenamento dos dados.

---

## 16. Relatório Diário

O relatório diário DEVE considerar a data solicitada conforme as regras do contrato.

O tempo médio deve considerar somente os bilhetes encerrados no dia correspondente.

Bilhetes abertos ou cancelados NÃO DEVEM ser utilizados no cálculo do tempo médio de estacionamento.

O cálculo da média DEVE seguir a regra de arredondamento definida pelo contrato, incluindo o arredondamento de `0,5` para cima.

Os valores de faturamento DEVEM ser representados em centavos inteiros.

---

## 17. Tratamento de Erros

Os erros retornados pela API DEVEM utilizar exatamente os códigos e identificadores definidos pelo contrato.

Quando o contrato especificar um erro no campo `erro`, o nome desse erro DEVE ser preservado exatamente.

Exemplos de identificadores definidos pelo contrato:

* `placa_invalida`;
* `entrada_invalida`;
* `bilhete_nao_encontrado`;
* `bilhete_ja_encerrado`;
* `bilhete_nao_aberto`;
* `bilhete_em_aberto`;
* `data_invalida`.

A implementação NÃO DEVE substituir esses identificadores por mensagens genéricas.

---

## 18. Configuração e Execução

O serviço DEVE ser executável em ambiente de container.

A aplicação deve escutar na porta definida pela variante do projeto.

A implementação NÃO DEVE assumir uma porta fixa diferente daquela determinada pela configuração da variante.

O sistema NÃO DEVE exigir variáveis de ambiente obrigatórias quando o contrato não as exigir.

A execução deve ser reproduzível e não deve depender de interação manual para iniciar o serviço.

---

## 19. Testabilidade

As regras de negócio DEVEM ser implementadas de forma que possam ser verificadas por entradas controladas.

A implementação NÃO DEVE depender da passagem de tempo real para validar cenários que possam utilizar datas fornecidas explicitamente.

Os cálculos de cobrança DEVEM poder ser reproduzidos utilizando:

* duração;
* tarifa;
* fração;
* tolerância;
* teto diário.

As regras de negócio NÃO DEVEM depender de aleatoriedade.

---

## 20. Padrões de Código

A implementação DEVE:

* utilizar nomes claros e coerentes;
* manter funções e componentes com responsabilidades bem definidas;
* evitar duplicação desnecessária;
* manter regras de negócio isoladas;
* tratar erros explicitamente;
* utilizar constantes ou configurações para valores variáveis da aplicação;
* priorizar legibilidade e manutenção.

A implementação NÃO DEVE introduzir abstrações desnecessárias apenas para aumentar a complexidade estrutural do projeto.

---

## 21. Restrições

É PROIBIDO:

1. alterar os endpoints definidos pelo contrato;
2. alterar métodos HTTP;
3. alterar códigos de erro;
4. alterar nomes dos campos da API;
5. retornar valores monetários em ponto flutuante;
6. descontar a tolerância quando ela já tiver sido ultrapassada;
7. permitir mais de um bilhete aberto para a mesma placa;
8. permitir transições de estado não previstas;
9. ignorar o timezone definido;
10. retornar conflito `409` antes de validar uma entrada inválida;
11. adicionar funcionalidades que alterem o comportamento especificado;
12. depender de serviços externos desnecessários para executar a aplicação.

---

## 22. Prioridade das Regras

Em caso de conflito entre decisões, deve ser seguida a seguinte prioridade:

1. Contrato oficial da prova;
2. Regras obrigatórias desta Constituição;
3. Especificação funcional;
4. Decisões técnicas documentadas no plano;
5. Preferências de implementação.

Uma decisão técnica NUNCA pode alterar uma regra de prioridade superior.

---

## 23. Critérios de Conformidade

A implementação será considerada conforme quando:

* respeitar integralmente o contrato da API;
* retornar os status HTTP especificados;
* preservar os nomes dos campos e erros;
* aplicar corretamente as regras de cobrança;
* utilizar exclusivamente centavos inteiros para valores monetários;
* respeitar as regras de tolerância;
* respeitar o teto diário;
* manter a integridade dos estados dos bilhetes;
* impedir múltiplos bilhetes abertos para a mesma placa;
* respeitar as regras de validação;
* funcionar na porta determinada pela variante;
* permanecer determinística e testável.

Qualquer implementação que viole uma regra obrigatória deste documento deve ser considerada não conforme.
