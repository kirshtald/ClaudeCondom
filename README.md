# Настройка часового пояса для Claude Desktop и Claude Code CLI на Windows 11

**Версия руководства:** 1.0  
**Дата:** 11 ноября 2026 года  
**Целевой часовой пояс:** Europe/Amsterdam (Нидерланды)

## 1. Назначение

Руководство описывает настройку Claude Desktop и Claude Code CLI таким образом, чтобы они отображали время выбранного часового пояса независимо от системного часового пояса Windows 11.

В результате:

- Системные часы, регион и часовой пояс Windows остаются без изменений.
- Claude Desktop запускается с подменённым часовым поясом через Locale Remulator.
- Claude Code CLI отображает время нужного региона.
- Claude Desktop можно запускать через отдельный ярлык или автоматически при входе в Windows.
- После обновления MSIX-пакета Claude Desktop скрипт самостоятельно определяет новый путь к исполняемому файлу.

**Важно:** это не способ изменить географическое положение устройства или страну аккаунта Anthropic. VPN и IP-адрес настраиваются отдельно. Настройка времени интерфейса также не гарантирует изменения времени в облачных задачах Cowork или в командах Windows, выполняемых агентами.

## 2. Необходимые программы

Для выполнения руководства потребуются:

| Компонент | Назначение |
|---|---|
| Windows 11 | Основная операционная система |
| Claude Desktop | Настольное приложение Anthropic |
| Claude Code CLI | Терминальный агент программирования |
| Locale Remulator | Подмена часового пояса для Claude Desktop |
| PowerShell | Автоматизация запуска и настройки |

Ссылки:

- Claude Desktop: https://claude.com/download
- Claude Code: https://code.claude.com/docs
- Locale Remulator: https://github.com/InWILL/Locale_Remulator
- Релизы Locale Remulator: https://github.com/InWILL/Locale_Remulator/releases/latest

Для Locale Remulator могут потребоваться .NET Framework 4.8 и Microsoft Visual C++ Redistributable.

Проверенное решение использовало Locale Remulator 1.6.0 и MSIX-версию Claude Desktop на Windows 11. Совместимость с другими версиями необходимо проверять отдельно.

---

# Часть I. Настройка Claude Desktop

## 3. Установка Locale Remulator

1. Скачайте Locale Remulator из официального репозитория.
2. Распакуйте архив в постоянную папку, например:

   `C:\Tools\Locale_Remulator`

3. Запустите `LRInstaller.exe` и выполните установку.
4. Откройте `LREditor.exe`.

**Не переносите папку Locale Remulator после установки без необходимости:** зарегистрированные компоненты могут использовать её исходный путь.

## 4. Создание профиля Amsterdam

В LREditor создайте новый профиль со следующими параметрами.

| Настройка | Значение |
|---|---|
| Название профиля | Claude Amsterdam |
| CodePage | Оставить значение по умолчанию |
| TimeZone | (UTC+01:00) Амстердам, Берлин, Берн, Вена, Рим, Стокгольм |
| Run as Administrator | Выключено |
| Hook Input Method Editor | Выключено |
| Fake system UI Language | Выключено |
| Enter Arguments here | Пусто |

Выбранная зона Windows соответствует `W. Europe Standard Time`. Она предусматривает зимнее время UTC+1 и летнее UTC+2.

Нажмите **Save As...**, сохраните профиль под именем `Claude Amsterdam` и убедитесь, что он появился в списке.

## 5. Поиск установленного Claude Desktop

Откройте PowerShell и выполните:

```powershell
Get-AppxPackage -Name Claude |
    Select-Object Name, PackageFullName, InstallLocation
```

Если Claude Desktop установлен через MSIX, вы увидите путь следующего вида:

```text
C:\Program Files\WindowsApps\Claude_ВЕРСИЯ_x64__ИДЕНТИФИКАТОР
```

Исполняемый файл обычно расположен во вложенной папке:

```text
app\Claude.exe
```

Проверка:

```powershell
$pkg = Get-AppxPackage -Name Claude
$exe = Join-Path $pkg.InstallLocation "app\Claude.exe"

Test-Path $exe
```

Ожидаемый результат:

```text
True
```

## 6. Первый запуск через Locale Remulator

Перед запуском закройте обычный Claude Desktop, включая фоновый экземпляр в системном трее. Сначала завершите активные задачи Cowork.

В Locale Remulator выберите профиль `Claude Amsterdam`, затем используйте **Run** и укажите найденный `Claude.exe`.

Если программа откроется, проверьте временные метки в интерфейсе.

