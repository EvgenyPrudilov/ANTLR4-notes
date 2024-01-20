
--- 4.1 ---

Сделаем язык для арифметических выражений, т.е. простой калькулятор для целых чисел, в котором будут: сложение, вычитание, умножение, деление, числа, переменные и можно использовать скобки. Например:

```
193 
a = 5 
b = 6 
a + b * 2 
(1 + 2) * 3
```

У нас здесь программа представляет собой предложения, разделённые новой строкой. Предложение - это выражение, присваивание или пустая строка. Грамматика:

```
grammar Expr;

prog: stat+ ;

stat: expr NEWLINE 
    | ID '=' expr NEWLINE 
    | NEWLINE 
    ;

expr: expr ('*' | '/') expr 
    | expr ('+' | '-') expr 
    | INT 
    | ID 
    | '(' expr ')' 
    ;

ID  : [a-zA-Z]+ 
    ; 
INT : [0-9]+ 
    ; 
NEWLINE
    : '\r'? '\n' 
    ; 
WS  : [ \t]+ -> skip 
    ;
```

Если кратко(детально всё будет рассматриваться в следующей главе):
* Имена правил парсера записываются в нижнем регистре.
* Имена правил лексера записываются в верхнем регистре.
* с помощью оператора -> мы указываем, что лексеру выбросить найденные совпадения(в данном случае все пробельные символы).
* Альтернативы правила записываются через символ |. А с помощью скобок мы можем создавать подправила, например: ('+' | '-').
* Особенность ANTLR четверной версии - способность правильно обрабатывать **правила с явной левой рекурсией**(left-recursive rules) - это такое правила, которые ссылаются на себя с левого края альтернативы.

Компилируем и запускаем на тестирование: 

```
$ antlr4 Expr.g4
$ javac Expr*.java
$ grun Expr prog -tree
1 1 + 2
a = 3
b = a - 4
c = a * b
```

Получаем: 

```
(prog (stat (expr 1) \n)
      (stat (expr (expr 1) + (expr 2)) \n)
      (stat a = (expr 3) \n)
      (stat b = (expr (expr a) - (expr 4)) \n)
      (stat c = (expr (expr a) * (expr b)) \n) )
```

Теперь пишем программу:

```
import org.antlr.v4.runtime.;
import org.antlr.v4.runtime.tree.;
import java.io.FileInputStream;
import java.io.InputStream;

public class ExprJoyRide {
  public static void main(String[] args) throws Exception {
    String inputFile = null;
    if (args.length > 0)
      inputFile = args[0];
    InputStream is = System.in;
    if (inputFile != null)
      is = new FileInputStream(inputFile);

    ANTLRInputStream input = new ANTLRInputStream(is);
    ExprLexer lexer = new ExprLexer(input);
    CommonTokenStream tokens = new CommonTokenStream(lexer);
    ExprParser parser = new ExprParser(tokens);
    ParseTree tree = parser.prog();
    System.out.println(tree.toStringTree(parser));
  }
}
```

Компилируем и запускаем:

```
$ javac ExprJoyRide.java Expr*.java
$ java ExprJoyRide
1 1 + 2
a = 3
b = a - 4
c = a * b
```

Получаем тоже самое, что и раньше: 

```
(prog (stat (expr 1) \n)
      (stat (expr (expr 1) + (expr 2)) \n)
      (stat a = (expr 3) \n)
      (stat b = (expr (expr a) - (expr 4)) \n)
      (stat c = (expr (expr a) * (expr b)) \n) )
```

Мы можем разбивать большие грамматики на логические части. Например, можно разделить грамматику для парсера и лексера. Это может быть полезно, т.к. очень много лексических фрагментов встречаются в большинстве языков: идентификаторы, числа, ... - вынеся такие структуры отдельно в файл, его можно будет переиспользовать в других грамматиках. Пример:

```
lexer grammar CommonLexerRules;

ID  : [a-zA-Z]+
    ;
INT : [0-9]+
    ;
NEWLINE
    : '\r'? '\n'
    ;
WS  : [ \t]+ -> skip
    ;
```

