## JTA

# 1. Visão Geral
Java Transaction API, mais comumente conhecido como JTA, é uma API para gerenciar transações em Java. Ele nos permite iniciar, confirmar e reverter transações de uma forma agnóstica de recursos.

O verdadeiro poder do JTA está em sua capacidade de gerenciar vários recursos (ou seja, bancos de dados, serviços de mensagens) em uma única transação.

Neste tutorial, vamos conhecer o JTA no nível conceitual e ver como o código de negócios normalmente interage com o JTA.

# 2. API universal e transação distribuída
JTA fornece uma abstração sobre o controle de transação (início, confirmação e reversão) para o código de negócios.

Na ausência dessa abstração, teríamos que lidar com as APIs individuais de cada tipo de recurso.

Por exemplo, precisamos lidar com o recurso JDBC assim. Da mesma forma, um recurso JMS pode ter um modelo semelhante, mas incompatível.

Com o JTA, podemos gerenciar vários recursos de diferentes tipos de maneira consistente e coordenada.

Como uma API, JTA define interfaces e semânticas a serem implementadas por gerenciadores de transações. As implementações são fornecidas por bibliotecas como Narayana e Bitronix.

# 3. Exemplo de configuração do projeto
O aplicativo de amostra é um serviço de backend muito simples de um aplicativo financeiro. Temos dois serviços, o BankAccountService e AuditService usando dois bancos de dados diferentes. Esses bancos de dados independentes precisam ser coordenados no início, confirmação ou reversão da transação.

Para começar, nosso projeto de amostra usa Spring Boot para simplificar a configuração:

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.5.1</version>
</parent>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jta-bitronix</artifactId>
</dependency>
```

Finalmente, antes de cada método de teste, inicializamos AUDIT_LOG com dados vazios e o banco de dados ACCOUNT com 2 linhas:

```
+-----------+----------------+
| ID        |  BALANCE       |
+-----------+----------------+
| a0000001  |  1000          |  
| a0000002  |  2000          |
+-----------+----------------+
```

# 4. Demarcação de transação declarativa
A primeira forma de trabalhar com transações em JTA é com o uso da anotação @Transactional. Para obter uma explicação e configuração mais elaboradas, consulte este artigo.

Vamos anotar o método de serviço de fachada executeTranser() com @Transactional. Isso instrui o gerenciador de transações a iniciar uma transação:

```
@Transactional
public void executeTransfer(String fromAccontId, String toAccountId, BigDecimal amount) {
    bankAccountService.transfer(fromAccontId, toAccountId, amount);
    auditService.log(fromAccontId, toAccountId, amount);
    ...
}
```

Aqui, o método executeTranser() chama 2 serviços diferentes, AccountService e AuditService. Esses serviços usam 2 bancos de dados diferentes.

Quando executeTransfer() retorna, o gerenciador de transações reconhece que é o fim da transação e se compromete com os dois bancos de dados:

```
tellerService.executeTransfer("a0000001", "a0000002", BigDecimal.valueOf(500));
assertThat(accountService.balanceOf("a0000001"))
  .isEqualByComparingTo(BigDecimal.valueOf(500));        
assertThat(accountService.balanceOf("a0000002"))
  .isEqualByComparingTo(BigDecimal.valueOf(2500));

TransferLog lastTransferLog = auditService
  .lastTransferLog();
assertThat(lastTransferLog)
  .isNotNull();        
assertThat(lastTransferLog.getFromAccountId())
  .isEqualTo("a0000001");
assertThat(lastTransferLog.getToAccountId())
  .isEqualTo("a0000002"); 
assertThat(lastTransferLog.getAmount())
  .isEqualByComparingTo(BigDecimal.valueOf(500));
```

### 4.1. Revertendo na Demarcação Declarativa
No final do método, executeTransfer() verifica o saldo da conta e lança RuntimeException se o fundo de origem for insuficiente:

```
@Transactional
public void executeTransfer(String fromAccontId, String toAccountId, BigDecimal amount) {
    bankAccountService.transfer(fromAccontId, toAccountId, amount);
    auditService.log(fromAccontId, toAccountId, amount);
    BigDecimal balance = bankAccountService.balanceOf(fromAccontId);
    if(balance.compareTo(BigDecimal.ZERO) < 0) {
        throw new RuntimeException("Insufficient fund.");
    }
}
```

Uma RuntimeException não tratada após o primeiro @Transactional reverterá a transação para ambos os bancos de dados. Com efeito, a execução de uma transferência com um valor maior do que o saldo causará uma reversão:

```
assertThatThrownBy(() -> {
    tellerService.executeTransfer("a0000002", "a0000001", BigDecimal.valueOf(10000));
}).hasMessage("Insufficient fund.");

assertThat(accountService.balanceOf("a0000001")).isEqualByComparingTo(BigDecimal.valueOf(1000));
assertThat(accountService.balanceOf("a0000002")).isEqualByComparingTo(BigDecimal.valueOf(2000));
assertThat(auditServie.lastTransferLog()).isNull();
```

# 5. Demarcação de transação programática
Outra maneira de controlar a transação JTA é programaticamente por meio de UserTransaction.

Agora vamos modificar executeTransfer() para lidar com a transação manualmente:

```
userTransaction.begin();
 
bankAccountService.transfer(fromAccontId, toAccountId, amount);
auditService.log(fromAccontId, toAccountId, amount);
BigDecimal balance = bankAccountService.balanceOf(fromAccontId);
if(balance.compareTo(BigDecimal.ZERO) < 0) {
    userTransaction.rollback();
    throw new RuntimeException("Insufficient fund.");
} else {
    userTransaction.commit();
}
```

Em nosso exemplo, o método begin() inicia uma nova transação. Se a validação do saldo falhar, chamamos rollback(), que fará rollback em ambos os bancos de dados. Caso contrário, a chamada para commit() confirma as alterações em ambos os bancos de dados.

É importante observar que tanto commit() quanto rollback() finalizam a transação atual.

Em última análise, o uso da demarcação programática nos dá a flexibilidade de um controle de transação refinado.

# 6. Conclusão
Neste artigo, discutimos o problema que o JTA tenta resolver. Os exemplos de código ilustram o controle da transação com anotações e de forma programática, envolvendo 2 recursos transacionais que precisam ser coordenados em uma única transação.