Не следует проверять результат только вопросом модели «Который час?»: ответ модели не доказывает, что изменился часовой пояс интерфейса.

Если Claude запускается и его интерфейс отображает нужное время, можно переходить к автоматизации.

---

# Часть II. Автоматизация Claude Desktop

## 7. Получение GUID профиля

Locale Remulator хранит конфигурации в файле `LRConfig.xml`.

Откройте этот файл в папке Locale Remulator.

Найдите сохранённый профиль `Claude Amsterdam` и его атрибут `Guid`.

Пример формата GUID:

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**Каждый пользователь должен указать собственный GUID.**

Правильный GUID содержит пять групп символов в формате `8-4-4-4-12`. Ошибка даже в одном символе может привести к тому, что Claude не запустится.

Для просмотра файла через PowerShell:

```powershell
$lrDir = "C:\Tools\Locale_Remulator"

Select-String `
    -Path "$lrDir\LRConfig.xml" `
    -Pattern "Claude Amsterdam|Guid=" `
    -Context 2,3
```

## 8. Скрипт автоматического запуска Claude Desktop

Создайте файл:

```text
%USERPROFILE%\Documents\Start-Claude-Amsterdam.ps1
```

Для создания можно выполнить:

```powershell
notepad "$env:USERPROFILE\Documents\Start-Claude-Amsterdam.ps1"
```

Вставьте следующий скрипт.

### Полный скрипт Start-Claude-Amsterdam.ps1

```powershell
$ErrorActionPreference = "Stop"

# =====================================
# НАСТРОЙКИ ПОЛЬЗОВАТЕЛЯ
# =====================================

# Папка установленного Locale Remulator
$lrDir = "C:\Tools\Locale_Remulator"

# GUID профиля Claude Amsterdam
$guid = "ВСТАВЬТЕ_СВОЙ_GUID"

# =====================================
# ПРОВЕРКА ПРОФИЛЯ
# =====================================

try {
    [guid]::Parse($guid) | Out-Null
}
catch {
    throw "Некорректный GUID профиля: $guid"
}

# =====================================
# ПОИСК АКТУАЛЬНОГО CLAUDE
# =====================================

$pkg = Get-AppxPackage -Name Claude |
    Sort-Object Version -Descending |
    Select-Object -First 1

if (-not $pkg) {
    throw "Claude Desktop не установлен."
}

$claudeExe = Join-Path `
    $pkg.InstallLocation "app\Claude.exe"

# =====================================
# ПОИСК LOCALE REMULATOR
# =====================================

$lrProc = Join-Path $lrDir "LRProc.exe"

if (-not (Test-Path $lrProc)) {
    throw "Не найден LRProc.exe: $lrProc"
}

if (-not (Test-Path $claudeExe)) {
    throw "Не найден Claude.exe: $claudeExe"
}

Write-Host "Locale Remulator: $lrProc"
Write-Host "Claude Desktop: $claudeExe"
Write-Host "Profile GUID: $guid"

# =====================================
# ЗАПУСК CLAUDE
# =====================================

# Для некоторых приложений важно
# запускаться из собственной папки

$claudeDir = Split-Path -Parent $claudeExe

Push-Location $claudeDir

try {
    & $lrProc $guid $claudeExe
}
finally {
    Pop-Location
}
```

### Что нужно изменить

В начале файла измените только два значения:

```powershell
$lrDir = "C:\Tools\Locale_Remulator"

$guid = "ВАШ_НАСТОЯЩИЙ_GUID"
```

Укажите реальное расположение Locale Remulator и GUID своего профиля.

**Не нужно изменять путь к Claude.exe вручную.** Скрипт сам определяет его с помощью `Get-AppxPackage` даже после обновления Claude Desktop.

### Почему в скрипте используется Push-Location

Во время настройки была обнаружена особенность: Locale Remulator может не запустить приложение при вызове из другой рабочей папки.

Команда:

```powershell
Push-Location $claudeDir
```

временно устанавливает рабочую папку Claude, а `Pop-Location` восстанавливает прежнюю.

Для использованной конфигурации такой способ запуска оказался рабочим.

## 9. Проверка скрипта

Полностью закройте Claude Desktop и выполните:

```powershell
& "$env:USERPROFILE\Documents\Start-Claude-Amsterdam.ps1"
```

Если появляется ошибка политики выполнения PowerShell, можно протестировать скрипт в отдельном процессе:

```powershell
powershell.exe -NoProfile `
    -ExecutionPolicy Bypass `
    -File "$env:USERPROFILE\Documents\Start-Claude-Amsterdam.ps1"
```

