# Resposta-Questao-5
 API REST,   

• Movimentação de uma conta corrente;    

• Consulta do saldo da conta corrente;



### 1. **Movimentação de uma conta corrente (POST /movimentacao)**
   - **Requisitos:**
     - Receber identificação da requisição (idempotência), identificação da conta corrente, valor a ser movimentado, e tipo de movimento (C = Crédito, D = Débito).
     - Verificações de validação:
       - A conta existe e está cadastrada (se não, retornar `INVALID_ACCOUNT`).
       - A conta está ativa (se não, retornar `INACTIVE_ACCOUNT`).
       - O valor da movimentação é positivo (se não, retornar `INVALID_VALUE`).
       - O tipo de movimentação é válido (C ou D, caso contrário, retornar `INVALID_TYPE`).
     - Implementar idempotência: armazenar requisições idempotentes no banco e reutilizar o resultado quando a mesma requisição for repetida.
     - Persistir movimentações válidas na tabela `movimento` e retornar o ID do movimento.
     - Caso as validações falhem, retornar erro HTTP 400 com a descrição da falha.
     - Caso seja bem-sucedido, retornar HTTP 200 com o ID do movimento.

   - **Implementação:**
     - No serviço de movimentação, ao receber uma requisição, verificar:
       1. Se a conta existe e está ativa.
       2. Se o valor da movimentação é positivo.
       3. Se o tipo de movimentação é válido.
       4. Se o ID de idempotência já existe, retornar o resultado prévio.
       5. Persistir a movimentação na tabela `movimento` com o tipo e valor.
       6. Retornar o ID do movimento.

   - **Exemplo de código (simplificado em C#):**
     ```csharp
     [HttpPost("movimentacao")]
     public IActionResult MovimentarConta(MovimentacaoRequest request)
     {
         // Validar conta corrente
         var conta = _db.Contas.FirstOrDefault(c => c.Id == request.IdContaCorrente);
         if (conta == null) return BadRequest("INVALID_ACCOUNT");
         if (!conta.Ativo) return BadRequest("INACTIVE_ACCOUNT");

         // Validar valor
         if (request.Valor <= 0) return BadRequest("INVALID_VALUE");

         // Validar tipo de movimento
         if (request.TipoMovimento != 'C' && request.TipoMovimento != 'D') return BadRequest("INVALID_TYPE");

         // Verificar idempotência
         var idempotencia = _db.Idempotencias.FirstOrDefault(i => i.Chave == request.ChaveIdempotencia);
         if (idempotencia != null) return Ok(idempotencia.Resultado);

         // Criar movimento
         var movimento = new Movimento
         {
             Id = Guid.NewGuid(),
             IdContaCorrente = request.IdContaCorrente,
             DataMovimento = DateTime.Now,
             TipoMovimento = request.TipoMovimento,
             Valor = request.Valor
         };
         _db.Movimentos.Add(movimento);
         _db.SaveChanges();

         // Persistir idempotência
         _db.Idempotencias.Add(new Idempotencia { Chave = request.ChaveIdempotencia, Requisicao = JsonConvert.SerializeObject(request), Resultado = JsonConvert.SerializeObject(movimento) });
         _db.SaveChanges();

         return Ok(new { IdMovimento = movimento.Id });
     }
     ```

### 2. **Consulta do saldo da conta corrente (GET /contacorrente/saldo/{idConta})**

   - **Requisitos:**
     - Receber a identificação da conta corrente e retornar o saldo atual da conta.
     - O saldo é calculado como a diferença entre a soma dos créditos (`C`) e a soma dos débitos (`D`) para a conta.
     - Validações de negócio:
       - Verificar se a conta está cadastrada e ativa.
     - Caso as validações sejam atendidas, retornar o saldo atual e informações da conta.
     - Caso contrário, retornar erro HTTP 400 com a descrição da falha.

   - **Implementação:**
     - No serviço de consulta de saldo, ao receber a requisição:
       1. Verificar se a conta existe e está ativa.
       2. Somar todos os créditos (movimentos do tipo `C`) e subtrair os débitos (movimentos do tipo `D`).
       3. Retornar o saldo calculado junto com o número da conta e nome do titular.
   
   - **Exemplo de código (simplificado em C#):**
     ```csharp
     [HttpGet("contacorrente/saldo/{idConta}")]
     public IActionResult ConsultarSaldo(string idConta)
     {
         // Validar conta corrente
         var conta = _db.Contas.FirstOrDefault(c => c.Id == idConta);
         if (conta == null) return BadRequest("INVALID_ACCOUNT");
         if (!conta.Ativo) return BadRequest("INACTIVE_ACCOUNT");

         // Calcular saldo
         var creditos = _db.Movimentos.Where(m => m.IdContaCorrente == idConta && m.TipoMovimento == 'C').Sum(m => m.Valor);
         var debitos = _db.Movimentos.Where(m => m.IdContaCorrente == idConta && m.TipoMovimento == 'D').Sum(m => m.Valor);
         var saldo = creditos - debitos;

         // Retornar saldo
         return Ok(new
         {
             NumeroConta = conta.Numero,
             NomeTitular = conta.Nome,
             DataConsulta = DateTime.Now,
             Saldo = saldo
         });
     }
     ```

### 3. **Considerações adicionais:**
   - **Idempotência**: A implementação usa a tabela de idempotência para armazenar requisições repetidas, permitindo a repetição segura de chamadas sem causar movimentações duplicadas.
   - **Validações de negócio**: As validações foram implementadas para garantir a integridade dos dados e as regras de negócio (ex.: apenas contas ativas podem movimentar).
   - **Testes unitários**: Para garantir a qualidade, criar testes para os serviços que verificam tanto as rotas de sucesso quanto as de falha (ex.: conta inexistente, conta inativa, valores negativos).
   - **Swagger**: Documentar as APIs no Swagger, para que o consumo seja simples e padronizado, incluindo exemplos de requisição e respostas possíveis.


