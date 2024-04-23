# Guia de Estilo Solidity da Coinbase

Este é um guia para os engenheiros da Coinbase que desenvolvem contratos inteligentes baseados em EVM. Utilizamos Solidity para desenvolver tais contratos, portanto, chamamos isso de "Guia de Estilo Solidity." Este guia também abrange práticas de desenvolvimento e teste. Estamos compartilhando isso publicamente caso seja útil para outros.

## Por quê?

Devemos ser extremamente específicos e minuciosos ao definir nosso estilo, testes e práticas de desenvolvimento. Qualquer tempo que economizamos ao não ter que debater essas coisas em solicitações de pull é tempo produtivo que pode ser dedicado a outras discussões e revisões. Seguir o guia de estilo é evidência de cuidado.

![tom-sachs-gq-style-spring-2019-05](https://github.com/coinbase/solidity-style-guide/assets/6678357/9e904107-e83f-4d89-a405-d3f1394d8de4)

## 1. Estilo

### A. A menos que uma exceção ou adição seja especificamente notada, seguimos o [Guia de Estilo Solidity](https://docs.soliditylang.org/en/latest/style-guide.html).

### B. Exceções

#### 1. Nomes de funções internas em uma biblioteca não devem ter um prefixo de sublinhado.

O guia de estilo afirma:

> Prefixo de sublinhado para funções e variáveis não externas

Uma das motivações para esta regra é que ela é uma pista visual útil.

> Sublinhados iniciais permitem que você reconheça imediatamente a intenção de tais funções...

Concordamos que um sublinhado inicial é uma pista visual útil, e é por isso que nos opomos ao uso deles para funções internas de biblioteca que podem ser chamadas de outros contratos. Visualmente, parece errado.

```solidity
Library._function()
```

ou

```solidity
using Library for bytes
bytes._function()
```

Observe, não podemos remediar isso insistindo no uso de funções públicas. Se as funções de uma biblioteca são internas ou externas tem implicações importantes. Na [documentação do Solidity]
(https://docs.soliditylang.org/en/latest/contracts.html#libraries):

```
... o código das funções internas de uma biblioteca que são chamadas de um contrato e todas as funções chamadas a partir daí serão incluídas no contrato que faz a chamada no momento da compilação, e uma chamada JUMP regular será usada em vez de uma DELEGATECALL.
```

Os desenvolvedores podem preferir funções internas porque são mais eficientes em termos de gás para chamadas.

Se uma função nunca deve ser chamada de outro contrato, ela deve ser marcada como privada e seu nome deve ter um sublinhado inicial.

### C. Adições

#### 1. Prefira erros personalizados.

Erros personalizados são, em alguns casos, mais eficientes em termos de gás e permitem a passagem de informações úteis.

#### 2. Nomes de erros personalizados devem seguir o estilo CapWords.

Por exemplo, `InsufficientBalance`.

#### 3. Nomes de eventos devem ser no passado.

Por exemplo, `UpdatedOwner` não `UpdateOwner`.

Eventos devem rastrear coisas que _aconteceram_ e, portanto, devem ser no passado. Usar o passado também ajuda a evitar colisões de nomes com estruturas ou funções.

Estamos cientes de que isso não segue o precedente dos primeiros ERCs, como o [ERC-20](https://eips.ethereum.org/EIPS/eip-20). No entanto, isso se alinha com algumas implementações mais recentes de Solidity de alto perfil, exemplos: [1](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/976a3d53624849ecaef1231019d2052a16a39ce4/contracts/access/Ownable.sol#L33),[2](https://github.com/aave/aave-v3-core/blob/724a9ef43adf139437ba87dcbab63462394d4601/contracts/interfaces/IAaveOracle.sol#L25-L31),[3](https://github.com/ProjectOpenSea/seaport/blob/1d12e33b71b6988cbbe955373ddbc40a87bd5b16/contracts/zones/interfaces/PausableZoneEventsAndErrors.sol#L25-L41).

#### 4. Evite usar assembly.

O código assembly é difícil de ler e auditar. Devemos evitá-lo, a menos que as economias de gás sejam muito significativas, por exemplo, > 25%.

#### 5. Evite argumentos de retorno nomeados desnecessários.

Em funções curtas, argumentos de retorno nomeados são desnecessários.

NÃO:

```solidity
function add(uint a, uint b) public returns (uint result) {
  result = a + b;
}
```

Argumentos de retorno nomeados podem ser úteis em funções com múltiplos valores retornados.

```solidity
function validate(UserOperation calldata userOp) external returns (bytes memory context, uint256 validationData)
```

No entanto, é importante ser explícito ao retornar mais cedo.

NÃO:

```solidity
function validate(UserOperation calldata userOp) external returns (bytes memory context, uint256 validationData) {
  context = "";
  validationData = 1;

  if (condition) {
    return;
  }
}
```

SIM:

```solidity
function validate(UserOperation calldata userOp) external returns (bytes memory context, uint256 validationData) {
  context = "";
  validationData = 1;

  if (condition) {


 return (context, validationData);
  }
}
```

#### 6. Prefira composição a herança.

Se uma função ou conjunto de funções poderia razoavelmente ser definido como seu próprio contrato ou como parte de um contrato maior, prefira definir como parte de um contrato maior. Isso torna o código mais fácil de entender e auditar.

Observe que isso _não_ significa que devemos evitar a herança, em geral. A herança é útil às vezes, especialmente quando se constrói em contratos existentes e confiáveis. Por exemplo, _não_ reimplemente a funcionalidade `Ownable` para evitar a herança. Herde `Ownable` de um fornecedor confiável, como [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/) ou [Solady](https://github.com/Vectorized/solady).

#### 7. Evite escrever interfaces.

Interfaces separam NatSpec da lógica do contrato, exigindo que os leitores façam mais trabalho para entender o código. Por essa razão, elas devem ser evitadas.

#### 8. Evite restrições desnecessárias de versão Pragma.

Embora os contratos principais que implantamos devem especificar uma única versão do Solidity, todos os contratos de suporte e bibliotecas devem ter um Pragma tão aberto quanto possível. Uma boa regra é até a próxima versão principal. Por exemplo:

```solidity
pragma solidity ^0.8.0;
```

#### 9. Definições de Struct e Erro

##### A. Prefira declarar structs e erros dentro da interface, contrato ou biblioteca onde são usados.

##### B. Se um struct ou erro é usado em muitos arquivos, sem nenhuma interface, contrato ou biblioteca sendo razoavelmente o "dono," então defina-os em seu próprio arquivo. Vários structs e erros podem ser definidos juntos em um arquivo.

#### 10. Importações

##### A. Use importações nomeadas.

Importações nomeadas ajudam os leitores a entender o que exatamente está sendo usado e onde é originalmente declarado.

NÃO:

```solidity
import "./contract.sol"
```

SIM:

```solidity
import {Contract} from "./contract.sol"
```

Para conveniência, importações nomeadas não precisam ser usadas em arquivos de teste.

##### B. Ordene importações alfabeticamente (de A a Z) pelo nome do arquivo.

NÃO:

```solidity
import {B} from './B.sol'
import {A} from './A.sol'
```

SIM:

```solidity
import {A} from './A.sol'
import {B} from './B.sol'
```

##### C. Agrupe importações por externas e locais com uma nova linha entre elas.

Por exemplo:

```solidity
import {Math} from '/solady/Math.sol'

import {MyHelper} from './MyHelper.sol'
```

Em arquivos de teste, importações de `/test` devem ser seu próprio grupo também.

```solidity
import {Math} from '/solady/Math.sol'

import {MyHelper} from '../src/MyHelper.sol'

import {Mock} from './mocks/Mock.sol'
```

#### 11. Comentar para agrupar seções do código é permitido.

Às vezes, autores e leitores acham útil comentar divisórias entre grupos de funções. Isso é permitido, no entanto, garanta que o guia de estilo [ordenação de funções](https://docs.soliditylang.org/en/latest/style-guide.html#order-of-functions) ainda seja seguido.

Por exemplo:

```solidity
/// External Functions ///
```

```solidity
/*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/
/*                   VALIDATION OPERATIONS                    */
/*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/
```

#### 12. Arte ASCII

Arte ASCII é permitida no espaço entre o fim do Pragma e o início das importações.

#### 13. Prefira argumentos nomeados.

Passar argumentos para funções, eventos e erros com nomeação explícita ajuda na clareza

, especialmente quando o nome da variável passada não corresponde ao nome do parâmetro.

NÃO:

```
pow(x, y, v)
```

SIM:

```
pow({base: x, exponent: y, scalar: v})
```

## 2. Desenvolvimento

### A. Uso do [Forge](https://github.com/foundry-rs/foundry/tree/master/crates/forge) para teste e gerenciamento de dependências.

### B. Testes

#### 1. Nomes de arquivos de teste devem seguir as convenções do Guia de Estilo Solidity para nomes de arquivos e também ter `.t` antes de `.sol`.

Por exemplo, `ERC20.t.sol`

#### 2. Nomes de contratos de teste devem incluir o nome do contrato ou função sendo testado, seguido por "Test".

Por exemplo,

- `ERC20Test`
- `TransferFromTest`

#### 3. Nomes de teste devem seguir a convenção `test_functionName_outcome_optionalContext`

Por exemplo

- `test_transferFrom_debitsFromAccountBalance`
- `test_transferFrom_debitsFromAccountBalance_whenCalledViaPermit`
- `test_transferFrom_reverts_whenAmountExceedsBalance`

Se o contrato é nomeado após uma função, então o nome da função pode ser omitido.

```solidity
contract TransferFromTest {
  function test_debitsFromAccountBalance() ...
}
```

#### 4. Prefira testes que testem uma coisa.

Isso é geralmente uma boa prática, mas especialmente porque Forge não fornece números de linha em falhas de afirmação. Isso dificulta rastrear o que, exatamente, falhou se um teste tem muitas afirmações.

NÃO:

```solidity
function test_transferFrom_works() {
  // debita corretamente
  // credita corretamente
  // emite corretamente
  // reverte corretamente
}
```

SIM:

```solidity
function test_transferFrom_debitsFrom() {
  ...
}

function test_transferFrom_creditsTo() {
  ...
}

function test_transferFrom_emitsCorrectly() {
  ...
}

function test_transferFrom_reverts_whenAmountExceedsBalance() {
  ...
}
```

Observe, isso não significa que um teste deve ter apenas uma afirmação. Às vezes, ter várias afirmações é útil para certeza sobre o que está sendo testado.

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  ...
  assertEq(balanceOf(to), amount);
}
```

#### 5. Use variáveis para valores importantes em testes

NÃO:

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  transferFrom(from, to, 10);
  assertEq(balanceOf(to), 10);
}
```

SIM:

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  uint amount = 10;
  transferFrom(from, to, amount);
  assertEq(balanceOf(to), amount);
}
```

#### 6. Prefira testes de fuzz.

Tudo o mais sendo igual, prefira testes de fuzz.

NÃO:

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  uint amount = 10;
  transferFrom(from, to, amount);
  assertEq(balanceOf(to), amount);
}
```

SIM:

```solidity
function test_transferFrom_creditsTo(uint amount) {
  assertEq(balanceOf(to), 0);
  transferFrom(from, to, amount);
  assertEq(balanceOf(to), amount);
}
```

### C. Configuração do Projeto

#### 1. Evite remapeamentos personalizados.

[Remeapeamentos](https://book.getfoundry.sh/projects/dependencies?#remapping-dependencies) ajudam Forge a encontrar dependências com base em instruções de importação. Forge deduzirá automaticamente alguns remapeamentos, por exemplo

```rust
forge-std/=lib/forge-std/src/
solmate/=lib/solmate/src/
```

Devemos evitar adicionar a estes ou definir quaisquer remapeamentos explicitamente, pois isso torna nosso projeto mais difícil para outros usarem como uma dependência. Por exemplo, se nosso projeto depende de Solmate e o deles também, queremos evitar que nosso projeto tenha algum nome de importação irregular, resolvido com um remapeamento personalizado, que entrará em conflito com o nome de importação deles.

### D. Atualizabilidade

#### 1. Prefira a convenção "Namespaced Storage Layout" do [ERC-7201](https://eips.ethereum.org/EIPS/eip-7201) para evitar colisões de armazenamento.

### E. Structs

#### 1. Sempre que possível, os valores da struct devem ser compactados para minimizar SLOADs e SSTOREs.

#### 2. Campos de carimbo de data/hora em uma struct devem ter pelo menos uint32 e idealmente ser uint40.

`uint32` dará ao contrato cerca de 82 anos de validade `(2^32 / (60*60*24*365)) - (2024 - 1970)`. Se o espaço permitir, uint40 é o tamanho preferido.

## 3. NatSpec

### A. A menos que uma exceção ou adição seja especificamente notada, siga [Solidity NatSpec](https://docs.soliditylang.org/en/latest/natspec-format.html).

### B. Adições

#### 1. Todas as funções externas, eventos e erros devem ter NatSpec completo.

Minimamente incluindo um `@notice`. `@param` e `@return` devem estar presentes se houver parâmetros ou valores de retorno.

#### 2. NatSpec de Struct

Structs podem ser documentados com um `@notice` acima e, se desejado, `@dev` para cada campo.

```solidity
/// @notice Uma struct que descreve a posição de uma conta
struct Position {
  /// @dev O carimbo de data/hora unix (segundos) do bloco quando a posição foi criada.
  uint created;
  /// @dev A quantidade de ETH na posição
  uint amount;
}
```

#### 3. Novas linhas entre tipos de tags.

Para facilitar a leitura, adicione uma nova linha entre os tipos de tags, quando várias estão presentes e há três ou mais linhas.

NÃO:

```solidity
/// @notice ...
/// @dev ...
/// @dev ...
/// @param ...
/// @param ...
/// @return
```

SIM:

```solidity
/// @notice ...
///
/// @dev ...
/// @dev ...
///
/// @param ...
/// @param ...
///
/// @return
```

#### 4. O autor deve ser a Coinbase.

Se você estiver usando a tag `@author`, ela deve ser

```solidity
/// @author Coinbase
```

Seguido opcionalmente por um link para o repositório público no Github.
