
---

# 🧪 Lab 1 — Introducción a ANTLR (Análisis)

Esta sección documenta el desarrollo del **Laboratorio 1**, ubicado en la carpeta [`lab-1/`]

## ▶️ Cómo se ejecutó

Enlace al video: https://youtu.be/3RUHrsLA_04 

### Resultados de las pruebas

**✅ Entrada válida** (`program_test.txt`): no imprime nada y termina con código de salida `0`.

```text
5 * 5
a = 4
b = 6
c = a + b
```

**❌ Entrada inválida** (`program_bad.txt`, creado para demostrar el caso de error): ANTLR reporta los errores de sintaxis en consola.

```text
line 1:4 extraneous input '*' expecting {'(', ID, INT}
line 2:4 extraneous input '=' expecting {'(', ID, INT}
line 3:2 missing NEWLINE at '4'
line 3:5 mismatched input '\n' expecting {'(', ID, INT}
```

---

## Análisis de la gramática `MiniLang.g4`

### Secciones de un archivo `.g4`

Un archivo de gramática ANTLR (`.g4`) se compone típicamente de:

1. **Declaración de la gramática:** `grammar MiniLang;`. El nombre **debe coincidir** con el del archivo (`MiniLang.g4`). Existen tres tipos: `grammar` (combinada, lexer + parser en un solo archivo, como aquí), `lexer grammar` y `parser grammar` (separadas).
2. **Reglas del parser (sintácticas):** empiezan con **minúscula** (`prog`, `stat`, `expr`). Describen la estructura gramatical a partir de tokens.
3. **Reglas del lexer (léxicas / tokens):** empiezan con **MAYÚSCULA** (`ID`, `INT`, `NEWLINE`, `WS`). Convierten los caracteres del texto en tokens.

Esta convención de **mayúscula/minúscula** es la forma en que ANTLR distingue entre reglas de lexer y de parser.

### Reglas del parser

```antlr
prog:   stat+ ;

stat:   expr NEWLINE                 # printExpr
    |   ID '=' expr NEWLINE          # assign
    |   NEWLINE                      # blank
    ;

expr:   expr ('*'|'/') expr          # MulDiv
    |   expr ('+'|'-') expr          # AddSub
    |   INT                          # int
    |   ID                           # id
    |   '(' expr ')'                 # parens
    ;
```

- **`prog: stat+ ;`** — La regla inicial. Un programa es **una o más** sentencias (`stat`). El `+` significa "uno o más" (cuantificador); `*` sería "cero o más" y `?` "opcional".
- **`|` (barra vertical)** — Separa **alternativas** de una misma regla. Una `stat` puede ser una expresión seguida de salto de línea, una asignación, o una línea en blanco.
- **`# etiqueta` (numeral/almohadilla)** — El `#` asigna una **etiqueta a cada alternativa** de una regla (por ejemplo `# printExpr`, `# assign`, `# MulDiv`). ANTLR genera un método distinto por etiqueta en el Visitor/Listener (p. ej. `visitMulDiv`, `visitAssign`), lo que permite manejar cada caso por separado en fases posteriores (análisis semántico, evaluación). **Regla:** si se etiqueta una alternativa, se deben etiquetar **todas** las de esa regla.
- **Literales entre comillas simples** (`'='`, `'*'`, `'('`) — Son **tokens implícitos**: ANTLR crea automáticamente un token para cada cadena literal usada en las reglas del parser.
- **Recursión y precedencia** — `expr` es **recursiva por la izquierda** (`expr ('*'|'/') expr`). ANTLR 4 soporta recursión izquierda directa y resuelve la **precedencia según el orden** en que aparecen las alternativas: como `MulDiv` está **antes** que `AddSub`, la multiplicación/división tiene **mayor precedencia** que la suma/resta. La asociatividad por defecto es a la **izquierda**.

### Reglas del lexer (tokens)

```antlr
MUL : '*' ;                 // token para multiplicación
DIV : '/' ;                 // token para división
ADD : '+' ;                 // token para suma
SUB : '-' ;                 // token para resta
ID  : [a-zA-Z]+ ;           // identificadores
INT : [0-9]+ ;              // enteros
NEWLINE:'\r'? '\n' ;        // salto de línea (fin de sentencia)
WS  : [ \t]+ -> skip ;      // descarta espacios y tabs
```

- **`[a-zA-Z]`, `[0-9]`** — **Conjuntos/rangos de caracteres**, similares a expresiones regulares. `ID` es una o más letras; `INT` uno o más dígitos.
- **`'\r'? '\n'`** — `NEWLINE` reconoce el salto de línea, con el retorno de carro `\r` **opcional** (`?`) para compatibilidad entre Windows (`\r\n`) y Unix (`\n`). Es el token que marca el **fin de sentencia**.
- **`-> skip`** — Es una **acción/comando del lexer** que hace que el token se **descarte** (no se envía al parser). Se usa en `WS` para ignorar espacios y tabulaciones, de modo que no interfieran con la gramática. Otros comandos comunes son `-> channel(HIDDEN)` (oculta el token pero lo conserva) y `-> more`/`-> type(...)`.
- **`//` comentarios** — ANTLR admite comentarios de línea (`//`) y de bloque (`/* */`).
- **Resolución de ambigüedad léxica** — Cuando dos reglas pueden coincidir, el lexer elige la de **coincidencia más larga**; en empate, gana la **regla definida primero** en el archivo.

---

## Análisis del `Driver.py`

```python
import sys
from antlr4 import *
from MiniLangLexer import MiniLangLexer
from MiniLangParser import MiniLangParser

def main(argv):
    input_stream = FileStream(argv[1])
    lexer = MiniLangLexer(input_stream)
    stream = CommonTokenStream(lexer)
    parser = MiniLangParser(stream)
    tree = parser.prog()   # 'prog' es la regla inicial de la gramática

if __name__ == '__main__':
    main(sys.argv)
```

El driver es el **punto de entrada** que conecta las clases que ANTLR generó y ejecuta la **cadena de análisis** (pipeline) clásica de un compilador:

1. **`import` de las clases generadas** — `MiniLangLexer` y `MiniLangParser` son producidos por el comando `antlr -Dlanguage=Python3 MiniLang.g4`. `from antlr4 import *` trae el *runtime* de ANTLR para Python (`FileStream`, `CommonTokenStream`, etc.).
2. **`FileStream(argv[1])`** — Lee el archivo de entrada (el que se pasa como argumento en la línea de comandos) y lo convierte en un flujo de caracteres.
3. **`MiniLangLexer(input_stream)`** — El **análisis léxico**: transforma el flujo de caracteres en tokens según las reglas de MAYÚSCULA de la gramática.
4. **`CommonTokenStream(lexer)`** — Buffer intermedio que almacena los tokens producidos por el lexer y los entrega al parser bajo demanda.
5. **`MiniLangParser(stream)`** — El **análisis sintáctico**: consume los tokens y verifica que sigan la estructura definida por las reglas del parser.
6. **`parser.prog()`** — Invoca la **regla inicial** (`prog`) y construye el **árbol de análisis sintáctico** (*parse tree*). Como el driver no imprime nada, si la entrada es válida **no hay salida**; si no lo es, el `ErrorListener` por defecto de ANTLR imprime los errores en consola (como se vio en la prueba inválida).

---
