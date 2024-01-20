
--- 3.0 ---

В этой главе мы создадим грамматику и приложение для чтения для выражений, типа

```
{1, 2, 3} {1, {2, 3}, 4}
```

--- 3.1 ---

У ANTLR есть два ключевых компонента: сам инструмент ANTLR4(класс org.antlr.v4.Tool) и библиотека времени выполнения. Когда мы запускаем ANTLR4, он генерирует код для парсера и лексера, который распознаёт **предложения**(sentences) описанной грамматики. Сама библиотека состоит из классов и методов, необходимых сгенерированному коду, например: Parser, Lexer и Token. Уже сгенерированный код мы компилируем с этой библиотекой и получаем желанный jar.

Есть такая грамматика(файл ArrayInit.g4):

```
grammar ArrayInit;

init : '{' value (',' value)* '}' ; value : init | INT ;

INT : [0-9]+ ; WS : [ \t\r\n]+ -> skip
```

Мы всегда начинаем файл с грамматикой с заголовка grammar и имени файла после(без расширения). Имена правил для парсера имеют нижний регистр, а для лексера - верхний.

Компиляция: 

```
$ antlr4 ArrayInit.g4 


Получаем:
* ArrayInitParser.java - содержит класс парсера ArrayInitParser для нашей грамматики, который расширяет класс Parser и содержит методы для каждого правила в грамматике.

* ArrayInitLexer.java - содержит класс лексера ArrayInitLexer, который расширяет класс Lexer и генерируется на основе лексических правил INT и WS и литералов, типа '{'.

* ArrayInitListener.java - содержит интерфейс с методами, которые вызываются по ходу обхода дерева.

* ArrayInitBaseListener.java - содержит реализации по умолчанию для методов слушателя, чтобы мы могли переопределять только то, что нам интересно.

* ArrayInitLexer.tokens - 
* ArrayInit.tokens - ANTLR4 присваивает каждому типу токена уникальный номер и помещает это всё в файлы. Это необходимо для синхронизации, когда мы разделяем большую грамматику на множество файлов.

Если мы ходим, чтобы у нас для обхода дерева использовался визитор, то нужно использовать опцию -visitor: 

```
$ antlr4 ArrayInit -visitor
```

В таком случаемы мы помимо упомянутых ранее файлов получим:

```
ArrayInitBaseVisitor.java
ArrayInitVisitor.java
```

--- 3.2 ---

Теперь можно компилировать:

```
$ javac *.java 
```
После можно запускать grun: 

```
$ grun ArrayInit init -tokens
{99, 3, 451}
```

Получаем такой вывод: 

```
[@0,0:0='{',<1>,1:0]
[@1,1:2='99',<4>,1:1]
[@2,3:3=',',<2>,1:3]
[@3,5:5='3',<4>,1:5]
[@4,6:6=',',<2>,1:6]
[@5,8:10='451',<4>,1:8]
[@6,11:11='}',<3>,1:11]
[@7,13:12='',<-1>,2:0]
```

Каждая строка представляет токен и всё, что бы о нём знаем: [@5,8:10='451',<4>,1:8], где @5 - индекс токена(начиная в 0) 8:10='451' - позиции символов от и до(начиная с 0) и сам токен <4> - тип токена 1:8 - строка 1(начиная с 1) и начальная позиция 8

Можно запускать grun и получить дерево: 

```
$ grun ArrayInit init -tree
{99, 3, 451}
```

Получаем такой вывод в стиле Лиспа: 

```
(init { (value 99) , (value 3) , (value 451) })
```

Можно запускать grun и получить дерево на картинке: 

```
$ grun ArrayInit init -gui
{99, 3, 451}
```

Получаем такой вывод:

![Screenshot_16](https://github.com/EvgenyPrudilov/ANTLR4-notes/assets/123429404/e8d60dbc-fcaf-43f0-a459-d4f0959f247f)

--- 3.3 ---

Мы можем интегрировать сгенерированный код в наше приложение. Попробуем сами написать TestRig -tree опцию. Код приложения(класс Test.java):

```
import org.antlr.v4.runtime.; import org.antlr.v4.runtime.tree.;

public class Test { 
    public static void main(String[] args) throws Exception { 
        ANTLRInputStream input = new ANTLRInputStream(System.in);
        ArrayInitLexer lexer = new ArrayInitLexer(input);
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        ArrayInitParser parser = new ArrayInitParser(tokens);
        
        ParseTree tree = parser.init();
        System.out.println(tree.toStringTree(parser));
    }
}
```

Компилируем и запускаем:

```
$ javac ArrayInit*.java Test.java 
$ java Test 
{1, {2, 3}, 4} 
```

Получаем:

```
(init { (value 1) , (value (init { (value 2) , (value 3) })) , (value 4) })
```

Если мы где-то допустим синтаксическую ошибку, ANTLR4 парсер нам об этом с радостью сообщит.

--- 3.4 ---

Допустим, мы хотим преобразовывать массив целых чисел {99, 3, 451} в строку с юникод символами "\u0063\u0003\u01c3". Для этого мы можем проходя парсером по массиву вызывать колбэки в слушателе. Для этого нам нужно переопределить нужные методы в подклассе класса ArrayInitBaseListener. Нам нужно понять КАК транслировать входной токен или фразу в выходную строку.

Код для методов(класс ShortToUnicodeString.java):

```
public class ShortToUnicodeString extends ArrayInitBaseListener {
    @Override
    public void enterInit(ArrayInitParser.InitContext ctx) {
        System.out.print('"');
    }
    @Override
    public void exitInit(ArrayInitParser.InitContext ctx) {
        System.out.print('"');
    }
    @Override
    public void enterValue(ArrayInitParser.ValueContext ctx) {
        int value = Integer.valueOf(ctx.INT().getText());
        System.out.printf("\\u%04x", value);
    }
}
```

Как видно, мы не переопределяем каждый метод - переопределяем только то, что нужно. Выражение ctx.INT() возвращает токен для соответствующего числа в правиле value.

Напишем код для транслятора(класс Translate.java):

```
import org.antlr.v4.runtime.; import org.antlr.v4.runtime.tree.;

public class Translate { 
    public static void main(String[] args) throws Exception { 
        ANTLRInputStream input = new ANTLRInputStream(System.in); 
        ArrayInitLexer lexer = new ArrayInitLexer(input); 
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        
        ArrayInitParser parser = new ArrayInitParser(tokens);
        ParseTree tree = parser.init();
        
        ParseTreeWalker walker = new ParseTreeWalker();
        walker.walk(new ShortToUnicodeString(), tree);
        System.out.println();
    }
}
```

Здесь мы создаём walker для обхода дерева, и просим его сделать это, по ходу дела вызывая колбэки из ShortToUnicodeString(здесь определены только нужные методы, а остальные мы наследуем от ArrayInitBaseListener).

Теперь осталось всё это вместе скомпилиторать:

```
$ javac ArrayInit*.java Translate.java 
$ java Translate {99, 3, 451} 
```

Получим:

```
"\u0063\u0003\u01c3"
```

Важно: мы можем теперь получить любой другой вывод, просто подменив другого слушателя(наследника ArrayInitBaseListener) - грамматику нам трогать не нужно. Т.е. образом слушатели как бы изолируют приложение от грамматики, давая другим приложения возможность переиспользовать эту самую грамматику.