Теперь это можно использовать в нашей грамматике. С помощью import мы можем включить все правила из **модуля**(module) CommonLexerRules.g4(это делается автоматически во время вызова antlr4, т.е. порядок компиляции и тестирования не меняется):

```
grammar Expr;
import CommonLexerRules;

prog: stat+ ;

stat: expr NEWLINE
    | ID '=' expr NEWLINE
    | NEWLINE
    ;

expr: expr ('*' | '/') expr
    | expr ('+' | '-') expr
    | INT
    | ID
    | '(' expr ')'
    ;
```

Парсер автоматически информирует нас о синтаксическом ошибке, а также самостоятельно **восстанавливается**(recover) после неё - это нужно для того, чтобы он смог дальше обрабатывать поток токенов:

```
$ java ExprJoyRide (1 + 2 3
```

Получаем:

```
line 1:4 mismatched input '\n' expecting {'/', ')', '*', '+', '-'}
(prog (stat (expr ( (expr (expr 1) + (expr 2)) <missing ')'>) \n)
      (stat (expr 3) \n) )
```

В ANTLR4 механизм ошибок очень гибкий - мы можем менять текст сообщений, можем перехватывать ошибка, и даже менять стратегию обработки ошибок(обсуждается в 9 главе).

--- 4.2 ---

Теперь сделаем так, чтобы калькулятор считал. Реализация будет использовать **паттерн Визитор**(visitor). Но сначала нужно внести изменения в грамматику - пометить альтернативы **меткой**(label), при этом она не должна конфликтовать с именами правил. Если не добавлять метки, то ANTLR4 будет генерировать только один визитор-метод для одного правила. Метки ставятся справа от альтернатив с помощью символа # :

```
stat: expr NEWLINE              # printExpr
    | ID '=' expr NEWLINE       # assign
    | NEWLINE                   # blank
    ;

expr: expr op=('*' | '/') expr  # MulDiv
    | expr op=('+' | '-') expr  # AddSub
    | INT                       # int
    | ID                        # id
    | '(' expr ')'              # parens
    ;
```

Теперь можем определить токены для литералов, чтобы потом можно было ссылаться на них как на константы визиторе:

```
MUL : '*';
DIV : '/' ;
ADD : '+' ;
SUB : '-' ;
```

Нам нужно реализовать визитора, который вычисляет и возвращает значения по ходу обохода дерева. Сначала сгенерируем визитора:

```
$ antlr4 -no-listener -visitor LabeledExpr.g4
```

В итоге мы получаем интерфейс с методами для каждой помеченной альтернативы:

```
public interface LabeledExprVisitor { T visitId(LabeledExprParser.IdContext ctx); T visitAssign(LabeledExprParser.AssignContext ctx); T visitMulDiv(LabeledExprParser.MulDivContext ctx); ... }
```

Также сгенерировался класс с реализацией по умолчанию LabeledExprBaseVisitor, от которого мы будем наследоваться как LabeledExprBaseVisitor и переопределять методы, необходимые для вычисления выражений в нашем калькуляторе. Сам код:

import java.util.HashMap; import java.util.Map;

public class EvalVisitor extends LabeledExprBaseVisitor { Map<String, Integer> memory = new HashMap<String, Integer>();

@Override
public Integer visitAssign(LabeledExprParser.AssignContext ctx) {
    String id = ctx.ID().getText();
    int value = visit(ctx.expr());
    memory.put(id, value);
    return value;
}

@Override
public Integer visitPrintExpr(LabeledExprParser.PrintExprContext ctx) {
    Integer value = visit(ctx.expr());
    System.out.println(value);
    return 0;
}

@Override
public Integer visitInt(LabeledExprParser.IntContext ctx) {
    return Integer.valueOf(ctx.INT().getText());
}

@Override
public Integer visitId(LabeledExprParser.IdContext ctx) {
    String id = ctx.ID().getText();
    if (memory.containsKey(id)) return memory.get(id);
    return 0;
}

@Override
public Integer visitMulDiv(LabeledExprParser.MulDivContext ctx) {
    int left = visit(ctx.expr(0));
    int right = visit(ctx.expr(1));
    if (ctx.op.getType() == LabeledExprParser.MUL)
        return left * right;
    return left / right;
}

@Override
public Integer visitAddSub(LabeledExprParser.AddSubContext ctx) {
    int left = visit(ctx.expr(0));
    int right = visit(ctx.expr(1));
    if (ctx.op.getType() == LabeledExprParser.ADD)
        return left + right;
    return left - right;
}

@Override
public Integer visitParens(LabeledExprParser.ParensContext ctx) {
    return visit(ctx.expr());
}
}

