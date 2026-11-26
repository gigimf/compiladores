# CP5 — Giovana (Verificador de Tipos)

Entrega do Checkpoint 5 (Compiladores) por **Giovana Ferreira**: o compilador da smollang passa a calcular o tipo de
cada expressão e a recusar, com linha e mensagem, o programa que viola as regras de tipos. Parte da AST decorada do CP4
(cada uso de nome aponta para a sua declaração). **Não gera código**: o tipo gravado em cada expressão é o que o CP6 vai ler.

## Entregáveis

| Parte | Onde |
|---|---|
| 1. Regras de tipos (à mão) | `tipos-a-mao.pdf` — **a ser escrito à mão e fotografado/digitalizado**. Em `tipos-a-mao-RASCUNHO.pdf` há uma versão digitada só para copiar (regras, tabela de operadores, 11 decisões, registro da TAM). O rascunho não vale como entrega. |
| 2. Verificador | `src/` (este README explica a representação e onde fica cada regra) |
| 3. Testes | `test/programas/` + `test/esperado/` (+ `OperatorTableTest.java`, `TypeCheckTest.java`) |
| Log de IA | `log-uso-ia.md` |

Herdados sem mudança de conteúdo: `grammar.bnf`, `escopo-design.md` (documento do CP4), `programas/` (programas do CP4) e `cp1-sha256.txt`.

## Como compilar, testar e usar

Requisitos: JDK 17 ou superior e Python 3.

```bash
cd cp5-giovana
mkdir -p out
javac --release 17 -encoding UTF-8 -Xlint:all -Werror -d out $(find src test -name '*.java')

java -cp out OperatorTableTest   # as regras isoladas (tabela de operadores), sem parser
java -cp out TypeCheckTest       # programas x saídas esperadas, tipos gravados, robustez
java -cp out SymbolTableTest     # regressão CP4 (tabela de símbolos)
java -cp out IdentificationTest  # regressão CP4 (identificação)
java -cp out ParserTest          # regressão CP2
java -cp out AstTest             # regressão CP3

python3 tools/verify_all.py      # tudo: SHA256 do CP1, build com -Werror, todas as suítes e a CLI

java -cp out Main test/programas/valido_todos_tipos.txt   # ACEITO (exit 0)
java -cp out Main test/programas/varios_erros.txt         # mensagens no stderr, REJEITADO (exit 1)
```

Com qualquer erro (de identificação ou de tipos) a CLI imprime `REJEITADO` e sai com código 1: **o compilador não segue para a
geração de código**. Avisos não bloqueiam.

## Como os tipos estão representados

- **`TypeKind`** (enum, `src/ast/`): `INT, DOUBLE, BOOL, CHAR, STRING` (os tipos da linguagem), mais dois tipos internos:
  `VOID` (retorno de procedimento; nunca aparece em declaração) e `ERRO` (expressão já reportada; compatível com tudo, não gera mensagem nova).
- **Expressão**: o tipo fica gravado no nó, em `Node.getTipo()` / `setTipo()` (é `null` antes do `TypeChecker` passar).
- **Símbolo** (`src/symbols/Symbol`): `type` = tipo da variável/parâmetro ou tipo de retorno da função; `getParamTypes()` = tipos dos
  parâmetros, em ordem; `getReturnType()` = tipo de retorno (`VOID` para procedimento).

## Onde ficam as regras (rótulos do papel → código)

