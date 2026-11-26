# Log de uso de IA — CP5

Todo uso de IA nesta etapa está registrado abaixo. Esta entrega foi produzida com assistência **substancial** de IA a pedido de
Giovana; não deve ser descrita como implementação independente. A parte 1 (`tipos-a-mao.pdf`) tem de ser escrita à mão pela aluna: o
PDF digitado `tipos-a-mao-RASCUNHO.pdf` foi gerado por IA apenas como material de estudo para ser copiado e entendido.

| Data | Ferramenta | Solicitação e uso efetivo | Relação com o enunciado |
|---|---|---|---|
| 2026-11-25 | Claude (Anthropic), chat | Leitura do enunciado do CP5 e dos zips do CP1 a CP4 (AST, tabela de símbolos, passada de identificação, testes e logs). Projeto do verificador: tipos `VOID`/`ERRO`, campo `tipo` em `Node`, `decl` em nós de uso, assinatura no `Symbol`. | Planejamento e leitura de código existente. |
| 2026-11-25 | Claude (Anthropic), chat | Implementação de `TabelaOperadores`, `TypeChecker`, `RetornoVisitor`, `Analise` e da integração no `Main` e na `IdentificationVisitor`. Escolha das 11 decisões de projeto (propostas e registradas com alternativa e motivo). | Geração de código; excede o uso de IA como apoio pontual. |
| 2026-11-25 | Claude (Anthropic), chat | Escrita dos programas de teste, das saídas esperadas (derivadas das regras do papel; erros de contagem de linha corrigidos conferindo cada arquivo), de `OperatorTableTest` e `TypeCheckTest`, e atualização de `tools/verify_all.py`. | Testes. |
| 2026-11-25 | Claude (Anthropic) com execução de Java/Python no ambiente do chat | Instalação de JDK no ambiente do chat, compilação com `-Werror`, execução de todas as suítes (CP5 e regressões CP2–CP4), testes de mutação (8 defeitos inseridos de propósito, todos detectados) e correção de robustez a entradas muito profundas (pilha). Saída em `test/verification.log`. | Validação automatizada. |
| 2026-11-25 | Claude (Anthropic), chat | `README.md` e `tipos-a-mao-RASCUNHO.pdf` (regras, tabela de operadores, 11 decisões e registro da TAM digitados). | Documentação; o PDF **não** substitui a parte à mão. |

Nenhum código foi enviado a serviços externos além do próprio chat. Os arquivos léxicos do CP1 (`Scanner.java`, `Token.java`,
`TipoToken.java`) não foram alterados (SHA256 conferido em `tools/verify_all.py`). As decisões de projeto são escolhas desta versão;
não há evidência de que reflitam o que a aluna decidiria sozinha.

Revisão e entendimento do material pela aluna: *a preencher por Giovana antes de entregar.*