Параметр `-ExecutionPolicy Bypass` здесь относится только к запускаемому процессу PowerShell и не меняет постоянную политику всей системы. Используйте его только для доверенного локального скрипта.

При успешном запуске должен открыться Claude Desktop с настроенным часовым поясом.

## 10. Создание ярлыка

После проверки скрипта создайте отдельный ярлык.

Откройте PowerShell:

```powershell
$script = Join-Path $env:USERPROFILE `
    "Documents\Start-Claude-Amsterdam.ps1"

$desktop = [Environment]::GetFolderPath("Desktop")

$wsh = New-Object -ComObject WScript.Shell

$link = $wsh.CreateShortcut(
    (Join-Path $desktop "Claude Amsterdam.lnk")
)

$link.TargetPath = "powershell.exe"

$link.Arguments = (
    '-NoProfile -ExecutionPolicy Bypass ' +
    '-WindowStyle Hidden -File "' +
    $script + '"'
)

$link.Description = (
    "Claude Desktop - Europe/Amsterdam"
)

$link.Save()
```

На рабочем столе появится ярлык:

**Claude Amsterdam**

Он запускает скрипт в скрытом окне PowerShell. В некоторых конфигурациях Windows консоль может кратковременно появляться при запуске.

### Если ярлык не работает

В свойствах ярлыка временно уберите:

```text
-WindowStyle Hidden
```

Добавьте:

```text
-NoExit
```

После этого окно PowerShell останется открытым и можно будет увидеть сообщение об ошибке.

Когда проблема устранена, верните обычный скрытый запуск.

## 11. Автозапуск вместе с Windows

Для автоматического запуска Claude Desktop:

1. Нажмите `Win + R`.
2. Введите `shell:startup`.
3. Скопируйте в открывшуюся папку ярлык `Claude Amsterdam.lnk`.
4. Отключите штатный автозапуск Claude в параметрах Windows или в самом Claude, чтобы не было двух конкурирующих способов запуска.

Теперь при входе в Windows будет запускаться скрипт Locale Remulator.

Настройки часового пояса Windows не изменяются.

---

# Часть III. Настройка Claude Code CLI

## 12. Проверка версии Claude Code

Откройте PowerShell:

```powershell
claude --version
```

Для настройки отображения времени через `timeZone` и `timeFormat` необходим Claude Code версии **2.1.257 или новее**.

Эти параметры предусмотрены официальной документацией Anthropic.

## 13. Настройка часового пояса CLI

Откройте файл:

```powershell
notepad "$env:USERPROFILE\.claude\settings.json"
```

Если файл существует, сохраните его остальные настройки.

Добавьте свойства:

```json
{
  "timeZone": "Europe/Amsterdam",
  "timeFormat": "24-hour"
}
```

Если файл уже содержит другие поля, добавьте новые свойства внутрь существующего JSON-объекта. Не создавайте второй корневой объект.

Здесь:

- `timeZone` устанавливает часовой пояс для поддерживаемых временных меток интерфейса.
- `timeFormat` включает 24-часовой формат.

Значение `Europe/Amsterdam` — идентификатор IANA. Claude Code использует его с учётом переходов между CET и CEST.

## 14. Автоматическое изменение settings.json через PowerShell

Вместо ручного редактирования можно использовать следующий скрипт. Он сохраняет другие настройки и делает резервную копию существующего файла.

```powershell
$ErrorActionPreference = "Stop"

$dir = Join-Path $env:USERPROFILE ".claude"
$path = Join-Path $dir "settings.json"

New-Item -ItemType Directory `
    -Path $dir -Force | Out-Null

if (Test-Path $path) {
    $raw = Get-Content $path -Raw -Encoding UTF8

    if ([string]::IsNullOrWhiteSpace($raw)) {
        $config = [PSCustomObject]@{}
    }
    else {
        $config = $raw | ConvertFrom-Json
    }

    $backup = "$path.bak-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
    Copy-Item $path $backup
}
else {
    $config = [PSCustomObject]@{}
}

$config | Add-Member `
    -NotePropertyName "timeZone" `
    -NotePropertyValue "Europe/Amsterdam" `
    -Force

$config | Add-Member `
    -NotePropertyName "timeFormat" `
    -NotePropertyValue "24-hour" `
    -Force

$json = $config | ConvertTo-Json -Depth 100

[System.IO.File]::WriteAllText(
    $path,
    $json,
    [System.Text.UTF8Encoding]::new($false)
)

Write-Host "Часовой пояс Claude Code настроен."
```

После выполнения проверьте значения:

```powershell
Get-Content "$env:USERPROFILE\.claude\settings.json" -Raw |
    ConvertFrom-Json |
    Select-Object timeZone, timeFormat
```