| Rótulo | Onde |
|---|---|
| `[LIT-*]` | `TypeChecker.visitIntLit … visitStringLit` |
| `[ID]`, `[ID-NDECL]`, `[ID-FUNC]` | `TypeChecker.visitIdent` |
| `[ARIT]`, `[REL]`, `[IGUAL-NUM]`, `[IGUAL-BOOL]` | `TypeChecker.visitBinOp` → `TabelaOperadores.binario` |
| `[UN-NUM]`, `[UN-NAO]` | `TypeChecker.visitUnaryOp` → `TabelaOperadores.unario` |
| relação `U <: T` (cabe em) | `TabelaOperadores.compativel` |
| `[ATRIB]`, `[INIT]` | `TypeChecker.visitAssignStmt`, `visitVarDecl` |
| `[IF]`, `[WHILE]`, `[BLOCO]` | `TypeChecker.visitIfStmt`, `visitWhileStmt`, `visitBlock` |
| `[RET]`, `[RET-PROC]` | `TypeChecker.visitReturnStmt` |
| `[FUNC-RET]`, `[RT-*]` (Decisão 9) | `TypeChecker.visitFuncDecl` + `RetornoVisitor` |
| `[CALL]`, `[CALL-PROC]`, `[CALL-CMD]`, `[CALL-VAR]` | `TypeChecker.conferirChamada`, `visitCallExpr`, `visitCallStmt` |

O `TypeChecker` é um `Visitor<TypeKind>`: sem `instanceof`, sem `switch` sobre nós, e sem `declare`/`openScope`/`closeScope`
(o `TypeCheckTest` confere isso lendo o código-fonte). O `switch` sobre *operadores* mora só em `TabelaOperadores`.

**Mudanças na AST/CP4 feitas para o CP5** (todas pequenas; as suítes do CP2, CP3 e CP4 continuam passando):

1. `TypeKind` ganhou `VOID` e `ERRO`.
2. `Node` ganhou o campo `tipo` — a mudança de “tipo nas expressões”.
3. `AssignStmtNode`, `CallStmtNode` e `CallExprNode` ganharam o campo `decl` (`Symbol`). No CP4 esses nós só guardavam o **nome**
   (`String`): a identificação conferia que o nome existia, mas descartava o `Symbol`. O verificador precisa da assinatura da rotina,
   então a passada de identificação agora grava o `Symbol` (sem mudar nenhuma mensagem do CP4).
4. `Symbol` ganhou `paramTypes` e `getReturnType()`; a `IdentificationVisitor` preenche `paramTypes` ao declarar funções e procedimentos.

Arquivos novos: `src/tipos/TabelaOperadores.java`, `src/visitor/TypeChecker.java`, `src/visitor/RetornoVisitor.java`,
`src/Analise.java` (pipeline scanner → parser → identificação → tipos, usado pelo `Main` e pelos testes).
Os arquivos do CP1 (`Scanner`, `Token`, `TipoToken`) não foram tocados (SHA256 em `cp1-sha256.txt`, conferido pelo `verify_all.py`).

## Formato das mensagens

Uma por linha, na ordem do texto; o verificador acumula todas e continua. Erros de identificação (CP4) vêm antes dos de tipos.

```
linha L: operador '+' não se aplica a int e bool          linha L: operador '!' não se aplica a int
linha L: condição de 'if' deve ser bool, mas é int        linha L: condição de 'while' deve ser bool, mas é string
linha L: atribuição a 'x': esperado int, veio double      linha L: inicializador de 'x': esperado int, veio string
linha L: return em 'f': esperado int, veio bool           linha L: return em 'g': esperado string, mas não há valor
linha L: return em procedimento 'p' não deve ter valor    linha L: função 'f' pode terminar sem return
linha L: 'fat' espera 1 argumento(s), recebeu 2           linha L: argumento 1 de 'mostra': esperado string, veio int
linha L: 'x' não é função                                 linha L: procedimento 'p' não devolve valor e não pode ser usado em expressão
linha L: 'f' é função e não pode ser usada como valor (faltam parênteses)
linha L: 'p' é procedimento e não pode ser usado como valor
linha L: 'f' não é variável e não pode receber atribuição
aviso linha L: variável 'x' pode ser usada sem valor      (aviso: não bloqueia a compilação)
```

**Sem cascata**: depois do primeiro erro o tipo `ERRO` sobe pela árvore e não gera mensagem nova; nome não declarado vira `ERRO`
(o CP4 já reportou). A linha de “pode terminar sem return” é a da declaração da função (a AST não guarda a linha do `}`).

