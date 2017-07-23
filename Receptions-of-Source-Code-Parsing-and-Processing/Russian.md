# Приемы парсинга и обработки исходного кода

* Теория и практика парсинга
* Преобразование деревьев
* Доклад не совсем про .NET
* Зато имеет прикладное значение, используется в открытых и коммерческих проектах

---

# О себе

* Иван Кочуркин
* Работаю в [Positive Technologies](https://www.ptsecurity.com/ru-ru/) над открытым универсальным сигнатурным анализатором кода [PT.PM](https://github.com/PositiveTechnologies/PT.PM)
* Подрабатываю в [Swiftify](http://swiftify.io/), веб-сервисе для конвертинга кода Objective-C в Swift
* Веду активную деятельность на GitHub:
	* Сделал более [10 Pull Request](https://github.com/antlr/antlr4/pulls?utf8=%E2%9C%93&q=state%3Amerged%20is%3Apr%20author%3AKvanTTT%20) в ANTLR 4
	* Разработал/доработал грамматики PHP, PL/SQL, T-SQL, Objective-C, C#, Java 8 в [grammars-v4](https://github.com/antlr/grammars-v4/pulls?utf8=%E2%9C%93&q=state%3Amerged%20is%3Apr%20author%3AKvanTTT%20)
	* Создал и закрыл много Issues для различных проектов
* Пишу статьи на [Хабре](https://habrahabr.ru/users/kvanttt/) и [GitHub](https://github.com/KvanTTT/Articles) под ником KvanTTT

---

# Парсинг

Процесс преобразования исходного кода в структурированный вид (AST).

* Языки программирования: C#, Java, T-SQL, PL/SQL и т.д.
* Предметно-ориентированный языки DSL

**Лексер** группирует символы исходного кода в значащие последовательности.
**Парсер** из потока токенов строит связную древовидную структуру.

![](https://habrastorage.org/files/6c4/385/fbe/6c4385fbe3d8471982c9b2a030106d38.png)

Существуют безлексерные парсеры.

---

# Парсинг и грамматики

### Типы языков

* Регулярные
* Контекстно-свободные
* Контекстно-зависимые

### Пример грамматики

```
expr
    : expr '*' expr
    | expr '+' expr
    | ID
    ;

ID: [a-zA-Z]+;
```

### Пример данных

```
a + b * c
```

---

# Проблемы и задачи парсинга

* Неоднозначность
* Регистронезависимость
* Островные языки и конструкции
* Парсинг фрагментов кода
* Скрытые токены
* Препроцессорные директивы
* Обработка и восстановление от ошибок
* Использование парсеров в IDE
* Производительность

---

# Неоднозначность

* Ключевые слова как идентификаторы
```CSharp
var var = 0; // Valid code!
```
* Хак лексера ([The lexer hack](https://en.wikipedia.org/wiki/The_lexer_hack))
* Контекстно-зависимые конструкции

---

## Ключевые слова как идентификаторы

### Решение с помощью грамматики

```ANTLR
// Lexer
VAR: 'var';
ID:  [0-9a-zA-Z];

// Parser
varDeclaration
    : VAR identifier ('=' expression)? ';'
    ;

identifier
    : ID
    // Other conflicted tokens
    | VAR;
```

---

## Ключевые слова как идентификаторы

### Решение с использованием вставок кода

```ANTLR
// Lexer
ID:  [0-9a-zA-Z];

// Parser
varDeclaration
    : id {_input.Lt(-1).Type == VAR} id ('=' expression)? ';'
    ;

id
    : ID;
```

---

## The lexer hack. Плохое решение

* Вход: `static MyType staticGlobaVar;`

* Грамматика

```ANTLR
varDeclaration
    : specifiers initDeclaratorList? ';'
    ;
```

* Токены

	* **static** - `specifier`
	* **MyType** - `specifier`
	* **staticGlobaVar** - - `specifier`

* Решение

Визитор: поиск последнего `specifier` и "превращение" его в `initDeclaratorList`.

---

## The lexer hack. Изящное решение

* Вход: `static MyType staticGlobaVar;`

* Грамматика

```ANTLR
varDeclaration
    : (specifiers initDeclaratorList | specifiers) ';'
    ;
```

* Токены

	* **static** - `specifier`
	* **MyType** - `specifier`
	* **staticGlobaVar** - - `initDeclaratorList`

* Решение

Визитор: уже имеем `initDeclaratorList`, т.к. такая альтернатива приоритетней.

---

## Контекстно-зависимые конструкции

Heredoc в PHP или интерполируемые строки в C#

```PHP
<?php
    echo <<< HeredocIdentifier
Line 1.
Line 2.
HeredocIdentifier
;
```

### Решение

Использование вставок кода, смотри лексеры [PHP](https://github.com/antlr/grammars-v4/blob/master/php/PHPLexer.g4) и [C#](https://github.com/antlr/grammars-v4/blob/master/csharp/CSharpLexer.g4).

---

# Регистронезависимость

Нечувствительные к регистру языки: Delphi, T-SQL, PL/SQL и другие.

### Решение с помощью грамматики

```ANTLR
Abstract:           A B S T R A C T;
BoolType:           B O O L E A N | B O O L;
BooleanConstant:    T R U E | F A L S E;

fragment A: [aA];
fragment B: [bB];
```

---

# Регистронезависимость

## Решение на уровне рантайма

```ANTLR
Abstract:           'ABSTRACT';
BoolType:           'BOOLEAN' | 'BOOL';
BooleanConstant:    'TRUE' | 'FALSE';
```

Используется [CaseInsensitiveInputStream](https://gist.github.com/sharwell/9424666).
Чувствительные к регистру токены обрабатываются на этапе обхода дерева. Например `$id` и `$ID` в PHP.

Достоинства:
* Код лексера чище и проще
* Производительность лучше

---

# Островные языки и конструкции

JavaScript внутри PHP или C# внутри Aspx.

```
<?php
<head>
  <script type="text/javascript">
    document.body.innerHTML="<svg/onload=alert(1)>"
  </script>
</head>
```

* Использовать режимы переключения лексем `mode`.
* Парсер **PHP** распознает код внутри тегов `<script>` как строку:
`document.body.innerHTML="<svg/onload=alert(1)>"`
* Парсер **JavaScript** используется во время обхода дерева.
Для него это обычный код + смещение.

---

# Использование другого режима лексем для JavaScript

```
ScriptText:   ~[<]+;
ScriptClose:  '</script'? '>' -> popMode;
PHPStart:     '<?=' -> type(Echo), pushMode(PHP);
PHPStart:     '<?' 'php'? -> channel(SkipChannel), pushMode(PHP);
ScriptText2:  '<' ~[<?/]* -> type(ScriptText);
ScriptText3:  '?' ~[<]* -> type(ScriptText);
ScriptText4:  '/' ~[<]* -> type(ScriptText);
```

---

# Парсинг фрагментов кода

### Задача

Определение корректного правила для фрагмента кода

### Применение

* Конвертинг кода в плагине IDE (в Swiftify для плагинов к AppCode, XCode)
* Определение языка программирования (в PT.PM для редактора [Approof](https://approof.ptsecurity.ru/))

### Решение

* Регулярные выражения
* Токенизация и операции с токенами
* Парсинг разными правилами

--- 

# Скрытые токены

* Включение скрытых токенов в грамматику
```ANTLR
declaration:
    property COMMENT* COLON COMMENT* expr COMMENT* prio?;
```
* Связывание скрытые токенов с правилами грамматики (ANTLR, Swiftify)
* Связывание скрытых токенов с основными (Roslyn)

---

## Связывание скрытых токенов с правилами грамматики (Swiftify)

* Предшествующие (**Precending**)

```
//First comment
'{' /*Precending1*/ a = b; /*Precending2*/ b = c; '}'
```

* Последующие (**Following**)

```
'{' a = b; b = c; /*Following*/ '}' /*Last comment*/ 
```

* Токены-сироты (**Orphans**)

```
'{' /*Orphan*/ '}'
```

---

## Связывание скрытых токенов с основными (Roslyn)

* Лидирующие (**LeadingTrivia**)
* Замыкающие (**TrailingTrivia**)

```CSharp

// leading 1 (var)
// leading 2 (var)
var foo = 42; // trailing (;)

// leading (int)
int bar = 100500; // trailing (;)

// leading (EOF)
EOF
```

* Более правильный подход!

---

# Препроцессорные директивы

## Одноэтапная обработка ([Swiftify](https://objectivec2swift.com/#/home/main))

* Одновременный парсинг директив докенов основного языка.
* Использование каналов для изоляции токенов препроцессорных директив.

## Двухэтапная обработка ([Codebeat](https://codebeat.co/))

1. Токенизация и разбор кода препроцессорных диреткив.
2. Вычисление условных директив `#if` и определение компилируемых блоков кода.
3. Замена директив из исходника на пробелы.
4. Токенизация и парсинг результирующего текста с удаленными директивами.

---

# Препроцессорные директивы

## Одноэтапная обработка

Интерпретация и обработка макросов вместе с функциями:

```
#define DEGREES_TO_RADIANS(degrees) (M_PI * (degrees) / 180)
```

```Swift
func DEGREES_TO_RADIANS(degrees: Double) 
   -> Double { return (.pi * degrees)/180; }
```

---

# Препроцессорные директивы

## Двухэтапная обработка

Исключение блоков кода, с которыми парсинг будет ошибочным:

```
BOOL trueFlag =
#if DEBUG
    YES
#else
    arc4random_uniform(100) > 95 ? YES : NO
#endif
;
```

```
BOOL trueFlag =

    YES

;
```

---

# Обработка ошибок

#### Лексическая ошибка

```CSharp
class # { int i; }
```

#### Единичный лишний токен

```CSharp
class T ; { int i; }
```

#### Единичный отсутствующий токен

```CSharp
class T { int f(x) { a = 3; } }
```

#### Несколько лишних токенов (режим «паники»)

```CSharp
class T { int f(x) { a = 3 4 5; } }
```

#### Отсутствующее альтернативное подправило

```CSharp
class T { int ; }
```

---

# Ошибки в Roslyn

```
namespace App
{
    class Program
    {
        ;                    // Skipped Trivia
        static void Main(string[] args)
        {
            a                // Missing ';'
            ulong ul = 1lu;  // Incorrect Numeric
            string s = """;  // Incorrect String
            char c = '';     // Incorrect Char
        }
    }

    class bControl flow
    {
        c                    // Incomplete Member
    }
}
```

---

# Использование в IDE

### Всплывающая подсказска

* Только с использование грамматики грамматики
* С использованием семантики
* Чудес не бывает - все равно придется писать код в ручную для лучшего качества

---

# Оптимизация грамматик

### Леворекурсивные правила вместо обычных

**Обычные**

```ANTLR
expression
    : multExpression
    | expression ('+'|'-') multExpression
    ;

multExpression
    : ID
    | multExpression ('*'|'/') ID
    ;
```

**Леворекурсивное**

```ANTLR
expression
    : expression ('*'|'/') expression
    | expression ('+'|'-') expression
    | ID
    ;
```

---

# Производительность

### Уменьшение количества правил парсинга

* Каждое правило - вызов метода

### Профилирование

* Уровень грамматики. Плагин к Idea [ANTLR v4 grammar plugin](https://plugins.jetbrains.com/plugin/7358-antlr-v4-grammar-plugin) 
* Уровень рантайма. Тестирование парсера на большом файле и определение узких мест.

---

# Преобразование деревьев

* Методы обхода
* Реализации Visitor и Listener
* Архитектура
* Доступ (Аксессинг) к узлам
* Фичи C# 7 на практике
* Сериализация
* Оптимизации

---

# Методы обхода

## Посетитель (**Visitor**)

* Тип возвращаемого значения для каждого правила. Например, `string` для конвертера исходных кодов (Swiftify).
* Выборочный обход дочерних узлов.

## Слушатель (**Listener**)

* Посещает все узлы.
* Ограниченный функционал: можно использовать для подсчета простых метрик кода.

---

## Реализации Visitor и Listener

### Написанные вручную
* Долгая и утомительная разработка

### Сгенерированные (ANTLR)
* Избыточность
* Нарушение Code Style
* Доступны не всегда
* Универсальность (Java, C#, Python2|3, JS, C++, Go, Swift)
* Скорость разработки

### Динамические

* Используется рефлексия и динамические типы с  `dynamic`
* Медленная скорость, но хорошо для прототипов

---

# Visitor

## Архитектура

* Один большой с использованием `partial` классов.
* Несколько маленьких. Создание визиторов при необходимости.

## Статические проверки

Перегрузка всех методов и использование "заглушек"

```
throw new ShouldNotBeVisitedException(context);
```

---

# Аксессоры

Используются для доступа к различным узлам дерева разбора.

---

## C# 7: Локальные функции и Is Expression

```
public static List<Terminal> GetLeafs(this Rule node)
{
    var result = new List<TerminalNode>();
    GetLeafs(node, result);
    
    // Local function
    void GetLeafs(Rule localNode, List<Terminal> localResult)
    {
        for (int i = 0; i < localNode.ChildCount; ++i)
        {
            IParseTree child = localNode.GetChild(i);
            // Is expression
            if (child is TerminalNode typedChild)
                localResult.Add(typedChild);

            GetLeafs(child, localResult);
        }
    }

    return result;
}
```

---

# Сериалзиация

* Бинарная
* Xml
* Json.NET

Подробности:
* Проект на GitHub: [TreeProcessing.NET](https://github.com/KvanTTT/TreeProcessing.NET).
* [Сериализация, отображение, сравнение и обход древовидных структур в .NET](https://github.com/KvanTTT/Articles/blob/tree-structures-serialization-comparison-and-mapping-on-net/Tree-Structures-Serialization-Comparison-and-Mapping-on-NET/Russian.md).

---

# JSON сериализация

* Сериализация по-умолчанию с параметром ` TypeNameHandling.All`
* Длинные и платформозависимые имена типов

Результирующий JSON:

```JSON
"Statements": {
    "$type": "System.Collections.Generic.List`1[[TreesProcessing.NET.Statement, TreesProcessing.NET]], mscorlib",
    "$values": [
        ...
    ]
```

---

# JSON сериалзиация

* **Свойство** в качестве идентификации типа
```JSON
override NodeType NodeType => NodeType.InvocationExpression;
```
* **Имя класса** в качестве идентификации типа
* **Аттрибут** в качестве идентификации типа
```CSharp
[NodeAttr(NodeType.BinaryOperatorExpression)]
public class BinaryOperatorExpression : Expression
```

Результирующий JSON:

```JSON
"Statements": {
    "NodeType": "BlockStatement",
    "Statements": [
        ...
    ]
```

---

# Оптимизации

## Мемоизация

* Поиск первого потомка определенного типа `FirstDescendantOfType`.
* Хранение всех потомков для каждого узла дерева вместо их поиска.
* Увеличение производительности в 2-3 раза.
* Увеличение потребления памяти в 3 раза.

## Уменьшение аллокаций

* Метод `Visit` - базовый, он часто вызывается.

---

# Ресурсы

* Исходники презентации: 
* Статьи:
	* [Теория и практика парсинга исходников с помощью ANTLR и Roslyn](https://habrahabr.ru/company/pt/blog/210772)
	* [Обработка препроцессорных директив в Objective-C](https://habrahabr.ru/post/318954/)
* Открытый сигнатурный движок поиска по шаблонам [PT.PM](https://github.com/PositiveTechnologies/PT.PM)
* Грамматики: [grammars-v4](https://github.com/antlr/grammars-v4)

---

# Перспективы и планы

* Грамматики с универсальными вставками кода.
* Использовать PT.PM в более продвинутом ядре поиска по потоку данных AI.Taint.
* Взяться наконец уже за XAML парсер для [Avalonia](https://github.com/AvaloniaUI/Avalonia)!

---

# На правах рекламы

## [Positive Technologies на GitHub](https://github.com/PositiveTechnologies)

## [Swiftify](http://swiftify.io/)