Ожидаемый результат:

```text
timeZone          timeFormat
--------          ----------
Europe/Amsterdam  24-hour
```

Затем полностью завершите текущую сессию Claude Code и запустите новую:

```powershell
claude
```

Для интерфейса CLI дополнительный запуск через Locale Remulator не требуется.

**Ограничение:** параметры `timeZone` и `timeFormat` не меняют системный часовой пояс Windows и не гарантируют, что выполняемые Claude команды, например `Get-Date`, будут использовать Амстердам.

---

# Часть IV. Решение типичных проблем

## 15. Claude Desktop не запускается через скрипт

### Проблема: неверный GUID

Проверьте атрибут `Guid` в `LRConfig.xml`.

Он должен состоять из 36 символов, включая дефисы, и соответствовать формату:

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

GUID нельзя сокращать или переносить в значение скрипта с пробелами.

### Проблема: неправильная папка Locale Remulator

Проверьте:

```powershell
Test-Path "C:\Tools\Locale_Remulator\LRProc.exe"
```

Если результат `False`, исправьте `$lrDir` в скрипте.

### Проблема: обновился Claude Desktop

Проверьте:

```powershell
Get-AppxPackage -Name Claude |
    Select-Object PackageFullName, InstallLocation
```

Подготовленный скрипт автоматически ищет новую версию. Если запуск перестал работать после обновления, проверьте совместимость Locale Remulator с новой сборкой.

### Проблема: обычный Claude уже запущен

Полностью завершите его работу, включая процессы в трее, и повторите запуск через ярлык.

### Проблема: Claude не принимает параметры отладки

Некоторые сборки Claude Desktop отказываются запускаться с параметрами вроде:

```text
--remote-debugging-port=49287
```

Это отдельная защита приложения. Она не требуется для описанного решения через Locale Remulator. Не нужно пытаться отключить её.

## 16. Claude Code CLI продолжает показывать время Windows

Проверьте:

```powershell
claude --version
```

Затем:

```powershell
Get-Content "$env:USERPROFILE\.claude\settings.json" -Raw |
    ConvertFrom-Json |
    Select-Object timeZone, timeFormat
```

Если значения пустые, настройки не сохранены корректно.

Также проверьте, что используется пользовательский конфигурационный каталог по умолчанию и что настройки проекта или организации их не переопределяют.

После исправления перезапустите CLI.

## 17. Меры предосторожности

Locale Remulator — сторонняя утилита, использующая перехват функций Windows. Это не официальная функция Anthropic.

- Загружайте её из доверенного репозитория и проверяйте происхождение программы.
- Не изменяйте содержимое `WindowsApps` и разрешения этой защищённой папки.
- Не запускайте Claude с административными правами без необходимости.
- Не отключайте защитные функции Windows ради запуска.
- Проверяйте поведение Claude после обновлений.
- Не рассчитывайте, что настройки Desktop автоматически распространяются на все дочерние процессы и облачное окружение Cowork.

---

# Часть V. Итог

| Компонент | Часовой пояс | Метод настройки |
|---|---|---|
| Основная Windows 11 | Исходный | Не меняется |
| Claude Desktop | Europe/Amsterdam | Locale Remulator + скрипт |
| Claude Code CLI | Europe/Amsterdam | settings.json |
| Дочерние процессы и команды Windows | Зависит от среды | Отдельная настройка |
| Claude Cowork | Зависит от режима выполнения | Требует отдельной проверки |
| VPN | Независимая настройка | Настраивается отдельно |

**Итоговый результат:** Claude Desktop может запускаться с подменённым часовым поясом, а Claude Code CLI отображает время Амстердама без изменения системного времени ноутбука.

Для Claude Desktop решение было практически проверено в конкретной конфигурации Windows 11, Claude MSIX и Locale Remulator. Для Claude Code CLI было подтверждено корректное отображение времени после изменения `settings.json`.

На других компьютерах необходимо создать собственный профиль Locale Remulator, указать его GUID, проверить пути к программам и убедиться, что используемые версии совместимы.

## Использованные источники

1. Locale Remulator — https://github.com/InWILL/Locale_Remulator
2. Locale Remulator Releases — https://github.com/InWILL/Locale_Remulator/releases
3. Claude Desktop — https://claude.com/download
4. Claude Code Settings — https://code.claude.com/docs/en/settings
5. Claude Code Settings Reference — https://code.claude.com/docs/en/settings-reference
6. Locale Remulator: проблема рабочей директории при запуске — https://github.com/InWILL/Locale_Remulator/issues/52