Компилируем и тестируем: $ antlr4 -no-listener -visitor LabeledExpr.g4 $ javac Calc.java LabeledExpr*.java $ cat > t.expr 193 a = 5 b = 6 a+b*2 (1+2)*3 $ java Calc t.expr

Получаем: 193 17 9

Основная мысль - нам не обязательно внедрять код Java в грамматику(как это было в ANTLR третьей версии).

--- 4.3 ---

Допустим есть задача: сделать инструмент для генерации файла с интерфейсом из какого-то класса, а также оставить комментарии в пределах сигнатур методов. У нас есть файл с грамматикой Java - Java.g4 . Сначала распарсим исходный код, потом возьмём имя для интерфейса из имени класса, а сигнатуры методов из определений методов.

Будем использовать паттерн слушателя - при компиляции грамматики он формируется автоматически(JavaListener), а нам в данном случае нужно будет лишь переопределить три метода(первые три перечисленные):

public interface JavaListener extends ParseTreeListener { void enterClassDeclaration(JavaParser.ClassDeclarationContext ctx); void exitClassDeclaration(JavaParser.ClassDeclarationContext ctx); void enterMethodDeclaration(JavaParser.MethodDeclaration ctx); ... }

Главное отличие визитора и слушателя - методы визитора должны явно обойти своих детей с помощью visit метода, а методы слушателя вызываются автоматически при обходе дерева.

Переопределяем нужные методы:

import org.antlr.v4.runtime.TokenStream; import org.antlr.v4.runtime.misc.Interval;

public class ExtractInterfaceListener extends JavaBaseListener { JavaParser parser;

public ExtractInterfaceListener(JavaParser parser) {
    this.parser = parser;
}

@Override
public void enterClassDeclaration(JavaParser.ClassDeclarationContext ctx) {
    System.out.println("interface I" + ctx.Identifier() + " {");
}

@Override
public void exitClassDeclaration(JavaParser.ClassDeclarationContext ctx) {
    System.out.println("}");
}

@Override
public void enterMethodDeclaration(JavaParser.MethodDeclarationContext ctx) {
    TokenStream tokens = parser.getTokenStream();
    String type = "void";
    if (ctx.type() != null) {
        type = tokens.getText(ctx.type());
    }

    String args = tokens.getText(ctx.formalParameters());
    System.out.println("\t" + type + " " + ctx.Identifier() + args + ";");
}
}

Код главной программы:

import org.antlr.v4.runtime.; import org.antlr.v4.runtime.tree.; import java.io.FileInputStream; import java.io.InputStream;

public class ExtractInterfaceTool { public static void main(String[] args) throws Exception { String inputFile = null; if (args.length > 0) inputFile = args[0]; InputStream is = System.in; if (inputFile != null) is = new FileInputStream(inputFile);

    ANTLRInputStream input = new ANTLRInputStream(is);
    JavaLexer lexer = new JavaLexer(input);
    CommonTokenStream tokens = new CommonTokenStream(lexer);
    JavaParser parser = new JavaParser(tokens);
    ParseTree tree = parser.compilationUnit();

    ParseTreeWalker walker = new ParseTreeWalker();
    ExtractInterfaceListener extractor = new ExtractInterfaceListener(parser);
    walker.walk(extractor, tree);
    
}
}

Компилируем и запускаем с файлом-классом Demo.java: $ antlr4 Java.g4 $ javac Java*.java Extract*.java $ java ExtractInterfaceTool Demo.java

Вывод:

interface IDemo { void f(int x, String y); int[ ] g(/no args/); List<Map<String, Integer>>[] h(); }

На самом деле эта реализация не закончена, т.к. не включает в файл импорты для типов, на которые ссылаются в методах интерфейса.

--- 4.4 ---

Теперь мы напишем программу для вывода определённого столбца из файла с данными, типа:

parrt Terence Parr 101 tombu Tom Burns 020 bke Kevin Edgar 008

В принципе, мы можем даже дерево парсинга не строить, а просто печатать данные по ходу парсинга. Но это требует внедрения кода в грамматику - мы можем внедрять код в грамматику и при компиляции он копируется в код парсера. Для этого нам нужно понимать как это повлияет на парсер и знать куда эти действия поместить.

Для начала:

file : (row NL)+ ; row : STUFF+ ;

Теперь внедрим код. Нам нужно создать конструктор которому мы передадим номер столбца(начиная с 1). Также нам нужны действия в правиле row - подробности синтаксиса будут обсуждаться в следующих разделах:

grammar Rows;

@parser::members { int col; public RowsParser(TokenStream input, int col) { this(input); this.col = col; } }

file: (row NL)+ ;

row locals [int i=0] : ( STUFF { $i++; if ($i == col) System.out.println($STUFF.text); } )+ ;

TAB : '\t' -> skip ; NL : '\r'? '\n' ; STUFF : ~[\t\r\n]+ ;

С помощью действия(action) member мы внедряем код в генерируемы класс парсера. С помощью local мы объявляем локальную переменную $i. С помощью $STUFF.text мы получаем текст последнего найденного STUFF токена.

В программе мы передаём колонку как параметр парсера:

import org.antlr.v4.runtime.; import org.antlr.v4.runtime.tree.; import java.io.FileInputStream; import java.io.InputStream;

public class Col { public static void main(String[] args) throws Exception { int col = 1; if (args.length >= 1) col = Integer.valueOf(args[0]) String inputFile = null; if (args.length > 1) inputFile = args[1]; InputStream is = System.in; if (inputFile != null) is = new FileInputStream(inputFile);

    ANTLRInputStream input = new ANTLRInputStream(is);
    RowsLexer lexer = new RowsLexer(input);
    CommonTokenStream tokens = new CommonTokenStream(lexer);
    RowsParser parser = new RowsParser(tokens, col);
    parser.setBuildParseTree(false);
    parser.file();
}
}

С помощью parser.setBuildParseTree(false) мы указываем, что строить дерево не нужно.

Компилируем и запускаем: $ antlr4 -no-listener Rows.g4 $ javac Rows*.java Col.java $ java Col 2 t.rows

Получаем: Terence Parr Tom Burns Kevin Edgar

В ANTLR4 есть семантические предикаты(semantics predicates). Рассмотрим их пользу на примере. Нам нужно прочитать входной поток групп целых чисел, но количество чисел в группе тоже определяется из входного потока, т.е. мы не знаем эти значения заранее. Пример:

2 9 10 3 1 2 3

Получаем: (9 10), (1 2 3)

Семантические предикаты - это булево выражение, например, {$i <= $n}? . В данном случае, оно будет = true, пока i не превысит значение n, заданное в параметре правила sequence. А если оно станет = false, но данная альтернатива завершиться и мы вернёмся из правила sequence. Грамматика:

grammar Data;

file : group+ ;

group : INT sequence[$INT.int] ;

sequence[int n] locals [int i = 1;] : ( {$i <= $n}? INT {$i++;} )* ;

INT : [0-9]+ ; WS : [\t\n\r]+ -> skip ;

Компилируем и тестируем: $ antlr4 -no-listener Data.g4 $ javac Data*.java $ grun Data file -tree t.data

