# Разработка плагина для работы с Gist в Notepad++

![Notepad++ love GitHub](Images/NPP-Love-GitHub.png)

Однажды мне потребовалось создать gist, приватный или публичный фрагмент кода,
которым можно делиться с коллегами. А еще я активный пользователь Notepad++.
После того, как найти плагин для работы с gist в Notepad++ мне не удалось
(есть только под Sublime), я решил написать свой. К тому же
это добавило опыта разработки плагинов и работы с GitHub API.
Сразу выкладываю ссылку на исходники: [NppGist
sources](https://github.com/KvanTTT/NppGist) и сам плагин:
[NppGist](https://github.com/KvanTTT/NppGist/releases/download/1.1/NppGist.dll)
(для его подключения просто перенесите файл в папку plugins, которая находится
в папке с установленным Notepad++). Под Notepad++ плагины можно писать на
нескольких языках: C++, Ada, Delphi, C#, но я остановился на последнем из-за
скорости разработки и большого опыта работы с ним. Для разработки были
использованы следующие библиотеки и инструменты:

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
устанавливать в **x86**, вместо **Any CPU** по-умолчанию, а также использовать
.NET не ниже версии 4.0. Инициализация плагина происходит в методах
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

## Выполнение команд в Notepad++

Notepad++ использует компонент Scintilla, который используется и в других
редакторах текста. С обоими компонентами можно общаться посредством сообщений.
Все возможные коды для сообщений прописаны в файле *NppPluginNETHelper.cs*. Notepad++
сообщения имеют префикс `NPPM` и они отвечают за работу с файлами, меню,
табами, языками и пр. Scintilla сообщения нужны непосредственно для работы с
текстом (вставка, удаление, выделение, настройка визуальных стилей, фолдинг,
скроллинг и т.д.).

## Перехват событий Notepad++

Для перехвата событий в Notepad++, используется метод
`beNotified` в файле *UnmanagedExports.cs*. Данные события имеют
префикс `NPPN` для Notepad++ (открытие, закрытие файла,
переключение вкладок) и `SCN` для Scintilla (изменение текста).
В данном плагине перехват событий не используется. Полный список и
подробное описание команд по Notepad++ находится здесь: [Messages And
Notifications](http://docs.notepad-plus-plus.org/index.php/Messages_And_Notifications).
А по Scintilla здесь: [ScintillaDoc](http://www.scintilla.org/ScintillaDoc.html).

## Корректное получение UTF8 текста из Notepad++

В .NET оболочке плагина почему-то нельзя получить текст в UTF8 формате,
хотя эта кодировка является самой распространенной. Для решения этой проблемы
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

* Складирование всех зависимостей  в отдельную подпапку.
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

Подробнее о динамической загрузке сборок написано в в статье на
CodeProject: [Load DLL From Embedded Resource](http://www.codeproject.com/Articles/528178/Load-DLL-From-Embedded-Resource).

Для того чтобы данный способ работал, все зависимости должны находиться в
корневой папке проекта. Поэтому необходимо использовать Pre-build событие для
перемещения сборок в папку проекта, так как nuget по-умолчанию закачивает их
в другие папки.

Также можно воспользоваться сторонней программой для объединения сборок в одну,
[ILMerge](https://github.com/Microsoft/ILMerge). Однако ее пришлось бы
запускать уже в Post-build событии.

## GitHub Api

Для авторизации на GitHub используется AccessToken, который нужно получить [на
сайте](https://github.com/settings/tokens/new). Он используется во все запросах
в виде параметра **access\_token**. Анонимные гисты в разработанном плагине не
поддерживаются. С полным списком используемых GitHub Gist API методов можно
ознакомиться [по ссылке](https://developer.github.com/v3/gists/).

## Заключение

Окно сохранения гиста выглядит так:

![NppGist Sample Screen](Images/Sample-Screen.png)

Но при инициализации плагина необходимо ввести ваш access token.

Надеюсь после прочтения всем желающим станет проще писать плагины под Notepad++.
Используйте плагин и присоединяйтесь к разработке!
