## Введение

Приветствую вас, хабравчане. В данном топике я хотел бы осветить подробности разработки на C\# под разнородные целевые платформы, в первую очередь такие как .NET и браузер (JavaScript). В качестве примера желающие могут изучить веб-сервис по обработке фотографий [gfranq.com](http://gfranq.com/), в котором реализована клиентская и серверная обработка фотографий с помощью фильтров, а также функциональность коллажей на основе материала, описанного в данной статье. Так как я не умею подбирать картинки для привлечения внимания, то она будет по теме: [![Схема функционирования](Images/Client-Server-Interaction-Ru.png)](http://habrahabr.ru/post/164439/)

<habracut text="Технические подробности под катом" />

## Содержание

* [Цель](#Goal)
    * [Описание фильтров](#Filters)
    * [Описание коллажей](#Collages)
* [Реализация](#Implementation)
    * [Выбор платформы для обработки фотографии](#Platform)
    * [Трансляция C\# в JavaScript](#Translation)
    * [Структура](#Structure)
        * [Использование alias](#Alias)
        * [Ссылки на файлы](#Links)
        * [Заметки о .NET реализации](#DotnetNotes)
            * [Использование Dispose](#DisposeUsing)
            * [Использование lock](#LockUsing)
        * [Хранение масок в памяти](#MasksInMemory)
    * [Заметки о JavaScript реализации](#JavascriptNotes)
        * [Минификация](#Minification)
            * [Вручную](#ManualMinify)
            * [Автоматически](#AutoMinify)
        * [Debug & Release режимы](#DebugRelease)
        * [Свойство crossOrigin](#crossOrigin)
    * [Оптимизации](#Optimizations)
        * [Использование предвычисленных (табличных) значений](#Precalculs)
        * [Преобразование изображения в массив пикселей](#Pixels)
* [Примеры кода](#CodeSamples)
    * [Общие](#General)
        * [Определение, является ли строка числом](#IsNumericSample)
        * [Целочисленное деление](#IntegerDivSample)
        * [Поворот и обращение изображения на canvas и Bitmap соответственно](#ImageRotateFlipSample)
        * [Асинхронная и синхронная загрузка картинок](#AsyncSyncImageLoadSample)
    * [Только Script\#](#JavascriptOnly)
        * [Определение типа и версии браузера](#BrowserDetermSample)
        * [Отрисовка пунктирной линии](#DashDotLineSample)
        * [Анимация вращения изображения](#RotateAnimationSample)
* [Заключение](#Conclusion)

<anchor>Goal</anchor>

## Цель

Итак, была поставлена задача реализации обработки фотографий фильтрами и
создания коллажей на клиенте, а также, по возможности, на сервере. Для
начала я расскажу как у нас представлены фильтры и коллажи.

<anchor>Filters</anchor>

### Описание фильтров

**Фильтр** в нашем проекте представляется последовательностью действий
(action), подготовленных в Photoshop, примененных к определенной
фотографии. Действие может быть например одно из:

* Изменение яркости
* Изменение контрасности
* Изменение насыщенности
* Коррекций цветовых кривых
* Наложение масок с различными режимами
* Наложение рамок
* ...

Для того, чтобы описать все эти действия нужен определенный формат.
Конечно, существуют стандартные форматы, типа JSON и XML, но было решено
разработать собственный формат по следующим причинам:

* Универсальность кода для всех платформ (.NET, JavaScript, WinPhone и
    др.).
* Формат фильтров простой, не имеющий иерархической структуры, что
    позволяет легко написать парсер для него.
* XML и JSON больше весят (для данного случая).

Например вот как выглядит последовательность действий для фильтра **XPro
Film**:
![Этапы обработки фильтра XProFilm](Images/XProFilm-Filter-Steps-Ru.jpg)

Кроме обработки фотографий фильтром, нужно было еще реализовать обрезку
и поворот изображения. Да, я знал, что существуют jQuery плагины для
реализации поворота и обрезки, но, во-первых, они оказались слишком
перегруженными, а, во-вторых, не вписывались в универсальную архитектуру
проекта.

<anchor>Collages</anchor>

### Описание коллажей

**Коллаж** же представляется в виде нескольких миниатюрных фотографий,
объединенных в одну с использованием маски или без. При этом нужно было
предоставить возможность перетаскивания доступных фоток на коллаж и
менять их положение и масштаб. Коллаж может выглядеть например так:
![Пример коллажа](Images/Collage-Sample.jpg)

Для коллажа тоже используется свой простой формат, хранящий набор
прямоугольников в относительных координатах от 0 до 1, адреса
фотографий, а также их трансформации. Относительные координаты
используются потому что на сервере та же самая клиентские трансформации
применяется к большим фоткам.

<anchor>Implementation</anchor>

## Реализация

Итак, нужно было выбрать платформу, на которой функциональность фильтров
и коллажей работала у пользователей.

<anchor>Platform</anchor>

### Выбор платформы для обработки фотографии

Существует следующие RIA технологии:

* Adobe Flash
* Microsoft Silverlight
* HTML 5 + JavaScript
* Native Client

Из всего этого списка по вполне очевидным причинам заслуживают внимания
в настоящее время только Flash и HTML 5, так как все остальные не
кроссплатформенны. А Silverlight еще и потихоньку отмирает. Хотя
концепция ~~соли~~ NaCl мне очень нравится, но, увы, существует он
только на Chrome, и не известно когда будет и будет ли поддерживаться он
остальными популярными браузерами. Итак, в качестве платформы была
выбрана модная и развивающаяся HTML 5, которая потенциально работает еще
и на iOS, в отличие от Flash. Также такой выбор обосновывается еще и
тем, что существует много библиотек, которые позволяют преобразовывать
C\# в JavaScript, в том числе и в Visual Studio. Об этом, впрочем, будет
рассказываться дальше.

<anchor>Translation</anchor>

### Трансляция C\# в JavaScript

В предыдущем разделе была выбрана платформа для разработки: HTML 5 +
JavaScript. Но возник вопрос, а можно ли написать универсальный C\# код,
который мог бы компилиться и под .NET и под JavaScript? Таким образом
были найдены несколько библиотек для реализации поставленной задачи:

* JSIL
* SharpKit
* Script\#
* И другие, которые можно посмотреть например [в этом списке на
    github](https://github.com/jashkenas/coffee-script/wiki/List-of-languages-that-compile-to-JS).

В итоге было решено использовать **Script\#** из-за того, что JSIL
работает непосредственно со сборками и генерирует менее чистый код (хотя
поддерживает больше возможностей C\#), а SharpKit является коммерческим.
Подробное сравнение подобных инструментов можно увидеть в [вопросе JSIL
vs Script\# vs SharpKit на
stackoverflow](http://stackoverflow.com/q/11547471/1046374) . Резюмируя,
хочу выделить следующие плюсы и минусы использования ScriptSharp по
сравнению с написанием JavaScript вручную:

**Достоинства:**

* Возможность написания универсального кода под .NET и другие
платформы (WP, Mono).
* Разработка на строго типизированном языке C\# с возможностями ООП.
* Поддержка возможностей IDE для разработки (автодополнение,
рефакторинг).
* Определение многих ошибок на этапе компиляции.

**Недостатки:**

* Избыточность и нестандартность генерируемого JavaScript (из-за
    mscorlib).
* Поддержка только спецификации ISO-2 (отсутствие перегрузки функций,
    вывода типов, расширений, генериков и другого).

<anchor>Structure</anchor>

### Структура

Компиляция под .NET и JavaScript одного и того же кода может быть
представлена в виде следующей схемы:
![Схема трансляции C# в .NET и JavaScript](Images/CSharp-Translation-to-NET-JavaScript-Scheme.png)

Несмотря на то, что .NET и HTML5 это совершенно разные технологии, у них
есть и похожие черты. Это относится и к работе с графикой. Например, в
.NET есть **Bitmap**, а в JavaScript аналогом ему является **canvas**.
Также и с **Graphics** и **Context**, и массивами пикселей. Для того
чтобы объединить все это в одном коде, было решено разработать следующую
архитектуру:
![Общий графический контекст в .NET и JavaScript](Images/Common-NET-JavaScript-GraphicsContext.png)

Разумеется дело не ограничивается двумя платформами. В дальнейшем
планируется добавить поддержку WP, а потом, возможно, Android и iOS.
Стоит отметить, что существует два типа графических операций:

* **Использующие функции API** (DrawImage, Arc, MoveTo, LineTo).
    Преимуществом является высокая скорость работы и возможное
    аппаратное ускорение. Недостатком - они могут быть реализованы
    по-разному на различных платформах.
* **Попиксельные.** Преимуществом является возможность реализации
    любых эффектов и унифицированная работа на всех платформах.
    Недостатком - низкая скорость работы. Однако недостатки можно
    нивелировать путем распараллеливания, использования шейдеров и
    использованием заранее рассчитанных таблиц (о чем будет рассказано
    дальше в разделе про оптимизацию).

Как видим, в абстрактом классе **Graphics** описаны все методы для
работы с графикой, а в производных классах они реализованы для различных
платформ. Для того, чтобы абстрагироваться и от таких классов как Bitmap
и Canvas были написаны следующие
[алиасы](http://msdn.microsoft.com/en-us/library/aa664765(v=vs.71).aspx).
Также в разрабатываемой WP версии используется еще и [паттерн
адаптер](http://ru.wikipedia.org/wiki/%D0%90%D0%B4%D0%B0%D0%BF%D1%82%D0%B5%D1%80_(%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD_%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F)).

<anchor>Alias</anchor>

#### Использование alias

```CSharp
#if SCRIPTSHARP
using System.Html;
using System.Html.Media.Graphics;
using System.Runtime.CompilerServices;
using Bitmap = System.Html.CanvasElement;
using Graphics = System.Html.Media.Graphics.CanvasContext2D;
using ImageData = System.Html.Media.Graphics.ImageData;
using Image = System.Html.ImageElement;
#elif DOTNET
using System.Drawing;
using System.Drawing.Imaging;
using System.Drawing.Drawing2D;
using Bitmap = System.Drawing.Bitmap;
using Graphics = System.Drawing.Graphics;
using ImageData = System.Drawing.Imaging.BitmapData;
using Image = System.Drawing.Bitmap;
#endif
```

Однако в C\# к сожалению нельзя сделать алиасы на небезопасные типы и
массивы, т.е. так ([Alias to pointer (byte\*) in
C\#](http://stackoverflow.com/q/13489903/1046374)):

```CSharp
using PixelArray = byte*, using PixelArray = byte[]
```

И для того чтобы иметь возможность использовать неуправляемый код в C\#
для быстрой обработки пикселей, при этом одновременно компилирующемся в
Script\#, была введена такая схема с помощью директив:

```CSharp
#if SCRIPTSHARP
	PixelArray data = context.GetPixelArray();
#elif DOTNET
	byte* data = context.GetPixelArray();
#endif
```

В дальнейшем массив data используется для различных попиксельных
операций (таких как наложение масок, рыбий глаз, изменение насыщенности
и другие), распараллеленых и нет.

<anchor>Links</anchor>

#### Ссылки на файлы

Для каждой платформы в солюшн добавляется отдельный проект, но, понятное
дело, проекты Mono, Script\# и даже Silverlight не могут ссылаться на
обычную .NET сборку. К счастью в Visual Studio существует механизм
добавления ссылок на файлы, что позволяет повторно использовать один и
тот же код в разных проектах. А указание директив компиляции (DOTNET,
SCRIPTSHARP и т.д.) добавляется в свойствах проекта в Conditional
Compilation Symbols.

<anchor>DotnetNotes</anchor>

### Заметки о .NET реализации

Благодаря вышеупомянутым абстракциям и алиасам, был написан код C\# с
низким уровнем избыточности. Однако далее я хочу обратить внимание на
проблемы платформ .NET и JavaScript с которыми нам пришлось столкнуться
при разработке, но которые были успешно решены.

<anchor>DisposeUsing</anchor>

#### Использование Dispose

Хочется обратить внимание на то, что для любого экземпляра C\# класса,
реализующего интерфейс IDisposable, всегда нужно вызывать **Dispose**
после его использования или использовать паттерн **using**. В данном
проекте такими классами являлись Bitmap и Context. Это не только мои
слова и просто теория, но и практика: На ASP.NET Developer Server x86
сервере обработка большого количества больших фотографий (до 2400\*2400)
приводила к исключению связанному с памятью. После расстановки Dispose в
нужных местах, проблема исчезла. Об этом и о многих других советах по
обработке изображений также пишут в статье [20 Image Resizing
Pitfalls](http://www.nathanaeljones.com/blog/2009/20-image-resizing-pitfalls)
и [.NET Memory Leak: To dispose or not to dispose, that’s the 1 GB
question](http://blogs.msdn.com/b/tess/archive/2009/02/03/net-memory-leak-to-dispose-or-not-to-dispose-that-s-the-1-gb-question.aspx).

<anchor>LockUsing</anchor>

#### Использование lock

В JavaScript существует разделение между уже загруженной картинкой с
тегом **img**, которому можно задавать источник и событие загрузки и
между полотном с тегом canvas, на котором можно что-то рисовать. Однако
в .NET все представляется одним классом Bitmap. Таким образом, алиасы
Bitmap и Image в .NET указывают на один и тот же класс
System.Drawing.Bitmap как это можно увидеть выше. Тем не менее это
разделение в JavaScript на img и canvas очень помогло и в .NET версии в
дальнейшем. Дело в том, что для фильтров используются предварительно
загруженные маски, которые используются разными потоками, а
следовательно для них нужно использовать паттерн **lock** во избежание
исключения синхронизации (копирование изображения происходит с lock, а
дальше результат используется без блокировки):

```CSharp
internal static Bitmap CloneImage(Image image)
{
#if SCRIPTSHARP
	Bitmap result = (Bitmap)Document.CreateElement("canvas");
	result.Width = image.Width;
	result.Height = image.Height;
	Graphics context = (Graphics)result.GetContext(Rendering.Render2D);
	context.DrawImage(image, 0, 0);
	return result;
#else
	Bitmap result;
	lock (image)
		result = new Bitmap(image);
	return result;
#endif
}
```

Не стоит забывать, что lock нужно использовать и в случае обращения к
свойствам синхронизируемого объекта (потому что любые свойства это по
сути методы).

<anchor>MasksInMemory</anchor>

#### Хранение масок в памяти

При старте сервера все потенциально используемые маски для фильтров
загружаются в память для ускорения обработки. Не стоит забывать, что
какой бы формат не имела маска, на сервере загруженный Bitmap занимает
4\*2400\*2400 \~ 24 Мб (максимальный размер изображения 2400\*2400,
кол-во байт на пиксель - 4), а значит все маски для фильтров (\~30 штук)
и коллажей (40 штук) будет занимать в памяти \~1.5 Гб, что в принципе не
много для сервера, однако при увеличении количества масок, это число
может значительно увеличиться. Так что в будущем, возможно, нужно будет
использовать сжатие масок в памяти (в виде форматов .jpg, .png) с
последующей распаковкой их во время использования, особенно учитывая,
что размер при этом можно уменьшить примерно в 300 раз. Дополнительным
преимуществом такого подхода является то, что сжатое изображение
копировать быстрее большого, а значит на операцию **lock** будет уходить
меньше времени и потоки будут меньше блокироваться.

<anchor>JavascriptNotes</anchor>

### Заметки о JavaScript реализации

<anchor>Minification</anchor>

#### Минификация

Я специально не употребил слово "обфускация" в заголовке данного
раздела, поскольку этот термин для языка со все время открытыми
исходниками, которым в данном случае является JavaScript, слабоприменим.
Однако запутать логику и читаемость кода можно путем "обезличивания"
различных идентификаторов (не будем сейчас говорить об извращенных
способов, продемонстрированных [в одном из
топиков](http://habrahabr.ru/post/112530/)). Ну и главное, такая
методика позволит существенно уменьшить размер скрипта (сейчас в сжатом
виде он весит \~80 Кб). Существует два подхода минификации JavaScript в
нашем случае:

* **Вручную**. На этапе генерации, используя ScriptSharp.
* **Автоматически**. После этапа генерации. Для этого используются
    внешние инструменты, такие как Google Closure Compiler, Yui.

<anchor>ManualMinify</anchor>

##### Минификация вручную

Для того чтобы укоротить названия методов, классов и атрибутов,
использовалась такая синтаксическая конструкция непосредственно перед
объявлениями этих сущностей. Естественно для методов, которые вызываются
из внешних скриптов и классов (public), такое делать конечно же не
нужно.

```CSharp
#if SCRIPTSHARP && !DEBUG
    [ScriptName("a0")]
#endif
```

Однако локальные переменные при этом все равно нельзя минифицировать.
Также недостатком является засорение кода такими конструкциями и, как
следствие, ухудшение читаемости кода. Однако данная методика позволяет
существенно уменьшить и запутать генерируемый JavaScript код. Еще одним
недостатком является то, что за такими короткими именами нужно следить,
если они переименовывают названия методов (особенно перегруженных
абстрактных в потомках) и полей, потому что в этом случае Script\# не
будет ругаться на повторяющиеся имена. Однако дублирующихся классов он
не допустит. Кстати, в разрабатывающейся версии Script\# вроде бы уже
добавили минификацию private и internal методов и полей.

<anchor>AutoMinify</anchor>

##### Минификация автоматически

Для минификации JavaScript существует множество утилит, однако я
воспользовался Google Closure Compiler в силу бренда, хорошего качества
сжатия. Однако недостатком Google минификатором является то, что он не
может сжимать CSS файлы, зато
[YUI](http://yui.github.com/yuicompressor/css.html) например может. На
самом деле Script\# тоже минифицирует скрипты, но делает это существенно
хуже GCC. Стоит отметить, что у Google минификатора существует несколько
уровней сжатия: Whitespace, Simple и Advanced. В проекте был выбран
уровень Simple, поскольку хотя и при Advanced можно достигнуть
максимального качества сжатия, для него нужно писать кода особым
образом, чтобы методы и классы были доступны извне. Ну и частично такая
минификация была сделана вручную с помощью Script\#. Если вы
интересуетесь Google Closure минификатором, то рекомендую просмотреть
[данный список](http://javascript.ru/optimize/google-closure-compiler)
русскоязычных статей.

<anchor>DebugRelease</anchor>

#### Debug & Release режимы

Подключение debug и release версий библиотек к ASP.NET страницам
делалось следующим образом:

```html
<% if (Gfranq.JavaScriptFilters.HtmlHelper.IsDebug)
   { %>
    <script src="Scripts/mscorlib.debug.js" ></script>
    <script src="Scripts/imgProcLib.debug.js" ></script>
<% }
   else
   { %>
    <script src="Scripts/mscorlib.js" ></script>
    <script src="Scripts/imgProcLib.js" ></script>
<% } %>
```

Кстати, в нашем проекте минифицировались не только скрипты, но и файлы
описания фильтров тоже.

<anchor>crossOrigin</anchor>

#### Свойство crossOrigin

Для того, чтобы можно было иметь доступ к пикселям какого-то
изображения, его нужно преобразовать сначала в canvas. Однако при этом
может возникнуть ошибка кроссдоменного доступа (CORS). В нашем случае
данная проблема была разрешена следующим образом:

* Проставление `crossOrigin = ''` на клиенте.
* Добавление специального заголовка к пакету http на сервере.

Но так как ScriptSharp не поддерживает такого свойства для img
элементов, был написан такой код:

```CSharp
[Imported]
internal class AdvImage
{
	[IntrinsicProperty]
	internal string CrossOrigin
	{
		get { return string.Empty; }
		set { }
	}
}
```

А использовать его потом так:

```JavaScript
((AdvImage)(object)result).CrossOrigin = "";
```

Стоит отметить, что подобная техника позволяет добавить любое свойство к
объекту без ошибки компиляции. В частности, в ScriptSharp еще [не
реализовано свойство
wheelDelta](http://stackoverflow.com/q/13572711/1046374) (во всяком
случае в версии 0.7.5), отображающее величину вращения колесика
(используется в коллажах). Поэтому оно было реализовано подобным
образом. На самом деле такой грязный хак со свойствами - это плохо, а по
хорошему нужно вносить форки в проект. Но я пока честно говоря не совсем
разобрался как компилить ScriptSharp из исходников. На сервере для таких
изображений нужно возвращать такие заголовки (в Global.asax):

```CSharp
Response.AppendHeader("Access-Control-Allow-Origin", "\*");
```

Более подробно о кроссдоменном доступе к ресурсам можно почитать
[здесь](http://enable-cors.org/).

<anchor>Optimizations</anchor>

### Оптимизации

<anchor>Precalculs</anchor>

#### Использование предвычисленных (табличных) значений

Для некоторых операций, например изменение яркости, контрастности и
цветовых кривых была применена оптимизация, заключающаяся в
предварительном вычислении результирующих компонент цвета (r, g, b) для
всевозможных значений, а затем использование полученных массивов для
изменения непосредственно цветов пикселей. Однако стоит отметить, что
такая оптимизация подходит только для операций, в которых на цвет
результирующего пикселя не влияют соседние. Вычисление компонент цвета
для всевозможных значений:

```CSharp
for (int i = 0; i < 256; i++)
{
	r[i] = <actionFuncR>(i);
	g[i] = <actionFuncG>(i);
	b[i] = <actionFuncB>(i);
}
```

Использование предварительно вычисленных компонент цвета:

```CSharp
for (int i = 0; i < data.Length; i += 4)
{
	data[i] = r[data[i]];
	data[i + 1] = g[data[i + 1]];
	data[i + 2] = b[data[i + 2]];
}
```

Стоит отметить, что если такие табличные операции идут подряд, то
промежуточные изображения вообще можно не вычислять, а передавать только
массивы компонент цветов. Но в силу того, что на клиенте и сервере код
работал и так довольно быстро, пока что было решено не реализовывать
такую оптимизацию. К тому же существовали некоторые другие проблемы
из-за нее. Однако листинг такой оптимизации я все же приведу:

<table>
	<tr>
        <td>Обычный код</td>
        <td>Оптимизированный код</td>
    </tr>
	<tr>
	<td>
// Вычисление первой таблицы.
for (int i = 0; i &lt; 256; i++)
{
      r[i] = &lt;actionFunc1R&gt;(i);
      g[i] = &lt;actionFunc1G&gt;(i);
      b[i] = &lt;actionFunc1B&gt;(i);
}
…

// Вычисление результирующего промежуточного изображения.
for (int i = 0; i &lt; data.Length; i += 4)
{
      data[i] = r[data[i]];
      data[i + 1] = g[data[i + 1]];
      data[i + 2] = b[data[i + 2]];
}
…

// Вычисление второй таблицы.
for (int i = 0; i &lt; 256; i++)
{
      r[i] = &lt;actionFunc2R&gt;(i);
      g[i] = &lt;actionFunc2G&gt;(i);
      b[i] = &lt;actionFunc2B&gt;(i);
}
…

// Вычисление результирующего изображения.
for (int i = 0; i &lt; data.Length; i += 4)
{
      data[i] = r[data[i]];
      data[i + 1] = g[data[i + 1]];
      data[i + 2] = b[data[i + 2]];
}
…
    </td>
    <td>
// Вычисление первой таблицы.
for (int i = 0; i &lt; 256; i++)
{
      r[i] = &lt;actionFunc1R&gt;(i);
      g[i] = &lt;actionFunc1G&gt;(i);
      b[i] = &lt;actionFunc1B&gt;(i);
}
…

// Вычисление второй таблицы.
tr = r.Clone();
tg = g.Clone();
tb = b.Clone();
for (int i = 0; i &lt; 256; i++)
{
      r[i] = tr[&lt;actionFunc2R&gt;(i)];
      g[i] = tg[&lt;actionFunc2G&gt;(i)];
      b[i] = tb[&lt;actionFunc2B&gt;(i)];
}
…

// Вычисление результирующего изображения.
for (int i = 0; i &lt; data.Length; i += 4)
{ 	
      data[i] = r[data[i]];
      data[i + 1] = g[data[i + 1]];
      data[i + 2] = b[data[i + 2]];
}
…
    </td>
	</tr>
</table>

Однако даже это еще не все. Если посмотреть на правую таблицу, то можно
заметить, что новые массивы там создаются с помощью Clone. На самом деле
массив можно не копировать, а просто менять указатели на старый и новый
массивы (тут вспоминается аналогия с [двойной
буферизацией](http://ru.wikipedia.org/wiki/%D0%94%D0%B2%D0%BE%D0%B9%D0%BD%D0%B0%D1%8F_%D0%B1%D1%83%D1%84%D0%B5%D1%80%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F)).

<anchor>Pixels</anchor>

#### Преобразование изображение в массив пикселей

Профилировщиком JavaScript в Google Chrome было выявлено, что функция
GetImageData (которая используется для преобразования canvas в массив
пикселей) выполняется достаточно долго, о чем, впрочем, можно почитать в
различных статьях по оптимизации Canvas в JavaScript. Однако количество
вызовов данной функции тоже можно минимизировать. А именно, использовать
один и тот массив пикселей для попиксельных операций, по аналогии с
предыдущей оптимизацией.

<anchor>CodeSamples</anchor>

## Примеры кода

Здесь я опишу примеры кода, которые показались мне интересными и
полезными. Чтобы статья не получилась слишком длинной, я заключил их в
спойлеры.

<anchor>General</anchor>

### Общие

<anchor>IsNumericSample</anchor>

<spoiler title="Определение, является ли строка числом">

```CSharp
internal static bool IsNumeric(string n)
{
#if !SCRIPTSHARP
	return ((Number)int.Parse(n)).ToString() != "NaN";
#else
	double number;
	return double.TryParse(n, out number);
#endif
}
```

</spoiler>

<anchor>IntegerDivSample</anchor>

<spoiler title="Целочисленное деление">

```CSharp
internal static int Div(int n, int k)
{
	int result = n / k;
#if SCRIPTSHARP
	result = Math.Floor(n / k);
#endif
	return result;
}
```

</spoiler>

<anchor>ImageRotateFlipSample</anchor>

<spoiler title="Поворот и обращение изображения на canvas и Bitmap соответственно">

Обратите внимание, что в html5 canvas нет функций
поворота изображения на 90, 180 градусов, кроме как с использованием
матриц, а в .NET есть. Так что была написана соответствующая точная
функция, работающая с пикселями. Также стоит отметить, что в .NET версии
поворот на 90 градусов в любую сторону может привести к неправильным
результатам. Поэтому после использования функции RotateFlip в данных
случаях нужно создавать новый Bitmap.

```CSharp
public static Bitmap RotateFlip(Bitmap bitmap, RotFlipType rotFlipType)
{
#if SCRIPTSHARP
	int t, i4, j4, w, h, c;

	if (rotFlipType == RotFlipType.RotateNoneFlipNone)
		return bitmap;

	GraphicsContext context;
	PixelArray data;

	if (rotFlipType == RotFlipType.RotateNoneFlipX)
	{
		context = GraphicsContext.GetContext(bitmap);
		data = context.GetPixelArray();
		w = bitmap.Width;
		h = bitmap.Height;

		for (int i = 0; i < h; i++)
		{
			c = (i + 1) * w * 4 - 4;
			for (int j = 0; j < w / 2; j++)
			{
				i4 = (i * w + j) * 4;
				j4 = j * 4;

				t = (int)data[i4]; data[i4] = data[c - j4]; data[c - j4] = t;
				t = (int)data[i4 + 1]; data[i4 + 1] = data[c - j4 + 1]; data[c - j4 + 1] = t;
				t = (int)data[i4 + 2]; data[i4 + 2] = data[c - j4 + 2]; data[c - j4 + 2] = t;
				t = (int)data[i4 + 3]; data[i4 + 3] = data[c - j4 + 3]; data[c - j4 + 3] = t;
			}
		}
		context.PutImageData();
	}
	else if (rotFlipType == RotFlipType.Rotate180FlipNone || rotFlipType == RotFlipType.Rotate180FlipX)
	{
		context = GraphicsContext.GetContext(bitmap);
		data = context.GetPixelArray();
		w = bitmap.Width;
		h = bitmap.Height;
		c = w * 4 - 4;
		int dlength4 = data.Length - 4;
		for (int i = 0; i < data.Length / 4 / 2; i++)
		{
			i4 = i * 4;
			if (rotFlipType == RotFlipType.Rotate180FlipNone)
				j4 = i4;
			else
				j4 = (Math.Truncate((double)i / w) * w + (w - i % w)) * 4;

			t = (int)data[j4]; data[j4] = data[dlength4 - i4]; data[dlength4 - i4] = t;
			t = (int)data[j4 + 1]; data[j4 + 1] = data[dlength4 - i4 + 1]; data[dlength4 - i4 + 1] = t;
			t = (int)data[j4 + 2]; data[j4 + 2] = data[dlength4 - i4 + 2]; data[dlength4 - i4 + 2] = t;
			t = (int)data[j4 + 3]; data[j4 + 3] = data[dlength4 - i4 + 3]; data[dlength4 - i4 + 3] = t;
		}
		context.PutImageData();
	}
	else
	{
		Bitmap tempBitmap = PrivateUtils.CreateCloneBitmap(bitmap);
		GraphicsContext tempContext = GraphicsContext.GetContext(tempBitmap);
		PixelArray temp = tempContext.GetPixelArray();

		t = bitmap.Width;
		bitmap.Width = bitmap.Height;
		bitmap.Height = t;
		context = GraphicsContext.GetContext(bitmap);
		data = context.GetPixelArray();

		w = tempBitmap.Width;
		h = tempBitmap.Height;
		if (rotFlipType == RotFlipType.Rotate90FlipNone || rotFlipType == RotFlipType.Rotate90FlipX)
		{
			c = w * h - w;
			for (int i = 0; i < temp.Length / 4; i++)
			{
				t = Math.Truncate((double)i / h);
				if (rotFlipType == RotFlipType.Rotate90FlipNone)
					i4 = i * 4;
				else
					i4 = (t * h + (h - i % h)) * 4;
				j4 = (c - w * (i % h) + t) * 4; //j4 = (w * (h - 1 - i4 % h) + i4 / h) * 4;

				data[i4] = temp[j4];
				data[i4 + 1] = temp[j4 + 1];
				data[i4 + 2] = temp[j4 + 2];
				data[i4 + 3] = temp[j4 + 3];
			}
		}
		else if (rotFlipType == RotFlipType.Rotate270FlipNone || rotFlipType == RotFlipType.Rotate270FlipX)
		{
			c = w - 1;
			for (int i = 0; i < temp.Length / 4; i++)
			{
				t = Math.Truncate((double)i / h);
				if (rotFlipType == RotFlipType.Rotate270FlipNone)
					i4 = i * 4;
				else
					i4 = (t * h + (h - i % h)) * 4;
				j4 = (c + w * (i % h) - t) * 4; // j4 = w * (1 + i4 % h) - i4 / h - 1;

				data[i4] = temp[j4];
				data[i4 + 1] = temp[j4 + 1];
				data[i4 + 2] = temp[j4 + 2];
				data[i4 + 3] = temp[j4 + 3];
			}
		}
		context.PutImageData();
	}

	return bitmap;
#elif DOTNET
	Bitmap result = null;
	switch (rotFlipType)
	{
		case RotFlipType.RotateNoneFlipNone:
			result = bitmap;
			break;
		case RotFlipType.Rotate90FlipNone:
			bitmap.RotateFlip(RotateFlipType.Rotate90FlipNone);
			result = new Image(bitmap);
			bitmap.Dispose();
			break;
		case RotFlipType.Rotate270FlipNone:
			bitmap.RotateFlip(RotateFlipType.Rotate270FlipNone);
			result = new Image(bitmap);
			bitmap.Dispose();
			break;
		case RotFlipType.Rotate180FlipNone:
			bitmap.RotateFlip(RotateFlipType.Rotate180FlipNone);
			result = bitmap;
			break;
		case RotFlipType.RotateNoneFlipX:
			bitmap.RotateFlip(RotateFlipType.RotateNoneFlipX);
			result = bitmap;
			break;
		case RotFlipType.Rotate90FlipX:
			bitmap.RotateFlip(RotateFlipType.Rotate90FlipX);
			result = new Image(bitmap);
			bitmap.Dispose();
			break;
		case RotFlipType.Rotate180FlipX:
			bitmap.RotateFlip(RotateFlipType.Rotate180FlipX);
			result = bitmap;
			break;
		case RotFlipType.Rotate270FlipX:
			bitmap.RotateFlip(RotateFlipType.Rotate270FlipX);
			result = new Image(bitmap);
			bitmap.Dispose();
			break;
	}

	return result;
#endif
}
```
</spoiler>

<anchor>AsyncSyncImageLoadSample</anchor>

<spoiler title="Асинхронная и синхронная загрузка картинок">

Обратите внимание, что в ScriptSharp версии
указывается другая функция CollageImageLoad, которая вызовется после
загрузки изображения, в то время как в .NET версии все происходит
синхронно (из файловой системы или интернета).

```CSharp
public CollageData(string smallMaskPath, string bigMaskPath, List<CollageDataPart> dataParts)
{
	SmallMaskImagePath = smallMaskPath;
	BigMaskImagePath = bigMaskPath;
#if SCRIPTSHARP
	CurrentMask = PrivateUtils.CreateEmptyImage();
	CurrentMask.AddEventListener("load", CollageImageLoad, false);
	CurrentMask.Src = CurrentMaskImagePath;
#else
	CurrentMask = PrivateUtils.LoadBitmap(CurrentMaskImagePath);
	if (!CurrentMaskImagePath.Contains("http://") && !CurrentMaskImagePath.Contains("https://"))
		CurrentMask = Bitmap(CurrentMaskImagePath);
	else
	{
		var request = WebRequest.Create(CurrentMaskImagePath);
		using (var response = request.GetResponse())
			using (var stream = response.GetResponseStream())
				CurrentMask = (Bitmap)Bitmap.FromStream(stream);
	}
#endif
	DataParts = dataParts;
}
```
</spoiler>

<anchor>JavascriptOnly</anchor>

#### Только Script\#

<anchor>BrowserDetermSample</anchor>

<spoiler title="Определение типа и версии браузера">

Эта функция используется например для определения
возможностей drag & drop в различных браузерах (я пробовал использовать [modernizr](http://modernizr.com/), но он возвращал что Safari (в моем
случае для Win) и IE9 реализуют его. Однако на практике оказалось, что
эти браузеры реализуют drag & drop не совсем правильно).

```CSharp
internal static string BrowserVersion
{
	get
	{
		DetectBrowserTypeAndVersion();
		return _browserVersion;
	}
}

private static void DetectBrowserTypeAndVersion()
{
	if (!_browserDetected)
	{
		string userAgent = Window.Navigator.UserAgent.ToLowerCase();
		if (userAgent.IndexOf("opera") != -1)
			_browser = BrowserType.Opera;
		else if (userAgent.IndexOf("chrome") != -1)
			_browser = BrowserType.Chrome;
		else if (userAgent.IndexOf("safari") != -1)
			_browser = BrowserType.Safari;
		else if (userAgent.IndexOf("firefox") != -1)
			_browser = BrowserType.Firefox;
		else if (userAgent.IndexOf("msie") != -1)
		{
			int numberIndex = userAgent.IndexOf("msie") + 5;
			_browser = BrowserType.IE;
			_browserVersion = userAgent.Substring(numberIndex, userAgent.IndexOf(';', numberIndex));
		}
		else
			_browser = BrowserType.Unknown;
		_browserDetected = true;
	}
}
```
</spoiler>

<anchor>DashDotLineSample</anchor>

<spoiler title="Отрисовка пунктирной линии">

Используется для прямоугольника при обрезке
изображений. За идеи спасибо всем, кто отвечал в этом [вопросе на
stackoverflow](http://stackoverflow.com/q/4576724/1046374).

```CSharp
internal static void DrawDahsedLine(GraphicsContext context, double x1, double y1, double x2, double y2, int[] dashArray)
{
	if (dashArray == null)
		dashArray = new int[2] { 10, 5 };

	int dashCount = dashArray.Length;
	double dx = x2 - x1;
	double dy = y2 - y1;
	bool xSlope = Math.Abs(dx) > Math.Abs(dy);
	double slope = xSlope ? dy / dx : dx / dy;

	context.MoveTo(x1, y1);
	double distRemaining = Math.Sqrt(dx * dx + dy * dy);
	int dashIndex = 0;
	while (distRemaining >= 0.1)
	{
		int dashLength = (int)Math.Min(distRemaining, dashArray[dashIndex % dashCount]);
		double step = Math.Sqrt(dashLength * dashLength / (1 + slope * slope));
		if (xSlope)
		{
			if (dx < 0) step = -step;
			x1 += step;
			y1 += slope * step;
		}
		else
		{
			if (dy < 0) step = -step;
			x1 += slope * step;
			y1 += step;
		}
		if (dashIndex % 2 == 0)
			context.LineTo(x1, y1);
		else
			context.MoveTo(x1, y1);
		distRemaining -= dashLength;
		dashIndex++;
	}
}
```
</spoiler>

<anchor>RotateAnimationSample</anchor>

<spoiler title="Анимация вращения изображения">

Для анимации вращения изображения используется
функция setInterval. Обратите внимание, что просчет результирующего
изображения result происходит во время анимации, чтобы не было небольших
лагов в конце анимации.

```CSharp
public void Rotate(bool cw)
{
	if (!_rotating && !_flipping)
	{
		_rotating = true;
		_cw = cw;
		RotFlipType oldRotFlipType = _curRotFlipType;
		_curRotFlipType = RotateRotFlipValue(_curRotFlipType, _cw);

		int currentStep = 0;
		int stepCount = (int)(RotateFlipTimeSeconds * 1000 / StepTimeTicks);
		Bitmap result = null;
		_interval = Window.SetInterval(delegate()
		{
			if (currentStep < stepCount)
			{
				double absAngle = GetAngle(oldRotFlipType) + currentStep / stepCount * Math.PI / 2 * (_cw ? -1 : 1);
				DrawRotated(absAngle);
				currentStep++;
			}
			else
			{
				Window.ClearInterval(_interval);
				if (result != null)
					Draw(result);
				_rotating = false;
			}
		}, StepTimeTicks);

		result = GetCurrentTransformResult();
		if (!_rotating)
			Draw(result);
	}
}

private void DrawRotated(double rotAngle)
{
	_resultContext.FillColor = FillColor;
	_resultContext.FillRect(0, 0, _result.Width, _result.Height);

	_resultContext.Save();

	_resultContext._graphics.Translate(_result.Width / 2, _result.Height / 2);
	_resultContext._graphics.Rotate(-rotAngle);
	_resultContext._graphics.Translate(-_origin.Width / 2, -_origin.Height / 2);
	_resultContext._graphics.DrawImage(_origin, 0, 0);

	_resultContext.Restore();
}

private void Draw(Bitmap bitmap)
{
	_resultContext.FillColor = FillColor;
	_resultContext.FillRect(0, 0, _result.Width, _result.Height);

	_resultContext.Draw2(bitmap, (int)((_result.Width - bitmap.Width) / 2), (int)((_result.Height - bitmap.Height) / 2));
}
```
</spoiler>

<anchor>Conclusion</anchor>

## Заключение

В данной статье было показано какой большой кросплатформенностью может
обладать C\#, совмещая в себе с одной стороны неуправляемый код, а с
другой - компиляцию под JavaScript. Несмотря на то, что был сделан
основной упор на .NET и JavaScript, компиляция под Android, iOS (с
помощью Mono) и Windows Phone также возможна на основе описанного
подхода, естественно со своими подводными камнями. Конечно, избыточный
код при такой универсальности существует, однако на деле он никак не
влияет на производительность, поскольку графические операции занимают
несоизмеримо больше времени. Я думаю что материал, описанный в данной
статье, окажется полезным и без исходников. Также я готов прислушаться к
критике и ответить на ваши вопросы. В следующей статье, если эта будет
хорошо воспринята сообществом, я собираюсь описать интеграцию карт
google с ASP.NET и MS SQL с целью отображения, а также добавления и
изменения собственных фотографий. В ней будут освещаться такие моменты
как:

* Географические типы данных MS SQL (работа с ними в TSQL, C\#, описание их внутренней структуры).
* Кеширование фотографий определенной области на стороне клиента и сервера.
* Google maps и geocoding api (краткое описание функций навигации, кастомизации маркеров).

**UPDATE** Код JavaScript,
генерируемый Script\# с ручной и автоматической минификацией:
[imgProcLib](http://gfranq.com/Scripts/gfranq.imgProcLib3_17.00.js) Код
mscorlib (без ручной минификации):
[mscorlib](http://gfranq.com/Scripts/mscorlib3.js) Как можно заметить,
благодаря ручному проставлению коротких имен идентификаторам, код
удалось сильно сжать, по сравнению с используемой по умолчанию
библиотекой mscorlib.

**UPDATE 2** Статья по картам была написана и
опубликована [здесь](http://habrahabr.ru/post/182532/)