Получаем: (file (group 2 (sequence 9 10)) (group 3 (sequence 1 2 3))

--- 4.5 ---

В ANTLR4 есть три связанные с токенами фичи:

Можно работать с изолированными грамматиками(island grammars) - когда в одном файле мы можем использовать несколько грамматик, т.е. языков. ANTLR4 предоставляет лексические режимы(lexical modes) - суть в том, чтобы переключать из одного режима в другой и обратно при считывании определённых символов.
Например, XML парсер считает текстом всё, кроме того, что находтся в тэгах и кроме символов типа £ . Когда лексер видит символ <, то он переключается во "внутренний" режим, а когда видит символы > или /> - в режим по умолчанию:

lexer grammar XMLLexer;

OPEN : '<' -> pushMode(INSIDE) ; COMMENT : '' -> skip ; EntityRef: '&' [a-z]+ ';' ; TEXT : ~('<' | '&')+ ;

//--------------- mode INSIDE;

CLOSE : '>' -> popMode ; SLASH_CLOSE : '/>' -> popMode ; EQUALS : '=' ; STRING : '"' .? '"' ; SlashName : '/' Name ; Name : ALPHA (ALPHA | DIGIT) ; S : [ \t\r\n] -> skip ;

fragment ALPHA : [a-zA-Z] ;

fragment DIGIT : [0-9] ;

Возьмём такой входной файл:

A parser generator
Компилируем и тестируем: $ antlr4 XMLLexer.g4 $ javac XML*.java $ grun XML tokens -tokens t.xml

Получаем: [@0,0:0='<',<1>,1:0] ... [@9,31:31='>',<5>,2:22] [@10,32:49='A parser generator',<4>,2:23] [@11,50:50='<',<1>,2:41] [@12,51:55='/tool',<9>,2:42] ... [@17,66:66='>',<5>,3:7] [@18,67:66='',<-1>,3:8]

Видно, что мы текст между тегами воспринимаем как нечто единое. Мы указываем начальное правило tokens, чтобы сказать grun, что нужно запускать только лексер.

Можно переписывать входной поток. Допустим, мы хотим внутри каждого класса вставить идентификатор serialVersionUID для использования с java.io.Serializable . Грамматика в файле Java.g4 . Идея - можно вставлять токены в поток токенов, а потом просто вывести этот поток токенов.
Код:

import org.antlr.v4.runtime.; import org.antlr.v4.runtime.tree.; import java.io.FileInputStream; import java.io.InputStream;

public class InsertSerialID { public static void main(String[] args) throws Exception { String inputFile = null; if (args.length > 0) inputFile = args[0]; InputStream is = System.in; if (inputFile != null) is = new FileInputStream(inputFile);

    ANTLRInputStream input = new ANTLRInputStream(is);
    JavaLexer lexer = new JavaLexer(input);
    CommonTokenStream tokens = new CommonTokenStream(lexer);
    JavaParser parser = new JavaParser(tokens);
    ParseTree tree = parser.compilationUnit();

    ParseTreeWalker walker = new ParseTreeWalker();
    InsertSerialIDListener extractor = new InsertSerialIDListener(tokens);
    walker.walk(extractor, tree);
    
    System.out.println(extractor.rewriter.getText());
}
}

Переопределяем только один метод интерфейса слушателя:

import org.antlr.v4.runtime.TokenStream; import org.antlr.v4.runtime.TokenStreamRewriter;

public class InsertSerialIDListener extends JavaBaseListener { TokenStreamRewriter rewriter; public InsertSerialIDListener(TokenStream tokens) { rewriter = new TokenStreamRewriter(tokens); }

@Override
public void enterClassBody(JavaParser.ClassBodyContext ctx) {
    String field = "\n\tpublic static final long serialVersionUID = 1L;";
    rewriter.insertAfter(ctx.start, field);
}
}

Ключевым здесь является объект TokenStreamRewriter - он создаёт вид, будто мы меняем поток токенов(но на самом деле это не так). У него есть методы(они похожи на команды), которые выполняются "лениво" - позже, когда мы вызовем у него метод getText().

Компилируем и тестируем: $ antlr4 Java.g4 $ javac InsertSerialID*.java Java*.java $ java InsertSerialID Demo.java

Получаем:

import java.util.List; import java.util.Map; public class Demo { public static final long serialVersionUID = 1L; void f(int x, String y) { } int[ ] g(/no args/) { return null; } List<Map<String, Integer>>[] h() { return null; } }

Можно направлять токены в разные каналы(channels). Мы можем, например, пробельные символы и комментарии просто игнорировать, но тогда они будут не доступны для последующей обработки. С помощью каналов мы можем их(или что-либо ещё) игнорировать, но сохранять и пользоваться ими позже:
COMMENT : '/' .? '*/' -> channel(HIDDEN) ; WS : [ \t\r\n\u000C]+ -> channel(HIDDEN) ;

Это делается с помощью команды -> channel(HIDDEN)