**Variáveis sem valor** (aviso, análise conservadora): só variáveis locais declaradas sem inicializador. Ao fim de um `if` sem `else`,
de um `while` ou de um `if/else` em que algum caminho não atribui, a variável continua “possivelmente sem valor”. Globais não são
rastreadas (qualquer rotina pode atribuí-las; não há análise entre rotinas).

## As 11 decisões de projeto (resumo; o papel tem escolha, alternativa descartada e motivo)

| # | Decisão | Escolha | Teste |
|---|---|---|---|
| 1 | int → double | implícito (atribuição, inicializador, argumento, return, operadores); double → int nunca | `dec01_int_double` |
| 2 | char com int | char é numérico; `char op int` = int; int → char nunca implícito (nem `char m = 65;`) | `dec02_char_int` |
| 3 | bool em aritmética | não; logo int não é condição | `dec03_bool_aritmetica` |
| 4 | string | só atribuição/argumento/return; sem `==`, `!=`, `<`, `+` | `dec04_string` |
| 5 | `==` entre tipos | num×num e bool×bool; o resto é erro | `dec05_igualdade` |
| 6 | função como comando | descarta o valor | `dec06_funcao_como_comando` |
| 7 | recursão mútua | não (passada única, como no CP4) | `dec07_recursao_mutua` |
| 8 | argumentos | mesmo alargamento da atribuição | `dec08_argumento_int_double` |
| 9 | return em todos os caminhos | análise de fluxo simples e conservadora | `dec09_return_todos_caminhos` |
| 10 | mesmo nome, outra assinatura | proibido | `dec10_mesmo_nome` |
| 11 | função sem parênteses | erro | `dec11_funcao_sem_parenteses` |

Representação na TAM (só registro, nada implementado): int/bool/char = 1 palavra; double = 2 palavras; string = 1 palavra com a
referência da sequência de caracteres; conversão int → double inserida pelo gerador onde o tipo gravado no nó é int/char e o contexto
espera double. A confirmar com a TAM usada em aula.

## O que a smollang não tem (regras que não existem)

Operadores `&&` e `||` (o único lógico é o unário `!`), `break`, `print`, `read`, funções aninhadas, conversão explícita e
declaração antecipada. Por isso não há regras para eles, nem pilha de laços no contexto do verificador. O análogo de “função aninhada
usando variável do nível acima” é testado em `func_usa_global` (global dentro de função e variável da função dentro de bloco aninhado).

## Testes

- `OperatorTableTest` — as regras isoladas, sem parser: cada linha da tabela de operadores (aceita e recusada; 5×5 tipos para cada
  grupo de operador), a relação “cabe em” e o tipo de erro. Tabelas escritas à mão no teste.
- `TypeCheckTest` — cada `test/programas/X.txt` roda o pipeline real e é comparado com `test/esperado/X.txt` (saídas esperadas escritas
  pelo grupo a partir das regras do papel, não copiadas da saída do verificador; `sem mensagens` = programa limpo). Confere também que
  erro ⇒ REJEITADO e aviso ⇒ ACEITO, os tipos gravados nos nós, a robustez (entradas malformadas, 50 000 somas, 3 000 parênteses
  aninhados: nenhuma exceção, nenhum laço infinito) e a ausência de `instanceof`/`switch` no verificador.
- Programas obrigatórios: `valido_todos_tipos`, `erro_expr`, `cond_int` (+ `cond_while`), `return_incompativel`, `chamada_aridade`,
  `chamada_tipo`, `proc_como_expr`, `varios_erros`, `sem_cascata_nome`, `aviso_sem_valor`; funções: `func_recursiva`, `dec07_recursao_mutua`,
  `func_usa_global`; mais `dec01` … `dec11`.
- Regressão: `SymbolTableTest` (24), `IdentificationTest` (24), `ParserTest` (35), `AstTest` (15) — os mesmos programas do CP2–CP4,
  em `programas/` e `test/esperado/`. Os nomes dos arquivos do CP5 não colidem com os deles.
- `test/verification.log` — saída da última execução de `tools/verify_all.py`.
