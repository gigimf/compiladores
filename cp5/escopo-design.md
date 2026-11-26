# CP4 — Projeto de Escopo e Identificação Contextual

**Aluna / Grupo:** Giovana Ferreira  
**Disciplina:** Compiladores / Construção de Compiladores  
**Etapa:** Checkpoint 4 (Análise Contextual — Tabela de Símbolos e Passada de Identificação)

---

## 1. Contexto e Objetivo

O Checkpoint 4 constrói a camada de **análise contextual (semântica)** sobre a árvore sintática abstrata (AST) validada no Checkpoint 3. Enquanto o parser (CP2/CP3) apenas garante que as sequências de tokens respeitam a gramática livre de contexto, ele é incapaz de detectar inconsistências contextuais, tais como:
- O uso de uma variável ou rotina que nunca foi declarada (`x = 1;` sem `int x;`).
- A redeclaração de um mesmo identificador no mesmo bloco léxico (`int a = 1; int a = 2;`).
- A visibilidade de identificadores fora do seu escopo de vida.

Nesta etapa, implementamos:
1. **O Modelo de Ambiente Léxico**: formalização do desenho de escopos e regras de visibilidade.
2. **A Tabela de Símbolos (`SymbolTable`)**: estrutura de dados responsável pelo gerenciamento de escopos e símbolos, implementando rigorosamente a interface da Aula 11.
3. **A Passada de Identificação (`IdentificationVisitor`)**: um `Visitor<Void>` puro sobre a AST, sem `instanceof` e sem `switch` de nós, que decora cada identificador de uso (`IdentNode`) com a referência ao seu respectivo `Symbol` (`decl`), além de coletar erros determinísticos.

---

## 2. Parte 1 — Desenho do Ambiente e Decisões de Escopo

### 2.1. Ambiente Ponto a Ponto em Programa de Teste

Abaixo apresentamos o programa representativo (`programas/valido-sombreamento.smol`), contendo:
- Linhas numeradas;
- Três níveis de blocos aninhados (nível 1 da função, bloco nível 2 e bloco nível 3);
- Um identificador sombreado (`x@5` sombreando `x@1`);
- Uso após o fechamento do bloco (`x` na linha 11 resolvendo para `x@1`, e `a` na linha 11 resolvendo para `a@3`);
- Uma função com parâmetro (`int f(int p)`).

```c
1: int x = 10;
2: int f(int p) {
3:     int a = 1;
4:     {
5:         int x = 20;
6:         {
7:             int b = x + p;
8:         }
9:         int c = x + a;
10:    }
11:    int d = x + a;
12:    return d;
13: }
14: main() {
15:    int res = f(x);
16: }
```

#### Evolução do Ambiente Léxico Passo a Passo

Representamos a pilha de escopos como `[Nível 0] -> [Nível 1] -> ... -> [Nível K]`, onde o topo da pilha é o escopo mais interno.

