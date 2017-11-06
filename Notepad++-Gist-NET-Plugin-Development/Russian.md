# Разработка .NET плагина для работы с Gist в Notepad++

![Notepad++ love GitHub](Images/NPP-Love-GitHub.png)

Однажды мне потребовалось создать Gist, фрагмент кода,
которым можно делиться с коллегами. А еще я активный пользователь Notepad++.
После того, как найти плагин для работы с Gist в Notepad++ не удалось
(есть только под Sublime), я решил написать свой. К тому же
это добавило опыта разработки плагинов и работы с GitHub API.
Сразу выкладываю ссылку на [исходники](https://github.com/KvanTTT/NppGist) и
сборки самого плагина [NppGist-x86-1.1.0.10](https://github.com/KvanTTT/NppGist/releases/download/1.2/NppGist-x86-1.1.0.10.zip),
[NppGist-x64-1.1.0.10](https://github.com/KvanTTT/NppGist/releases/download/1.2/NppGist-x64-1.1.0.10.zip).
Для его подключения нужно перенести соответствующий файл `NppGist.dll` в папку
plugins, которая находится в папке с установленным Notepad++. Плагины можно
разрабатывать на нескольких языках: C++, Ada, Delphi, C#, но я остановился на
последнем из-за его актуальности и большого опыта работы с ним. При разработке
использовались следующие библиотеки и инструменты:

1. [**NppPlugin.NET**](http://sourceforge.net/projects/sourcecookifier/files/other%20plugins/NppPlugin.NET.v0.5.zip/download) -
   шаблон Notepad++ плагина для .NET платформы.
2. [**ServiceStack.Text**](https://github.com/ServiceStack/ServiceStack.Text) -
   библиотека для сериализация и десериализация JSON. Имеет высокую
   производительность и небольшой размер).
3. **[hurl.it](http://www.hurl.it/)** - удобный онлайн-инструмент для
   составления и тестирования GET, POST, DELETE и других запросов.
4. **NUnit** - юнит-тестирование.

## Инициализация плагина

Взаимодействие с Notepad++ происходит посредством Win32 сообщений. Но, к
счастью, под .NET уже написан готовый шаблон плагина со многими
сообщениями, классами и структурами
([NppPlugin.NET.v0.5](http://sourceforge.net/projects/sourcecookifier/files/other%20plugins/NppPlugin.NET.v0.5.zip/download)).
Для корректной компиляции плагина в Visual Studio **Platform taget** нужно
устанавливать в **x86** или **x64**, вместо **Any CPU** по-умолчанию, а также
использовать .NET не ниже версии 4.0. Инициализация плагина происходит в методах
`CommandMenuInit` и `SetToolBarIcon`. В первый добавляются пункты, которые
отображаются в меню плагина. Делается это следующим образом:

```CSharp
PluginBase.SetCommand(OpenCommandId, "Open Gist", OpenGistCommand,
    new ShortcutKey(false, false, false, Keys.None));
```

Там же можно определить комбинации клавиш для команд
(в разработанном плагине они не используются). Метод `OpenGistCommand`
описывается разработчиком, в нем прописана логика открытия окна с выбором гиста.
В методе `SetToolBarIcon` добавляются иконки с командами плагина (Open, Save)
в панель инструментов Notepad++.

```CSharp
toolbarIcons tbIcons = new toolbarIcons();
tbIcons.hToolbarBmp = tbLoad.GetHbitmap();
IntPtr pTbIcons = Marshal.AllocHGlobal(Marshal.SizeOf(tbIcons));
Marshal.StructureToPtr(tbIcons, pTbIcons, false);
Win32.SendMessage(PluginBase.nppData._nppHandle, NppMsg.NPPM_ADDTOOLBARICON,
    PluginBase._funcItems.Items[OpenCommandId]._cmdID, pTbIcons);
Marshal.FreeHGlobal(pTbIcons);
```

## Сохранение и загрузка настроек

Для сохранения и загрузки параметров плагина, используются следующие методы:

```CSharp
saveLocally = Convert.ToBoolean(Win32.GetPrivateProfileInt("Settings", "SaveLocally", 1, IniFileName));
//...
Win32.WritePrivateProfileString("Settings", "SaveLocally",
    (Convert.ToInt32(saveLocally)).ToString(), Main.IniFileName);
```

## Выполнение команд

Notepad++ использует компонент Scintilla, который используется и в других
редакторах текста. С обоими компонентами можно общаться посредством сообщений.
Все возможные коды для сообщений прописаны в файле *NppPluginNETHelper.cs*. Notepad++
сообщения имеют префикс `NPPM` и они отвечают за работу с файлами, меню,
табами, языками и пр. Scintilla сообщения нужны непосредственно для работы с
текстом (вставка, удаление, выделение, настройка визуальных стилей, фолдинг,
скроллинг и т.д.).

## Перехват событий

Для перехвата событий в Notepad++, используется метод
`beNotified` в файле *UnmanagedExports.cs*. Данные события имеют
префикс `NPPN` для Notepad++ (открытие, закрытие файла,
переключение вкладок) и `SCN` для Scintilla (изменение текста).
В данном плагине перехват событий не используется. Полный список и
подробное описание команд по Notepad++ находится в [Messages And
Notifications](http://docs.notepad-plus-plus.org/index.php/Messages_And_Notifications).
А по Scintilla здесь: [ScintillaDoc](http://www.scintilla.org/ScintillaDoc.html).

## Корректное получение UTF8 текста

В .NET плагине почему-то нельзя получить текст в UTF8 формате, хотя эта
кодировка является самой распространенной. Для решения этой проблемы
было написано следующее свойство, в котором происходит корректная конвертация
текста, в том числе и русского. Это свойство вызывается при загрузке гиста
на GitHub.

```CSharp
public string lpstrTextUtf8
{
    get
    {
        _readNativeStruct();
        int len = 0;
        while (Marshal.ReadByte(_sciTextRange.lpstrText, len) != 0)
            ++len;
        if (len == 0)
            return string.Empty;
        byte[] buffer = new byte[len];
        Marshal.Copy(_sciTextRange.lpstrText, buffer, 0, buffer.Length);
        return Encoding.UTF8.GetString(buffer);
    }
}
```

## Встраивание сборок

Notepad++ загружает плагины из всех .dll файлов, находящихся в папке plugins.
Причем если плагин из dll загрузить не удалось, выводится сообщение с текстом
*The plugin is not compatible with current version of Notepad++*.
Таким образом, если в эту папку вместе с сами плагином копировать и его
зависимости (в нашем случае *ServiceStack.Text.dll*), то для каждой такой
зависимости будет отображаться сообщение об ошибке, что, конечно, не является
приемлемым. Решить эту проблему можно двумя способами:

* Помещать все зависимости в отдельную подпапку.
* Объединять плагин и его зависимости в одну сборку.

Я воспользовался вторым способом, т.к. один файл удобней распространять и
копировать. Для этого сторонние сборки помечались как **Embedded Resource**
и динамически подключались к плагину во время инициализации следующим образом:

```CSharp
static Main()
{
    AppDomain.CurrentDomain.AssemblyResolve += ResolveEventHandler;
}

private static Assembly ResolveEventHandler(object sender, ResolveEventArgs args)
{
    string resource = string.Format("{0}.{1}.dll", PluginName, args.Name.Remove(args.Name.IndexOf(',')));
    Assembly currentAssembly = Assembly.GetExecutingAssembly();
    using (Stream stream = currentAssembly.GetManifestResourceStream(resource))
    {
        var bytes = new byte[(int)stream.Length];
        stream.Read(bytes, 0, (int)stream.Length);
        return Assembly.Load(bytes);
    }
}
```

Подробнее о динамической загрузке сборок написано в статье
[Load DLL From Embedded Resource](http://www.codeproject.com/Articles/528178/Load-DLL-From-Embedded-Resource)
на CodeProject.

Для того чтобы данный способ работал, все зависимости должны находиться в
корневой папке проекта. Поэтому необходимо использовать Pre-build событие для
перемещения сборок в папку проекта, так как nuget по-умолчанию закачивает их
в другие папки.

Также можно воспользоваться сторонней программой для объединения сборок в одну,
[ILMerge](https://github.com/Microsoft/ILMerge). Однако ее пришлось бы запускать
уже в Post-build событии.

## GitHub Api

Для авторизации на GitHub используется AccessToken, который нужно получить в
[свойствах аккаунта на GitHub](https://github.com/settings/tokens/new).
Он используется во все запросах в виде параметра **access\_token**.
С полным списком используемых GitHub Gist API методов можно
ознакомиться [по ссылке](https://developer.github.com/v3/gists/).
Анонимные гисты в разработанном плагине не поддерживаются.

## Подмодуль NppNetInf

При разморозке проекта в 2017 скомпилировать рабочую x64 версию плагина не удалось:
пиктограммы загрузки и сохранения не добавлялись в Toolbar. Для решения этой
проблемы я задался вопросом, существуют ли другие x64 .NET плагины и правильно
ли они работают? После недолгих поисков был найден шаблон [NotepadPlusPlusPluginPack.Net](https://github.com/kbilsted/NotepadPlusPlusPluginPack.Net)
и плагин для рендеринга Markdown на его основе [MarkdownViewerPlusPlus](https://github.com/nea/MarkdownViewerPlusPlus),
который добавлял пиктограммы для обеих платформ x86 и x64 правильно.

Однако в самом шаблоне не понравилась громоздкость и зависимость от Visual C++.
И я задался другим вопросом: а можно ли убрать зависимость от Visual С++ и
сделать легкое и простое подключение инфраструктуры?

К счастью, поставленный вопрос был успешно решен, а такая инфраструктура
разработана: [NppNetInf](https://github.com/KvanTTT/NppNetInf). Для ее
подключения нужно выполнить несколько простых шагов:

1. Создать репозиторий с .NET проектом **MyAwesomePlugin**.
2. Добавить подмодуль <https://github.com/KvanTTT/NppNetInf.git> в репозиторий.
3. Добавить таргет в файл MyAwesomePlugin.csproj следующим образом:
   `<Import Project="$(SolutionDir)NppNetInf\src\DllExport\NppPlugin.DllExport.targets" />`
   Этот таргет будет выполняться после каждого билда. Он меняет финальную сборку
   специальным образом, чтобы Notepad++ "понимал" ее.
4. Создать поддиректорию NppNetInf внутри проекта MyAwesomePlugin и добавить
   все `*.cs` файлы *как ссылки* из закачанного подмодуля NppNetInf
  (Win32.cs, Scintilla.cs, PluginMain.cs и пр.).
5. Добавить класс `Main` и унаследовать его от `PluginMain`. Далее переопределить
   в нем необходимые абстрактный методы и свойства
   (`PluginName`, `CommandMenuInit`). Опционально можно переопределить и
   другие методы
   (`OnNotification(ScNotification notification)`, `PluginCleanUp()`, `SetToolBarIcon`).
6. Выбрать платформу (x86 или x64), построить проект и переместить
   скомпилированную сборку в соответствующую директорию плагинов Notepad++.
   Обычно это `C:\Program Files (x86)\Notepad++\plugins` для x86 платформы и
   `C:\Program Files\Notepad++\plugins` для x64.
7. Запустить Notepad++ и наслаждаться работой вашего классного плагина!

Для удаления зависимости от Visual C++ достаточно было просто удалить
атрибуты `LibToolPath` и `LibToolDllPath` в файле `NppPlugin.DllExport.targets`
(не знаю зачем они вообще были добавлены разработчиками).

NppNetInf при старте пытается найти класс плагина `Main`, реализующий
`PluginMain` с помощью рефлексии:

```CSharp
Type pluginMain = Assembly.GetExecutingAssembly().GetTypes()
    .FirstOrDefault(type => type.IsSubclassOf(typeof(PluginMain)));
_main = (PluginMain)Activator.CreateInstance(pluginMain);
```

К сожалению, полную модульность пока что реализовать не удалось (подключение с
помощью NuGet пакета, а не подмодуля). Однако и при текущей реализации проект
инфраструктуры добавлять в солюшн плагина не обязательно.

## Заключение

Окно сохранения гиста выглядит так:

![NppGist Sample Screen](Images/Sample-Screen.png)

При инициализации плагина необходимо ввести ваш access token.

Также в проект внедрена непрерывная интеграция AppVeyor для x86 и x64 платформ.
Результаты билда, тестов и готовые сборки доступны
[по ссылке](https://ci.appveyor.com/project/KvanTTT/nppgist/branch/master).

Надеюсь после прочтения всем желающим станет проще писать плагины под Notepad++.
Используйте плагин и присоединяйтесь к его разработке!
