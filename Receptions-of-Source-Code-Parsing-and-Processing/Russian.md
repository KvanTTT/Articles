<!-- $theme: default -->

# Теория и практика формальных языков

* Иван Кочуркин
* Работаю в [Positive Technologies](https://www.ptsecurity.com/ru-ru/) над открытым универсальным сигнатурным анализатором кода [PT.PM](https://github.com/PositiveTechnologies/PT.PM)
* Подрабатываю в [Swiftify](http://swiftify.io/), веб-сервисе для конвертинга
  кода Objective-C в Swift
* Веду активную деятельность на [GitHub](https://github.com/KvanTTT)
* Пишу статьи на [Хабре](https://habrahabr.ru/users/kvanttt/) и [GitHub](https://github.com/KvanTTT/Articles) под ником KvanTTT

---

# Почему не Regex?

1. `<table>(.*?)</table>`
2. А если аттрибуты? `<table.*?>(.*?)</table>`
3. А если элементы? `tr`, `td`
4. А если комментарии? `<!-- html comment>`
5. ...
6. [NO NOO̼O​O NΘ stop the an​*̶͑̾̾​̅ͫ͏̙̤g͇̫͛͆̾ͫ̑͆l͖͉̗̩̳̟̍ͫͥͨe̠̅s ͎a̧͈͖r̽̾̈́͒͑e n​ot rè̑ͧ̌aͨl̘̝̙̃ͤ͂̾̆ ZA̡͊͠͝LGΌ ISͮ̂҉̯͈͕̹̘̱ TO͇̹̺ͅƝ̴ȳ̳ TH̘Ë͖́̉ ͠P̯͍̭O̚​N̐Y̡ H̸̡̪̯ͨ͊̽̅̾̎Ȩ̬̩̾͛ͪ̈́̀́͘ ̶̧̨̱̹̭̯ͧ̾ͬC̷̙̲̝͖ͭ̏ͥͮ͟Oͮ͏̮̪̝͍M̲̖͊̒ͪͩͬ̚̚͜Ȇ̴̟̟͙̞ͩ͌͝S̨̥̫͎̭ͯ̿̔̀ͅ](https://stackoverflow.com/a/1732454/1046374)

---

# Лексемы и токены

* **Лексема** - распознанная последовательность символов
* **Токен** - лексема + тип

```ANTLR
MyKeyword: 'var';
Id:        [a-z]+;
Digit:     [0-9]+;
```

---

# Дерево разбора и AST

<!-- Возможно все же заменить на свою картинку -->

* **Дерево разбора** - древовидная структура, распознанная из потока токенов
* **AST** - дерево разбора без пробелов, точек с запятых и т.д.

![Лексер и парсер](https://habrastorage.org/files/6c4/385/fbe/6c4385fbe3d8471982c9b2a030106d38.png)

---

# Типы парсеров

* Готовые библиотеки парсинга (regex, JSON.NET)
  * API
  * Только самые распространенные языки
* Парсеры, написанные вручную (Roslyn)
  * Большие возможности и гибкость
  * Медленная скорость разработки
* Автоматически сгенерированные парсеры (ANTLR)
  * Большой порог вхождения
  * Быстрая скорость разработки после освоения
  * Меньшая гибкость по сравнению с ручными парсерами

---

# Грамматика

Формальное описание языка, которое может быть использовано для распознавания его структуры.

### Пример грамматики

```ANTLR
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

# Типы языков

* Регулярные
* Контекстно-свободные
* Контекстно-зависимые
* Тьюринг-полные

---

# Инструменты и библиотеки под C#

* Генераторы
  * Контекстно-свободные (ANTLR, Coco/R, Gardens Point Parser Generator, Grammatica, Hime Parser Generator, LLLPG)
  * Безлексерные PEG (IronMeta, Pagarus)
* Комбинаторы (Parseq, Parsley, LanguageExt.Parsec, Sprache, Superpower)
* Языковые фреймворки (JetBrains MPS, Nitra, Roslyn) 

Детально описание [Parsing In C#: Tools And Libraries](https://tomassetti.me/parsing-in-csharp).

---

# Парсер-комбинаторы

* Использование внутри языка разработки (C#)
* Использование в IDE

Библиотеки:

* [Sprache](https://github.com/sprache/Sprache)
* [superpower](https://github.com/datalust/superpower)

---

# Примеры кода парсер-комбинатора

```CSharp
// Parse any number of capital 'A's in a row
var parseA = Parse.Char('A').AtLeastOnce();
```

Правило для `id`:

```CSharp
Parser<string> identifier =
    from leading in Parse.WhiteSpace.Many()
    from first in Parse.Letter.Once()
    from rest in Parse.LetterOrDigit.Many()
    from trailing in Parse.WhiteSpace.Many()
    select new string(first.Concat(rest).ToArray());

var id = identifier.Parse(" abc123  ");

Assert.AreEqual("abc123", id);
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

Пример: `var var = 0;`

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

# Объектный конструктор в C#

```CSharp
class Foo
{
    public string Bar { get; set; }
}
public string Bar { get; set; } = "Bar2";

...

foo = new Foo
{
    Bar = Bar // Umbiguity here
};
```

---

# Какой результат возвращает `nameof`?

```CSharp
class Foo
{
    public string Bar { get; set; }
}

static void Main(string[] args)
{
    var foo = new Foo();
    WriteLine(nameof(foo.Bar));
}
```

---

# 42

```CSharp
class Foo
{
    public string Bar { get; set; }
}

static void Main(string[] args)
{
    var foo = new Foo();
    WriteLine(nameof(foo.Bar));
}

static string nameof(string str)
{
    return "42";
}
```

<!-- ![Troll-Face](Troll-Face.png) -->

---

# Неоднозначность: решение с использованием вставок кода

* **Действия** - производят вычисления на целевом языке парсера.
* **Семантические предикаты** - возвращают результат.

```ANTLR
// Lexer
ID:  [0-9a-zA-Z];

// Parser
varDeclaration
    : id {_input.Lt(-1).Text == "VAR"}? id ('=' expression)? ';'
    ;

id
    : ID;
```

---

# Контекстно-зависимые конструкции

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

Использование вставок кода, смотри лексеры [PHP](https://github.com/antlr/grammars-v4/blob/master/php/PHPLexer.g4).

---

# Интерполируемые строки в C# 6

```CSharp
s = $"{p.Name} is \"{p.Age} year{(p.Age==1 ? "" : "s")} old";
s = $"{(p.Age == 2 ? $"{new Person { } }" : "")}";
s = $@"\{p.Name}
                       ""\";
s = $"Color[R={func(b: 3):#0.##}, G={G:#0.##}, B={B:#0.##}]";
```

* Реализация в [CSharpLexer.g4](https://github.com/antlr/grammars-v4/blob/master/csharp/CSharpLexer.g4).

---

# Лексерный хак: плохое решение

* Вход:
  * `static MyType staticGlobaVar;`
  * [`unsinged int;`](https://goo.gl/kKhdCh)
* Грамматика
  ```ANTLR
  varDeclaration
      : specifiers initDeclaratorList? ';'
      ;
  ```
* Токены: `static` (specifier), `MyType` (specifier), `staticGlobaVar` (specifier)
* Решение: поиск последнего `specifier` в визиторе и "превращение" его в `initDeclaratorList`.

---

# Лексерный хак: изящное решение (Swiftify)

* Вход:
  * `static MyType staticGlobaVar;`
  * [`unsinged int;`](https://goo.gl/kKhdCh)
* Грамматика
  ```ANTLR
  varDeclaration
      : (specifiers initDeclaratorList | specifiers) ';'
      ;
  ```
* Токены: `static` (specifier), `MyType` (specifier), `staticGlobaVar` (initDeclaratorList)!
* Решение: уже имеем `initDeclaratorList`, т.к. такая альтернатива приоритетней.

---

# Регистронезависимость

Нечувствительные к регистру языки: Delphi, T-SQL, PL/SQL и другие.

### Решение с помощью грамматики

**Фрагментный токен** не используется в токенизации, но облегчает запись других токенов.

```ANTLR
Abstract:           A B S T R A C T;
BoolType:           B O O L E A N | B O O L;
BooleanConstant:    T R U E | F A L S E;

fragment A: [aA];
fragment B: [bB];
```

Без использования фрагметных токенов:

```ANTLR
Abstract:           [Aa][Bb][Ss][Tt][Rr][Aa][Cc][Tt];
```

---

# Регистронезависимость: решение на уровне рантайма

```ANTLR
Abstract:           'ABSTRACT';
BoolType:           'BOOLEAN' | 'BOOL';
BooleanConstant:    'TRUE' | 'FALSE';
```

Используется [CaseInsensitiveInputStream](https://gist.github.com/sharwell/9424666).
Чувствительные к регистру токены обрабатываются на этапе обхода дерева. Например `$id` и `$ID` в PHP.

Достоинства:

* Код лексера чище и проще
* Производительность выше

---

# Островные языки и конструкции

JavaScript внутри PHP или C# внутри Aspx.

```PHP
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

# PHP: альтернативный синтаксис

Смесь блоков кода на HTML и PHP

```PHP
<?php switch($a): case 1: // without semicolon?>
        <br>
    <?php break ?>
    <?php case 2: ?>
        <br>
    <?php break;?>
    <?php case 3: ?>
        <br>
    <?php break;?>
<?php endswitch; ?>
```

---

# Использование отдельного режима лексем для JavaScript

```ANTLR
ScriptText:   ~[<]+;
ScriptClose:  '</script'? '>' -> popMode;
PHPStart:     '<?=' -> type(Echo), pushMode(PHP);
PHPStart:     '<?' 'php'? -> channel(SkipChannel), pushMode(PHP);
ScriptText2:  '<' ~[<?/]* -> type(ScriptText);
```

* `pushMode` - зайти в другой режим распознавания лексем (JavaScript -> PHP)
* `popMode` - выйти из режима (PHP -> JavaScript)
* `type` - изменить тип токена
* `channel` - поместить токен в изолированный канал (пробелы, комментарии)

---

# Обработка комментариев и пробелов

* Включение скрытых токенов в грамматику
  ```ANTLR
  declaration:
      property COMMENT* COLON COMMENT* expr COMMENT* prio?;
  ```
* Связывание скрытые токенов с правилами грамматики (ANTLR, Swiftify)
* Связывание скрытых токенов с основными (Roslyn)

---

# Связывание скрытых токенов с узлами дерева (Swiftify)

### Предшествующие (**Precending**)

```C
//First comment
'{' /*Precending1*/ a = b; /*Precending2*/ b = c; '}'
```

### Последующие (**Following**)

```C
'{' a = b; b = c; /*Following*/ '}' /*Last comment*/
```

### Токены-сироты (**Orphans**)

```C
'{' /*Orphan*/ '}'
```

---

# Связывание скрытых токенов со значимыми (Roslyn)

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

# Препроцессорные директивы (Roslyn)

```CSharp
bool trueFlag =
#if NETCORE
    true
#else
    new Random().Next(100) > 95 ? true : false
#endif
;
```

### Лидирующие для `;`

```CSharp
#else
····new Random().Next(100) > 95 ? true : false
#endif
```

### Лидирующие для `true`

```
#if NETCORE
····
```

---

# Препроцессорные директивы: одноэтапная обработка (Swiftify)

* Одновременный парсинг директив докенов основного языка.
* Использование каналов для изоляции токенов препроцессорных директив.

Интерпретация и обработка макросов вместе с функциями:

### Objective-C

```Objective-C
#define DEGREES_TO_RADIANS(degrees) (M_PI * (degrees) / 180)
```

### Swift

```Swift
func DEGREES_TO_RADIANS(degrees: Double) 
    -> Double { return (.pi * degrees)/180; }
```

---

# Препроцессорные директивы: двухэтапная обработка (Codebeat)

Исключение блоков кода, с которыми парсинг будет ошибочным:

```CSharp
bool·trueFlag·=
·········
····true
·····
··············································
······
;
```

1. Токенизация и разбор кода препроцессорных диреткив.
2. Вычисление условных директив `#if` и определение компилируемых блоков кода.
3. Замена директив из исходника на пробелы.
4. Токенизация и парсинг результирующего текста с удаленными директивами.

---

# Парсинг фрагментов кода (Swiftify, PT.PM)

### Задача: определение корректного правила для фрагмента кода

### Применение

* Конвертинг кода в плагине IDE (в Swiftify для плагинов к AppCode, XCode)
* Определение языка программирования (в PT.PM для редактора [Approof](https://approof.ptsecurity.ru/))

### Решение

* Регулярные выражения
* Токенизация и операции с токенами
* Парсинг разными правилами

### Примеры

* [Утверждения](http://objc.to/6mpmhq)
* [Декларации методов](http://objc.to/nt25a1)
* [Свойства](http://objc.to/vnpasw)

---

# Ошибки парсинга

#### Лексическая ошибка

```CSharp
class # { int i; }
```

Отдельный канал: `ERROR: . -> channel(ErrorChannel)`

#### Отсутствующий и лишний токены

```CSharp
class T { int f(x) { a = 3; } // Отсутствующий токен
class T ; { int i; } // Лишний токен
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

# Ошибки парсинга в Roslyn

```CSharp
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

# Заключение о парсинге

О чем не будет рассказываться:

* Производительность парсинга
* Форматирование кода
* Использование в IDE

---

# Обработка древовидных структур

* Методы обхода
* Реализации Visitor и Listener
* Архитектура
* Доступ (Аксессинг) к узлам
* Фичи C# 7 на практике
* Сериализация
* Оптимизации

---

# Методы обхода деревьев

## Посетитель (**Visitor**)

* Тип возвращаемого значения для каждого правила. Например, `string` для конвертера исходных кодов (Swiftify).
* Выборочный обход дочерних узлов.

## Слушатель (**Listener**)

* Посещает все узлы.
* Ограниченный функционал: можно использовать для подсчета простых метрик кода.

---

# Visitor & Listener

![Visitor & Listener](https://habrastorage.org/files/bd1/69c/535/bd169c535e854f9681520f520d0db9c3.png)

---

# Реализации Visitor

## Написанные вручную

* Долгая и утомительная разработка

## Сгенерированные (ANTLR)

* Избыточность и нарушение Code Style
* Доступны не всегда
* Универсальность (Java, C#, Python2|3, JS, C++, Go, Swift)
* Скорость разработки

## Динамические

* Медленная скорость, но хорошо для прототипов

---

# Диспетчеризация с использованием рефлексии

```CSharp
// invocation in VisitChildren
Visit((dynamic)customNode);

// visit methods

public virtual T Visit(ExpressionStatement expressionStatement)
{
    return VisitChildren(expressionStatement);
}

public virtual T Visit(StringLiteral stringLiteral)
{
    return VisitChildren(stringLiteral);
}
```

---

# Архитектура Visitor

## Архитектура

* Несколько маленьких. Создание визиторов при необходимости.
* Один большой с использованием `partial` классов.

## Формализация

Перегрузка всех методов и использование "заглушек"

```CSharp
throw new ShouldNotBeVisitedException(context);
```

---

# C\# 7: локальные функции

```CSharp
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
* XML, Json и другие языки разметки
* Собственный формат

Подробности:
* Проект на GitHub: [TreeProcessing.NET](https://github.com/KvanTTT/TreeProcessing.NET).
* [Сериализация, отображение, сравнение и обход древовидных структур в .NET](https://github.com/KvanTTT/Articles/blob/tree-structures-serialization-comparison-and-mapping-on-net/Tree-Structures-Serialization-Comparison-and-Mapping-on-NET/Russian.md).

---

# JSON сериализация деревьев

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

# Своя логика сериализации с JsonConverter

* **Имя класса** в качестве идентификации типа
* **Свойство** в качестве идентификации типа
  ```CSharp
  override NodeType NodeType => NodeType.InvocationExpression;
  ```
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

* Исходники и текст презентации:
* Статьи:
  * [Теория и практика парсинга исходников с помощью ANTLR и Roslyn](https://habrahabr.ru/company/pt/blog/210772)
  * [Обработка препроцессорных директив в Objective-C](https://habrahabr.ru/post/318954/)
* Открытый сигнатурный движок поиска по шаблонам [PT.PM](https://github.com/PositiveTechnologies/PT.PM)
* Грамматики: [grammars-v4](https://github.com/antlr/grammars-v4)
  * [PL/SQL](https://github.com/antlr/grammars-v4/tree/master/plsql)
  * [T-SQL](https://github.com/antlr/grammars-v4/tree/master/tsql)
  * [PHP](https://github.com/antlr/grammars-v4/tree/master/php)
  * [C#](https://github.com/antlr/grammars-v4/tree/master/csharp)
  * [Java8](https://github.com/antlr/grammars-v4/tree/master/java8-pt)
  * [Objective-C](https://github.com/antlr/grammars-v4/tree/master/objc)

---

# Вопросы?

## [Positive Technologies на GitHub](https://github.com/PositiveTechnologies)

## [Swiftify](http://swiftify.io/)

---