| Ponto no Código | Ação na Tabela de Símbolos | Ambiente Resultante (Pilha de Escopos) | Resolução de Usos (se houver) |
|---|---|---|---|
| **Início** | Inicialização global | `[0: {}]` | — |
| **Linha 1** | Declara `x` (var global) | `[0: {x@1}]` | — |
| **Linha 2** | Declara função `f` no escopo 0 | `[0: {x@1, f@2}]` | — |
| **Linha 2** | `openScope()` e declara parâmetro `p` | `[0: {x@1, f@2}] -> [1: {p@2}]` | — |
| **Linha 3** | Declara `a` (var local de `f`) | `[0: {x@1, f@2}] -> [1: {p@2, a@3}]` | — |
| **Linha 4** | `openScope()` (bloco aninhado nível 2) | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {}]` | — |
| **Linha 5** | Declara `x` (sombreia `x@1`) | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5}]` | — |
| **Linha 6** | `openScope()` (bloco aninhado nível 3) | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5}] -> [3: {}]` | — |
| **Linha 7** | Usos de `x` e `p` | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5}] -> [3: {}]` | `x` resolve para **`x@5`** (nível 2); `p` resolve para **`p@2`** (nível 1) |
| **Linha 7** | Declara `b` no nível 3 | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5}] -> [3: {b@7}]` | — |
| **Linha 8** | `closeScope()` (sai do nível 3) | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5}]` | `b@7` é desalocado do ambiente |
| **Linha 9** | Usos de `x` e `a` | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5}]` | `x` resolve para **`x@5`** (nível 2); `a` resolve para **`a@3`** (nível 1) |
| **Linha 9** | Declara `c` no nível 2 | `[0: {x@1, f@2}] -> [1: {p@2, a@3}] -> [2: {x@5, c@9}]` | — |
| **Linha 10** | `closeScope()` (sai do nível 2) | `[0: {x@1, f@2}] -> [1: {p@2, a@3}]` | `x@5` e `c@9` saem de escopo |
| **Linha 11** | Usos de `x` e `a` | `[0: {x@1, f@2}] -> [1: {p@2, a@3}]` | `x` resolve para **`x@1`** (global restaurado!); `a` resolve para **`a@3`** |
| **Linha 11** | Declara `d` no nível 1 | `[0: {x@1, f@2}] -> [1: {p@2, a@3, d@11}]` | — |
| **Linha 12** | Uso de `d` | `[0: {x@1, f@2}] -> [1: {p@2, a@3, d@11}]` | `d` resolve para **`d@11`** |
| **Linha 13** | `closeScope()` (sai de `f`) | `[0: {x@1, f@2}]` | Escopo de `f` é fechado |
| **Linha 14** | Declara procedimento `main` no escopo 0 | `[0: {x@1, f@2, main@14}]` | — |
| **Linha 14** | `openScope()` (corpo de `main`) | `[0: {x@1, f@2, main@14}] -> [1: {}]` | — |
| **Linha 15** | Usos de `f` e `x` | `[0: {x@1, f@2, main@14}] -> [1: {}]` | `f` resolve para **`f@2`**; `x` resolve para **`x@1`** |
| **Linha 15** | Declara `res` no nível 1 | `[0: {x@1, f@2, main@14}] -> [1: {res@15}]` | — |
| **Linha 16** | `closeScope()` (sai de `main`) | `[0: {x@1, f@2, main@14}]` | Fim do programa |

---

### 2.2. Estrutura do `Symbol`: O que Guarda e Por Quê

A classe `Symbol` foi projetada para unificar as informações semânticas necessárias no CP4 e servir de base estável para as fases futuras (Aulas 12 a 19):

| Campo | Tipo | Papel / Significado | Fase do Compilador que Pergunta |
|---|---|---|---|
| `name` | `String` | Nome / identificador do símbolo | **Identificação (CP4):** chave de busca nos mapas da tabela (`lookup`, `lookupLocal`) e mensagens de erro contextuais. |
| `category` | `SymbolCategory` | Categoria sintático-semântica (`VARIABLE`, `PARAMETER`, `FUNCTION`, `PROCEDURE`) | **Identificação (CP4)** e **Tipos (CP5):** diferencia uso de variável vs chamada de função; **Geração de Código (CP6):** instrução `call` vs `load/store`. |
| `declNode` | `Node` | Ponteiro para o nó da AST onde o símbolo foi declarado | **Identificação (CP4):** recuperação da linha original de declaração para mensagens de erro determinísticas (`linha 2: 'a' já declarado neste escopo (linha 1)`); **Tipos (CP5):** inspeção de assinaturas. |
| `level` | `int` | Profundidade léxica de aninhamento onde o símbolo reside (0 = global, 1 = corpo da rotina, ...) | **Identificação (CP4):** verificação de integridade e consistência de escopo; **Geração de Código (CP6):** cálculo de display / static chain ou alocação global vs frame de pilha. |
| `type` *(reservado)* | `TypeKind` | Tipo estático inferido ou declarado (`int`, `double`, `bool`, `char`, `string`) | **Checagem de Tipos (CP5):** validação de compatibilidade em atribuições, operadores binários/unários e argumentos de rotinas. |
| `offset` *(reservado)* | `int` | Deslocamento relativo de memória (em bytes) na pilha de ativação ou segmento global | **Geração de Código (CP6):** cálculo de endereçamento `[rbp - offset]` para variáveis locais e parâmetros. |

---

### 2.3. As Cinco Decisões de Projeto

#### Decisão 1: Sombreamento (Shadowing) e Redeclaração no Mesmo Bloco
- **Escolha:** Sombreamento entre escopos distintos é **permitido**. Um bloco interno pode declarar um identificador de mesmo nome de um bloco mais externo (ou global), ocultando-o temporariamente até que o bloco interno seja fechado. Por outro lado, redeclarar um identificador **no mesmo bloco léxico é sempre erro**.
- **Alternativa Descartada:** Proibir qualquer sombreamento léxico (como regras restritas de certas linguagens ou C# para certas variáveis de escopo local aninhado).
- **Motivo:** O sombreamento léxico é o padrão consolidado em C, C++, Java e Pascal. Ele oferece modularidade para que blocos e rotinas declarem nomes naturais (como `i`, `temp`, `x`) sem conflitar com nomes de escopos externos. A proibição de redeclaração no mesmo bloco evita erros involuntários de duplicação.
- **Programa de Teste:** `programas/decisao1-sombreamento.smol`.

#### Decisão 2: Declarar Antes de Usar (Declare-before-use)
- **Escolha:** **Obrigatório**. Todo uso de variável deve ocorrer estritamente após sua declaração textual no fluxo de execução do bloco. Na declaração de variável com inicializador (`int x = expr;`), a expressão `expr` é avaliada antes de inserir `x` no escopo, garantindo que `x` não seja usado em seu próprio inicializador sem declaração externa.
- **Alternativa Descartada:** Içamento total de variáveis (hoisting) no escopo inteiro (estilo `var` em JavaScript ou tipos em C#).
- **Motivo:** A regra sequencial linear simplifica o raciocínio do programador e permite a implementação da identificação em passada única (`one-pass visitor`), garantindo determinismo temporal na construção do ambiente léxico.
- **Programa de Teste:** `programas/decisao2-declarar-antes-usar.smol`.

#### Decisão 3: Funções Mutuamente Recursivas
- **Escolha:** Em passada única, funções **não podem** ser mutuamente recursivas caso uma chame a outra antes da sua definição textual. A recursão simples (direta) é totalmente suportada, pois a função é inserida na tabela antes de percorrer seu corpo. Funções subsequentes podem chamar funções anteriores.
- **Alternativa Descartada:** Suportar recursão mútua por meio de uma passada prévia de coleta de cabeçalhos ou suporte a *forward declarations* (protótipos).
- **Motivo:** A gramática da linguagem (CP2/CP3) não possui produções para declarações antecipadas / protótipos de função sem corpo (`int f(int x);`). Para manter o compilador elegante, determinístico e fiel ao contrato de uma única passada semântica no CP4, exige-se declaração prévia.
- **Programa de Teste:** `programas/decisao3-recursao-mutua.smol`.

#### Decisão 4: Parâmetros e Corpo da Função
- **Escolha:** Parâmetros e o bloco raiz do corpo da função **compartilham o mesmo escopo**. Declarar uma variável local no primeiro nível do corpo com o mesmo nome de um parâmetro (ex.: `void f(int a) { int a = 1; }`) é categorizado imediatamente como erro de **redeclaração**. Blocos internos aninhados (`if`, `while`, `{}`) abrem novos escopos e podem sombrear o parâmetro.
- **Alternativa Descartada:** Abrir um novo escopo filho para o corpo da função, permitindo que a linha inicial do corpo sombreie o parâmetro formal.
- **Motivo:** Padrão adotado em Java, C e Go. Sombrear um parâmetro imediatamente no início da função quase invariavelmente representa um engano do desenvolvedor, tornando o parâmetro inacessível sem qualquer utilidade prática.
- **Programa de Teste:** `programas/decisao4-param-corpo.smol`.

#### Decisão 5: Espaços de Nomes (Namespaces)
- **Escolha:** **Espaço de nomes único unificado** para variáveis, parâmetros, procedimentos e funções.
- **Alternativa Descartada:** Espaços de nomes separados por categoria (ex.: tabela de funções separada da tabela de variáveis, estilo Common Lisp / Lisp-2).
- **Motivo:** Adota a semântica da linguagem C e Pascal. Evita ambiguidades em que o mesmo identificador refere-se a coisas diferentes no mesmo contexto léxico (`int foo; int foo() { ... }` é proibido), simplifica a arquitetura da `SymbolTable` e prepara o terreno para que, futuramente, funções possam ser tratadas como valores de primeira classe (ponteiros/referências para função).
- **Programa de Teste:** `programas/decisao5-espaco-nomes.smol`.

---

## 3. Parte 2 — A Tabela de Símbolos (`SymbolTable`)

### 3.1. Escolha da Implementação: Pilha de Mapas

Dentre as três abordagens clássicas vistas na Aula 11 (Pilha de Mapas, Escopos Encadeados e Tabela Única com Registro), escolhemos a **Pilha de Mapas** (`Deque<Map<String, Symbol>>` encapsulada na classe `ScopedSymbolTable`).

**Justificativa:**
A pilha de mapas reflete diretamente a semântica de pilha de blocos da linguagem estruturada. As operações `openScope` e `closeScope` operam em tempo constante $O(1)$ inserindo e descartando mapas do topo; `lookupLocal` consulta apenas o mapa ativo no topo em $O(1)$; e `lookup` inspeciona a pilha do topo até o escopo global. Essa implementação desacopla completamente os nós da AST do ciclo de vida da tabela e previne vazamento de referências após o fechamento de escopos.

### 3.2. Contrato da API e Tratamento de Casos de Borda

A classe `ScopedSymbolTable` implementa a interface `SymbolTable` da Aula 11 sem qualquer alteração de assinatura:

```java
public interface SymbolTable {
    void openScope();                // entra em um bloco
    void closeScope();               // sai do bloco
    boolean declare(Symbol s);       // false: já existe neste escopo
    Symbol lookupLocal(String name); // só no escopo do topo
    Symbol lookup(String name);      // do mais interno ao mais externo
    int level();                     // profundidade atual (0 = global)
}
```

- **Não substituição em redeclaração:** Caso `lookupLocal(s.getName())` encontre um símbolo preexistente no escopo atual, `declare(s)` retorna `false` e o símbolo original permanece intacto no mapa.
- **Consultas seguras:** `lookup` e `lookupLocal` devolvem `null` quando o identificador não é encontrado e tratam argumentos nulos sem lançar exceções.
- **Proteção do escopo global:** Caso `closeScope()` seja chamado quando `level() == 0` (isto é, tentando fechar o escopo global), o método lança uma `IllegalStateException` descritiva e documentada, impedindo qualquer `NullPointerException` descontrolada.
- **Rastreamento de nível:** O nível inicial na instanciação da tabela é 0 (global). Cada chamada a `openScope()` incrementa o nível e cada chamada a `closeScope()` decrementa-o de volta.

---

## 4. Parte 3 — A Passada de Identificação

A classe `IdentificationVisitor` implementa a interface `Visitor<Void>` sobre a hierarquia da AST:
- **Zero `instanceof` e zero `switch` sobre tipos de nós:** todo o despacho polimórfico ocorre exclusivamente pelo mecanismo padrão do Visitor (`accept`).
- **Decoração da AST:** o campo `decl` foi adicionado à classe `IdentNode` (`private Symbol decl;` com getter e setter correspondentes). Esta é rigorosamente a única alteração permitida na hierarquia de nós.
- **Tratamento de Declarações:** variáveis, parâmetros, procedimentos e funções criam instâncias de `Symbol` com `table.level()` e invocam `table.declare(sym)`. Em caso de falha, recupera-se a linha da declaração original através de `existing.getDeclNode().getLinha()` para formatar a mensagem de erro.
- **Tratamento de Usos:** `IdentNode` consulta a tabela via `table.lookup(name)`. Se o símbolo existir, vincula-o em `node.setDecl(s)`; caso contrário, reporta `'nome' não declarado` e deixa `decl` nulo. Os nós de comando que contêm identificadores (como `AssignStmtNode`, `CallStmtNode` e `CallExprNode`) também validam a declaração de seus alvos na tabela.
- **Formato Determinístico de Erros:**
  - Redeclaração: `linha <L>: '<nome>' já declarado neste escopo (linha <Ldecl>)`
  - Não declarado: `linha <L>: '<nome>' não declarado`
- **Robustez:** A passada executa sobre toda a AST acumulando todos os erros sem interrupção antecipada, nunca lança `NullPointerException` e não entra em laço infinito. Ao término, a lista de erros fica disponível via `getErros()`.

---

## 5. Suíte de Testes e Resultados

A suíte de testes é composta por:
1. `SymbolTableTest`: bateria de 24 verificações unitárias da tabela isolada (declaração, busca, sombreamento, restauração pós-fechamento, redeclaração, lookupLocal, nível e exceção em fechamento indevido).
2. `IdentificationTest`: bateria de 24 verificações semânticas completas, confrontando as saídas reais contra arquivos pré-escritos à mão em `test/esperado/`:
   - `valido-sombreamento.smol` vs `test/esperado/valido-sombreamento.txt`
   - `redeclaracao.smol` vs `test/esperado/redeclaracao.txt`
   - `nao-declarado.smol` vs `test/esperado/nao-declarado.txt`
   - Programas das cinco decisões de projeto (`decisao1` a `decisao5`)
   - `tres-erros.smol` (múltiplos erros acumulados na ordem do texto)
   - `procedimento-recursivo.smol` (procedimento recursivo válido)
   - `uso-apos-bloco.smol` (variável de bloco interno inacessível externamente)
3. `ParserTest`: regressão completa de 35 testes do CP2 (sintático).
4. `AstTest`: regressão completa de 15 testes do CP3 (impressão e round-trip da AST).

Todas as 98 verificações executam com **0 falhas**.
