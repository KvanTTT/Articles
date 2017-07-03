В данной публикации описываются различные подходы по сериализации и десериализации, сравнению, а также обходу древовидных структур с помощью Visitor и Listener. Рассматриваются подходы с написанием минимального количества кода, имеющие не очень хорошую производительно и традиционные подходы, имеющие тем не менее лучшую производительность. Затрагиваются аспекты преимущественно .NET разработки, однако какие-то идеи могут использоваться и в других платформах/языках. Материал данной статьи был собран и структурирован нами в результате работы над модулем сигнатурного анализатора кода в проекте [PT Applicatian Inspector](http://www.ptsecurity.ru/appsecurity/application-inspector/) и не только. Код примеров выложен в открытый доступ на GitHub по лицензией MIT: [TreesProcessing.NET](https://github.com/KvanTTT/TreesProcessing.NET). 

<habracut/>

## Содержание

* [Вступление](#Inroduction)
* [Сериализация](#Serialization)
    * [Бинарная сериализация](#BinarySerialization)
        * [BinaryFormatter](#BinaryFormatter)
        * [Protobuf](#Protobuf)
    * [XML сериализация](#XmlSerialization)
        * [XmlSerializer](#XmlSerializer)
        * [DataContractSerializer](#DataContractSerializer)
    * [JSON сериализация с помощью Json.NET](#JsonNET)
        * [Параметр TypeNameHandling.All](#TypeNameHandlingAll)
        * [Cвойство в качестве идентификации типа](#PropertyIdentification)
        * [Имя класса в качестве идентификации типа](#ClassNameIdentification)
        * [Аттрибут в качестве идентификации типа](#AttributeIdentification)
    * [Сериализация с помощью ServiceStack](#ServiceStack)
    * [Другие сериализаторы](#OtherSerializators)
* [Отображение древовдиных структур](#TreeStructuresMapping)
    * [Отображение вручную](#ManualMapping)
    * [Отображение с использованием рефлексии](#ReflectionMapping)
    * [Отображение с помощью Automapper](#AutomapperMapping)
    * [Другие мэпперы](#OtherMappers)
* [Обход древовидных структур](#TreeStructuresTravesing)
    * [Visitor](#Visitor)
    * [Listener](#Listener)
        * [Listener с методами](#ListenerWithMethods)
        * [Listener с событиями](#ListenerWithEvents)
* [Сравнение древовидных стурктур](#TreeStructuresComparison)
    * [Статическое](#StaticComparison)
    * [Динамическое](#DynamicComparison)
    * [Абстрактное синтаксическое дерево Меркелизда](#MerkelizedAST)
* [Использование шаблонизаторов](#TemplateEngines)
* [Заключение](#Conclusion)

<anchor>Inroduction</anchor>

## Вступление

В качестве тестового примера дерева во всех примерах используется простая иерархическая древовидная структура, которая отображает часть дерева разбора из распространенного языка программирования (находится в файле SampleTree.cs). Для наглядности также было представлено графическое представление дерева, полученное с помощью рендеринга генерируемого dot файла утилитой Graphviz. Тест называется **Render_DotFromNodes_PngGraphFile**:

![Sample Tree](https://habrastorage.org/files/1a6/131/83b/1a613183b8b942a794ec03bc0738f37e.png)

При разработке максимальная кросплатформенность была в приоритете, поэтому солюшен было принято разделить на три проекта:
* TreesProcessing.NET.Core
* TreesProcessing.NET.PCL
* TreesProcessing.NET.

Несмотря на сходство Core и PCL проектов, имеются и различия, которые не позволили их объединить в один. Для всех проектов также были добавлены юнит-тесты, которые находятся в одноименных проектах с суффиксом **.Tests**.

<anchor>Serialization</anchor>

## Сериализация

Часто возникает задача сохранения внутреннего состояния модели для последующего его использования (сериализация). Наглядным примером является хранение пользовательских настроек в какой-либо программе. Желательно при этом чтобы формат хранения был удобен для редактирования человеком, например JSON или XML. В .NET существует множество библиотек для работы с этими форматами, однако самой известной является Json.NET. Сериализация простых объектов не представляет особой сложности и интереса. Однако если необходимо сериализовать и десериализовать объект, содержащий в себе иерархическую структуру и свойства из абстрактных классов, которые конкретизируются во время исполнения, то необходимо писать дополнительный код для корректной обработки таких случаев. Примером такого сложного объекта является древовидная струтура. Более того, существует несколько способов для решения этой задачи.

В данной статье рассматривалась бинарная и текстовая сериализация с помощью популярных библиотек Protobuf, MessagePack, Json.NET, а также с помощью стандартных средств. Замеры производительности и эффективности сжатия для сериализации не проводились, потому что их, во-первых, и так достаточно много в сети [SerializersCompare](https://github.com/sidshetye/SerializersCompare), во-вторых, статья все же акцентирует внимание на других вещах.

<anchor>BinarySerialization</anchor>

### Бинарная сериализация

<anchor>BinaryFormatter</anchor>

#### BinaryFormatter

Данный способ сериализации является наверное одним из самых старых и неэффективных как по производителности, так и по размеру. Более того, без дополнительных библиотек его невозможно использовать в новых PCL (Portable Class Library), а значит и в .NET Core, в котором не получилось его использовать вообще никак. Однако он также был рассмотрен в силу того, что его поддержка может понадобится в legacy коде.

Как известно, для того, чтобы класс стало возмжно сериализовать с помощью BinaryFormatter, его нужно помечать аттрибутом `Serializable`. Для его поддержки в PCL нужно использовать обходные пути, например использовать библиотеку [Shim](https://github.com/cureos/shim).

<anchor>Protobuf</anchor>

#### Protobuf

Protobuf был предложен Google еще в 2008 году. Разработчикми утверждают, что по сравнению с XML данный проще, в 3-10 раз меньше, в 20-100 разе быстрее, а также обладает и другими достоинствами. На данный момент (2017) существуют библиотеки под почти все популярные языки: Java, C++, Python, C#, Go и другие.

К сожалению, для сериализации не обойтись без написания дополнительного кода: сериализуемые типы должны помечаться аттрибутом `ProtoContract` (или традиционным `DataContract`, `XmlType`), а сериализуемые члены - `ProtoMember` (`DataMember`, `XmlElement`) с указанием идентификатора. Понравилось то, что Protobuf умеет сериализовывать один объект, даже если он содержится в нескольких других сериализуемых объекта. Для этого используется аттрибут `AsReference`. Это может использоваться для сериализации не только деревьев, но и более сложных структур с циклами, т.е. произвольных графов. Также Protobuf имеет и другие интересные возможности, о которых можно почитать здесь: [protobuf-net attributes](https://github.com/mgravell/protobuf-net/wiki/Attributes).

<anchor>XmlSerialization</anchor>

### Xml сериализация

<anchor>XmlSerializer</anchor>

#### XmlSerializer

Для сериализации с использованием XmlSerializer никакие дополнительные аттрибуты и схемы не требуются, единственно что для абстрактных классов необходимо указать, о каких конкретных классах они "знают". Это осуществляется с помощью аттрибута `XmlInclude`, например, `[XmlInclude(typeof(BlockStatement))]` для `Statement`.

<anchor>DataContractSerializer</anchor>

#### DataContractSerializer

Для сериализации на основе DataContract необходимо помечать каждый сериализуемый класс аттрибутом `DataContract`, а каждый член - `DataMember`. Стоит отметить, что такие аттрибуты могут конфликтовать и с другими сериализаторами, например Json.NET. Последний в этом случае сериализует члены класса, помеченные аттрибутом `DataMember`, если хотя бы один член им помечен. По аналогии и с `XmlSerializer`, абстрактные классы должны быть помечены аттрибутами, указывающими какие конкретные классы могут быть созданы из данного абстрактного класса. Например, `[KnownType(typeof(BlockStatement))]` для `Statement`.

<anchor>JsonNET</anchor>

### JSON сериализация с помощью Json.NET

Сериализацию с помощью самого распространенного сериализатора Json.NET мы решили описать более подробно, описав различные способы. 

<anchor>TypeNameHandlingAll</anchor>

#### Параметр TypeNameHandling.All

Самым простым решением является использование параметра `TypeNameHandling.All` в настройках:
```CSharp
var settings = new JsonSerializerSettings() { TypeNameHandling = TypeNameHandling.All };
string expectedJson = JsonConvert.SerializeObject(tree, settings);
```

Недостатком такого простого подхода является то, что в результирующем Json будут фигурировать полные имена всех типов, что увеличит размер файла, усложнит его восприятие и сделает его "зависимым" от .NET платформы. Вот так монструозно будет выглядеть фрагмент JSON файла с полными типами:

```
"Statements": {
    "$type": "System.Collections.Generic.List`1[[TreesProcessing.NET.Statement, TreesProcessing.NET]], mscorlib",
    "$values": [
        ...
    ]
```

<anchor>PropertyIdentification</anchor>

#### Cвойство в качестве идентификации типа

Последующие решения основаны на реализации собственного `JsonConverter`, который позволяет переопределять логику десериализации определенных узлов.

В качестве метки конкретного класса в этом случае является свойство типа `Enum`, `string` и т.п.:
```CSharp
public override NodeType NodeType => NodeType.InvocationExpression;
```

Конвертер используется только при десериализации (потому что при сериализации это свойство и так сохраняется). Метод `ReadJson` будет выглядеть следующим образом:
<spoiler title="Метод ReadJson в PropertyJsonConverter">
```CSharp
public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
{
    if (reader.TokenType != JsonToken.Null)
    {
        JObject jObject = JObject.Load(reader);

        object target = null;
        if (objectType == typeof(Node) || objectType.GetTypeInfo().IsSubclassOf(typeof(Node)))
        {
            var nodeTypeObject = jObject[nameof(NodeType)];
            var nodeType = (NodeType)Enum.Parse(typeof(NodeType), nodeTypeObject.ToString());
            target = Activator.CreateInstance(nodeTypes[nodeType]);
        }
        else
        {
            throw new FormatException("Invalid JSON");
        }

        serializer.Populate(jObject.CreateReader(), target);
        return target;
    }

    return null;
}
```
</spoiler>

В данном коде проверяется, является ли класс свойства производным от класса `Node`, и, если является, то используется `Activator` для динамического создания конкретного класса-наследника `Node`.

Несмотря на то, что при использовании такого read-only свойства существует некоторая избыточность, т.к. имя свойства дублирует имя класса, оно добавляет гибкости, потому что его можно сериализовать как произвольное число или строку. Кроме того, данное свойство может оказаться полезным не только для сериализации, но и для других алгоритмов, например при сравнении деревьев.

По сравнению с Json.NET сериализацией древовидных структур по-умолчанию, текущий и последующие два способа имеют следующие недостатки и достоинства.

Достоинства:
* чистый Json, который можно использовать как в .NET, так и на других платформах;
* тип узла можно использовать и для других целей (для сравнений). Кроме того тип узла можно сохранять как в числовом, так и в текстовом виде.

Недостатки:
* необходимо реализовывать свой конвертер;
* больше кода в классах;
* неудобно, если классы расположены в разных сборках.

<anchor>ClassNameIdentification</anchor>

#### Имя класса в качестве идентификации типа

Для следующих двух решений необходимо реализовывать не только метод `ReadJson`, но и `WriteJson`, потому что в этих случаях при сериализации используется метаинформация: имя класса или его аттрибут, а не свойство.
<spoiler title="WriteJson в ClassNameJsonConverter">
```CSharp
public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
{
    JObject jObject = new JObject();
    Type type = value.GetType();
    jObject.Add(PropertyName, type.Name);
    PropertyInfo[] properties = ReflectionCache.GetClassProperties(type);
    foreach (PropertyInfo prop in properties)
    {
        object propVal = prop.GetValue(value, null);
        if (propVal != null)
        {
            jObject.Add(prop.Name, JToken.FromObject(propVal, serializer));
        }
    }
    jObject.WriteTo(writer);
}
```
</spoiler>

Как видим, в этом случае код получается более сложным и необходимо больше ~~рефлексировать~~ полагаться на информацию, доступную во время исполнения (рефлексию). В коде реализуется извлечение свойств-членов из каждого класса и производится их обход.

<anchor>AttributeIdentification</anchor>

#### Аттрибут в качестве идентификации типа

Данный подход предполагает использование собственных аттрибутов, которыми помечается каждый сериализуемый класс:

```CSharp
[NodeAttr(NodeType.BinaryOperatorExpression)]
public class BinaryOperatorExpression : Expression
```

По реализации данный подход практически ничем не отличается от предыдущего за исключением того, что на этапе записи информации о типе узла извлекается не имя класса, а его аттрибут.

<anchor>ServiceStack</anchor>

### Сериализация с помощью ServiceStack

По словам разработчиков ServiceStack является независимым и высокопроизводительным сериализатором, который поддерживает не только Json формат, но и JSV, CSV форматы. JSV является гибридом JSON и CSV: с одной стороны его можно использовать для сериализации сложных структур, в отличие от CSV, а с другой - он занимает меньше места, чем чистый Json за счет удаления кавычек и некоторых других символов `[]{},`. Правда разница в размере получается достаточно небольшой. Сейчас ServiceStack поддерживается в .NET Core, что не может не радовать. Более подробное данной библиотеке можно найти на гитхабе: [ServiceStack.Text](https://github.com/ServiceStack/ServiceStack.Text). А тесты же находятся в файле **SerializationTests.cs** и называются **ServiceStackJson_Serialization** и **ServiceStackType_Serialization**.

<anchor>OtherSerializators</anchor>

### Другие сериализаторы

К сожалению, заставить работать достаточно популярный и быстрый бинарный сериализатор MessagePack для древовидных структур не удалось: видимо он не поддерживает промежуточные абстрактные классы, например цепочку Node -> Statement -> ForStatement, потому что он выбрасывает исключение `This operation is not supported because 'TreeProcessing.NET.Statement' cannot be instanciated.` при попытке настроить сериализатор.

Другие сериализаторы пока что особо не рассматривались в этой статье и проекте. Хотя некоторые из них вполне себе могут посоревноваться с рассмотренными по размеру сериализуемых сообщений или производительности.

<anchor>TreeStructuresMapping</anchor>

## Отображение древовидных структур

Отображение - преобразование одного объекта в другой. Часто при задаче отображения, нужно решить и задачу сериализации. Существуют разные сценарии, где это может потребоваться:
* преобразование иерархической модели объектов в плоскую;
* преобразование доменных объектов в сериализуемые объекты для передачи через какую-либо коммуникационную среду. Например, получение модели из Data Transfer Object (DTO)
* сохранение сложных настроек приложения.

Мэппинг древовидных структур осуществить не так просто, потому что потомки узлов могут создаваться динамически. В статье и в репозитории рассматривается второй тип мэппинга, а именно преобразование модели в DTO и наоборот.

<anchor>ManualMapping</anchor>

### Отображение вручную

Для мэппинга был написан метод, осуществляющий отображение модели в DTO для каждого типа узла. Недостатком ручного подхода является большое количество кода, а, следовательно, и потенциальные ошибки.

```CSharp
public static Node DtoToModel(NodeDto dto)
{
    if (dto == null)
    {
        return null;
    }
    Node result;
    switch (dto.NodeType)
    {
        case NodeType.BinaryOperatorExpression:
            var binaryOperatorExpressionDto = dto as BinaryOperatorExpressionDto;
            result = new BinaryOperatorExpression(
                (Expression)DtoToModel(binaryOperatorExpressionDto.Left),
                binaryOperatorExpressionDto.Operator,
                (Expression)DtoToModel(binaryOperatorExpressionDto.Right));
            break;
```

<anchor>ReflectionMapping</anchor>

### Отображение с использованием рефлексии

Помимо метода с огромным switch на каждый тип узла был написан метод и с использованием рефлексии. Реализовывался он так, как описано в методах `DtoToModelViaReflection` и `ModelToDtoDynamicViaReflection` из файла **MapperHelper.cs**. Недостатком является низкая производительность и способность функционирования только для классов, имеющих определенные имена. Эти методы различаю класс-источник и класс-приемник по суффиксу Dto в конце. В случае, если какой-либо класс или свойство не найдены, выбрасывается исключение.

<anchor>AutomapperMapping</anchor>

### Отображение с помощью Automapper

Самой распространенной библиоткой мэппинга, пожалуй, является Automapper. Правда для Portable и Core платформы ее использовать не удалось.

Для того, чтобы отображать одни деревья в другие, нужно настроить конфигурацию мэппинга. Она настроивается в каком-либо статическом конструкторе или методе. Делается это следующим образом (файл **AutoMapperHelper.cs**):

```CSharp
Mapper.Initialize(cfg => {
    cfg.CreateMap<Node, NodeDto>()
        .Include<Statement, StatementDto>()
        .Include<Expression, ExpressionDto>()
        .Include<Terminal, TerminalDto>()
        .ReverseMap();
        
    cfg.CreateMap<Statement, StatementDto>()
        .Include<BlockStatement, BlockStatementDto>()
        .Include<ExpressionStatement, ExpressionStatementDto>()
        .Include<ForStatement, ForStatementDto>()
        .Include<IfElseStatement, IfElseStatementDto>()
        .ReverseMap();
        
    ...
});
```

Стоит обратить внимание, что если у абстрактного класса есть цепочка из нескольких абстрактных классов наследников, например как в случае `Node` -> `Statement` -> `BlockStatement`, то метод `CreateMap` необходимо вызывать как для всех абстрактных классов, так и для конкретных классов. Впрочем, это также продемонстрировано во фрагменте кода выше. Для того, чтобы иметь возможность отображать модели обратно в Dto, необходимо вызывать метод `ReverseMap`.

После того, как мэппинг был сконфигурирован, объекты можно легко мэппить в обе стороны следующим образом (тесты в **MapperTests.cs**):

```CSharp
AutoMapperHelper.Initialize();

NodeDto treeDto = Mapper.Map<NodeDto>(tree);
var mappedBackTree = Mapper.Map<Node>(treeDto);
```

Помимо простого мэппинга объектов, Automapper обладает и другими возможностями, среди которых:
* Flattering, уплощение. Возможность автоматического преобразования иерархических данных в плоские.
В данном случае, например при мэппинге объекта `Order` в `OrderDto` обращение к члену `Customer.Name` в объекте `Order` превратится в свойство `CustomerName` в объекте `OrderDto` автоматически:
```
public class Order
{
    public Customer Customer { get; set; }
}
public class Customer
{
    public string Name { get; set; }
}
public class OrderDto
{
    public string CustomerName { get; set; }
}
```

* Проекции используются в том случае, если определенные совйтсва отображаемых объектов совпадают не точно и требуется более тонкая настройка.
* Валидация конфигурации мэппинга. Используется если требуется, чтобы все поля отображаемых объектов имели взаимнооднозначное соответствие.
* Пользовательские конвертеры типов. Используются в случае если какие-то своства отображаемых объектов имеют разные типы и для них необходима своя логика обработки. Например, преобразование даты в строку.
* Пользовательские ресолверы значений. Применяются в случае, если для нескольких свойств используется своя логика отображения в единственное свойство. Например, если требуется посчитать сумму и отобразить ее в результат.
* Возможность подписки на события, вызываемые до или после отображения определенных свойств.
* Замещение null значений в объектах-источниках при необходимости.
* Мэппинг определенных свойств в зависимости от условий. Например, если требуется отобразить значение `int` в `uint` в том случае, если первое больше либо равно нулю.
* Продвинутая работа с Generics и Linq.
* Многие другие возможности, которые подробно описаны в [AutoMapper Wiki](https://github.com/AutoMapper/AutoMapper/wiki).

<anchor>OtherMappers</anchor>

### Другие мэпперы

В .NET есть несколько библиотек для мэппинга. Однако, например, несмотря на то, что ServiceStack помимо сериализации поддерживает и отображение, заставить работать его не удалось, поскольку он не поддерижвает абстрактные классы. Примерно по такой же причине не удалось заставить работать и быструю библиотеку для отображения TinyMapper. А жаль, ведь ее производительность впечатляет по сравнению с Automapper: [TinyMapper - a quick object mapper for .Net](https://github.com/TinyMapper/TinyMapper). На сайте пишется, что такая производительность достигается засчет генерации кода с помощью [ILGenerator](https://msdn.microsoft.com/en-us/library/system.reflection.emit.ilgenerator.aspx).

Помимо библиотек для мэппинга также можно использовать шаблонизаторы, например проект [T4Mapping](https://github.com/dj-raphael/T4Mapping) хабраюзера @dj_raphael.

<anchor>TreeStructuresTravesing</anchor>

## Обход древовидных структур

Для обхода древовидных структур в практике программировании существует два основных подхода: Посетитель (Visitor) и Слушатель (Listener). Различия и сценарии использования описаны в одной из предыдущих статей про AST: [Visitor и Listener](https://habrahabr.ru/company/pt/blog/210060/#visitor-vs-listener).

<anchor>Visitor</anchor>

### Visitor

Visitor позволяет выносить логику обработки узлов дерева в отдельные методы, которые, в свою очередь, возвращают значение с типом, определенным пользователем. Таким образом, Visitor может использоваться для преобразования деревьев, или, например, для трансформации одного исходного кода в другой, если результирующий тип будет строкой.

Обычно Visitor подразумевает реализацию метода `Accept(IVisitor visitor)` для каждого типа узла дерева, который в свою очередь вызывает дефолтный или перегруженный метод `Visit(Node node)` в переданном посетителе. Однако я решил не реализовывать эти методы, поскольку они только увеличивают код, а без них обойтись можно. Более того хотелось, чтобы вся логика обхода по возможности была сконцентрирована в одном классе.

В проекте реализовывались как статические визиторы, так и визиторы, построенные на рефлексии. Вот так, например, выглядит метод Метод Visit для обхода конечных (терминальных) узлов дерева:

```CSharp
public virtual Node Visit(Terminal terminal)
{
    switch (terminal.NodeType)
    {
        case NodeType.Identifier:
            return Visit((Identifier)terminal);
        case NodeType.BooleanLiteral:
            return Visit((BooleanLiteral)terminal);
        case NodeType.FloatLiteral:
            return Visit((FloatLiteral)terminal);
        case NodeType.IntegerLiteral:
            return Visit((IntegerLiteral)terminal);
        case NodeType.NullLiteral:
            return Visit((NullLiteral)terminal);
        case NodeType.StringLiteral:
            return Visit((StringLiteral)terminal);
        default:
            throw new InvalidOperationException();
    }
}
```

При реализации посетителя с использованием рефлексии были выделены следующие основные сущности в реализации обхода потомков по-умолчанию `VisitChildren` (файл **DynamicVisitor.cs**):
* примитивные значения просто копируются;
* значения типа `Node` или его наследники. Для них рекурсивно вызывается метод `Visit`;
* коллекция узлов. Если дочерний объект реализует интерфейс `IList`, то такая коллекция обходится и для каждого итерируемого элемента вызывается метод `Visit`.

Помимо вынесения логики обработки, Visitor также сразу же предоставляет возможность глубокого копирования древовидных структур, что также может оказаться полезным в определенных сценариях.

<anchor>Listener</anchor>

### Listener

Если изменять дерево не требуется, а нужно анализировать только определенные типы узлов, то можно использовать шаблон проектирования Listener. Listener можно реализовать как с событиями, так и с виртуальными методами.

<anchor>ListenerWithMethods</anchor>

#### Listener с методами

Реализации статического и динамического Listener с методами ничем не отличает от реализации Visitor за исключением того, что перед и после вызова методов `Visit` для потомков узла, вызываются "методы-слушатели" `Enter` и `Exit`, как это продемонстрировано в коде ниже. Реализации находятся в файлах **StaticListener.cs** и **DynamicListener.cs** соответственно.

<spoiler title="Метод Visit, вызывающий Enter и Exit, для терминальных узлов">
```CSharp
private void Visit(Terminal terminal)
{
    switch (terminal.NodeType)
    {
        case NodeType.Identifier:
            var identifier = (Identifier)terminal;
            Enter(identifier);
            Exit(identifier);
            break;
        case NodeType.BooleanLiteral:
            var booleanLiteral = (BooleanLiteral)terminal;
            Enter(booleanLiteral);
            Exit(booleanLiteral);
            break;
        case NodeType.FloatLiteral:
            var floatLiteral = (FloatLiteral)terminal;
            Enter(floatLiteral);
            Exit(floatLiteral);
            break;
        case NodeType.IntegerLiteral:
            var integerLiteral = (IntegerLiteral)terminal;
            Enter(integerLiteral);
            Exit(integerLiteral);
            break;
        case NodeType.NullLiteral:
            var nullLiteral = (NullLiteral)terminal;
            Enter(nullLiteral);
            Exit(nullLiteral);
            break;
        case NodeType.StringLiteral:
            var stringLiteral = (StringLiteral)terminal;
            Enter(stringLiteral);
            Exit(stringLiteral);
            break;
        default:
            throw new InvalidOperationException();
    }
}
```
</spoiler>

<anchor>ListenerWithEvents</anchor>

#### Listener с событиями

Listener с событиями можно реализовать с использованием рефлексии, так и без нее.

Listener с событиями и рефлексией работает только на полноценной платформе .NET, поскольку использует продвинутые возможности по рефлексии. Если в классе объявить события, но при этом не вызывать их (а события, как известно, можно вызывать только из методов класса), то компилятор будет помечать, что эти события нигде не используются, и будет прав. Однако он будет ошибаться в случае, если события будут вызываться динамически, через рефлексию, например так:

```CSharp
var eventDelegate = (Delegate)typeof(DynamicEventListener)
    .GetField("EnterBinaryOperatorExpression", BindingFlags.Instance | BindingFlags.NonPublic)
    .GetValue(this);
eventDelegate?.DynamicInvoke(new object[] { this, node });
```

К счастью, в C# компиляторе существует возможность отключения таких предупреждений с помощью директивы препроцессора:

```CSharp
#pragma warning disable CS0067
```

Однако такой Listener использовать крайне не рекомендуется и приведен он в образовательных целях. Статический Listener с событиями (**StaticEventListener.cs**) прекрасно работает на любых платформах.

<anchor>PatternMatching</anchor>

### Сопоставление с паттерном

C# 7 и функциональное программирование

<anchor>TreeStructuresComparison</anchor>

## Сравнение древовидных структур

Под сравнением древовидных структур подразумевается рекурсивное сранение потомков дерева в глубь. В одной из предыдущих статей цикла такой алгоритм рассматривался подробно: [Алгоритм сопоставления AST и шаблонов](https://habrahabr.ru/company/pt/blog/210060/#tree-matching-algorithm). В этой статье же описываются детали реализации сравнения деревьев на платформе .NET, а именно, статическое сравнение с переопределением метода `CompareTo` для дочерних узлов и динамическое с использованием рефлексии.

Фактически сравнение древовидных структур можно реализовать развернув структуру дерева в линейную, но в таком случае не получится реализовать более продвинутые алгоритмы. Однако такая линейная структура может передаваться, например, на вход какой-либо нейросети, анализирующей исходный код. И это будет не потоком обычных токенов, а потоком древовидных токенов. Для получения такой линейной последовательности в проекте используется свойство `AllDescendants` или метод `GetAllDescendants()`. Для получения же только непосредственных дочерних узлов используется свойство `Children` или метод `GetChildren()`.

<anchor>StaticComparison</anchor>

### Статическое

Для того, чтобы любой узел можно было бы сравнить с любым другим узлом, можно реализовать интерфейс `IComparable` для класса `Node`. Таким образом, необходимо будет реализовать метод `CompareTo` для всех наследников `Node`. Например, в `BlockStatement` он будет следующим:

```CSharp
public override int CompareTo(Node other)
{
    int result = base.CompareTo(other);
    if (result != 0)
    {
        return result;
    }

    BlockStatement statement = (BlockStatement)other;
    if (Statements.Count != statement.Statements.Count)
    {
        return Statements.Count - statement.Statements.Count;
    }

    for (int i = 0; i < Statements.Count; i++)
    {
        result = Statements[i].CompareTo(statement.Statements[i]);
        if (result != 0)
        {
            return result;
        }
    }

    return 0;
}
```

Данный код будет работать быстро, однако в нем будет легко ошибиться, из-за того, что большая часть логики будет дублироваться в разных классах.

<anchor>DynamicComparison</anchor>

### Динамическое

Так как все типы узлов по сути являются однотипными, т.е. содержат либо дочерний узел, либо значение примитивного типа (int, string, float и т.д.), либо коллекцию, то обход таких потомков можно обобщить с использованием рефлексии. Для получения всех свойст объекта используется `GetClassProperties`, далее, в зависимости от типа узла, производится сравнение примитивных значений, либо рекурсивное сравнение потомков. Детали реализации можно найти в классе [**StaticHelper.cs**](https://github.com/KvanTTT/TreesProcessing.NET/blob/master/TreesProcessing.NET/StaticHelper.cs) в методах `Compare`.

<anchor>HashComparison</anchor>

### Сравнение по хешу

Как известно, помимо перебора всех дочерних узлов и их сравнения, можно также вычислять хеш-сумму для всех узлов дерева, а затем уже сравнивать ее. В качестве функции хеширования для двух целых чисел `int` используется очень простая функция `unchecked((currentKey * (int)0xA5555529) + newKey)` заимствованная из Roslyn. Для примитивных значений, таких как `string`, `int`, `float` используется .NET реализация по-умолчанию, которая, кстати, не обязательно является портируемой.

<anchor>MerkelizedAST</anchor>

### Абстрактное синтаксическое дерево Меркелизда

На последок хотелось бы упомянуть и о более интересном дереве
[Merkelized Abstract Syntax Trees](http://www.mit.edu/~jlrubin/public/pdfs/858report.pdf)

<anchor>TemplateEngines</anchor>

## Использование шаблонизаторов

<anchor>Conclusion</anchor>

## Заключение