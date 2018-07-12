---
title: Chef сборщик
sidebar: doc_sidebar
permalink: chef.html
folder: definition
---

### Chef сборщик

Chef сборщик — это тип сборщика, использующий chef рецепты в качестве инструкций для сборки образов. Для организации кода используются cookbook'и:

* один основной [cookbook dapp](#cookbook-dapp);
* один или несколько опциональных [dimod](#dimod).

### Chef dimg

Chef dimg — это dimg собираемый chef сборщиком.

### Chef директория

Chef директория — это директория \<[директория dapp](definitions.html#директория-dapp)\>/.dapp_chef. Содержит файлы [cookbook dapp](#cookbook-dapp).

### Cookbook dapp

Cookbook dapp / cookbook сборщика — это основной chef cookbook, связанный с [dapp](definitions.html#dapp), содержащий инструкции сборки образов.

* Cookbook dapp может подключать [модули](#dimod) для расширения инструкций сборки.
* Cookbook dapp может объявить в зависимостях обыкновенные cookbook'и.

#### Организация файлов

* Структура файлов.
  * Атрибуты.
    * Файлы атрибутов не поддерживаются.
    * Хеш normal-атрибутов задается через [Dappfile](definitions.html#dappfile), см. [chef.attribute](chef_directives.html#chef-attributes), [chef.\<стадия\>\_attribute](chef_directives.html#chef-<стадия>_attributes).
  * Файлы.
    * Директория files/\<стадия\>/\<рецепт\> — содержит файлы рецепта для стадии, опциональна.
    * Директория files/\<стадия\>/common — содержит общие файлы для стадии, опциональна.
    * Недопустимо наличие одинаковых имен файлов в директории рецепта и common.
  * Шаблоны.
    * Директория templates/\<стадия\>/\<рецепт\> — содержит файлы шаблонов рецепта для стадии, опциональна.
    * Директория templates/\<стадия\>/common — содержит общие файлы шаблонов для стадии, опциональна.
    * Недопустимо наличие одинаковых имен файлов в директории рецепта и common.
  * Рецепты.
    * Директория recipes/\<стадия\> — содержит файлы рецептов для стадии, опциональна.

### Dimod

Dimod — это модуль cookbook сборщика.

* Дополнительный chef cookbook, который подключается к сборке [chef dimg](#chef-dimg).
* Имя cookbook'а должно начинаться с префикса 'dimod-'.

#### Включение в dapp

Фактически dimod может находится:

* В отдельном месте в файловой системе.
* В отдельном git-репозитории.
* В chef supermarket.

Чтобы подключить модуль к проекту, надо:

* Включить модуль в [Dappfile](definitions.html#dappfile) директивой [chef.dimod](chef_directives.html#chef-dimod-<mod>-<version-constraint>-<cookbook-opts>).
  * Т.к. dimod по факту является chef cookbook'ом, директива chef.dimod поддерживает те же опции, что и директива [chef.cookbook](chef_directives.html#chef-cookbook-<cookbook>-<version-constraint>-<cookbook-opts>).
    * Явно указывать chef.cookbook в дополнение к chef.dimod не надо.

#### Организация файлов

Все файлы описанные далее файлы должны находится в [chef директории dapp](#chef-директория).

* Атрибуты.
  * Файл attributes/\<стадия\>.rb — содержит атрибуты для стадии, опционален.
  * Файл attributes/common.rb — содержит атрибуты для всех стадий, опционален.
* Файлы.
  * Директория files/\<стадия\> — содержит файлы, доступные для стадии, опциональна.
  * Директория files/common — содержит файлы, доступные для всех стадий, опциональна.
  * Недопустимо наличие одинаковых имен файлов в директории стадии и common.
* Шаблоны.
  * Директория template/\<стадия\> — содержит файлы, доступные для стадии, опциональна.
  * Директория template/common — содержит файлы, доступные для всех стадий, опциональна.
  * Недопустимо наличие одинаковых имен файлов в директории стадии и common.
* Рецепты.
  * Файл recipes/\<стадия\>.rb — файл рецепта модуля для стадии, опционален.

### Стадия cookbook'а

Стадия cookbook'а — это часть cookbook'а, которая используется при сборке стадии для cookbook'а приложения и модулей dimod.

* Понятие применимо только к cookbook'у приложения и модулям dimod.
* Для всех остальных cookbook'ов при сборке стадии используется все файлы cookbook'а без изменений в файловой структуре.

### Установка стадии cookbook'а
Установка стадии cookbook'а — это процесс копирования файлов стадии cookbook'а во [временное хранилище](definitions.html#временная-директория-приложения), подключаемое в дальнейшем в контейнер для сборки стадии.

* Установка [cookbook'а приложения](#cookbook-dapp).
  * Атрибуты.
    * Хеш атрибутов, совмещенных из [chef.attribute](chef_directives.html#chef-attributes) и [chef.\<стадия\>\_attribute](chef_directives.html#chef-<стадия>_attributes), устанавливается в normal-атрибуты, передаваемые через JSON-файл.
    * Файлы атрибутов не поддерживаются.
  * Файлы.
    * Содержимое директории files/\<стадия\>/common при наличии устанавливается в директорию files/default.
    * Содержимое директории files/\<стадия\>/\<рецепт\> устанавливается в директорию files/default.
      * Для каждого включенного рецепта при наличии соответствующей директории.
    * Недопустимо наличие одинаковых имен файлов в директории рецепта и common.
  * Шаблоны.
    * Содержимое директории templates/\<стадия\>/common при наличии устанавливается в директорию templates/default.
    * Содержимое директории templates/\<стадия\>/\<рецепт\> устанавливается в директорию templates/default.
      * Для каждого включенного рецепта при наличии соответствующей директории.
    * Недопустимо наличие одинаковых имен файлов в директории рецепта и common.
  * Рецепты.
    * Файл recipes/\<стадия\>/\<рецепт\>.rb устанавливается в recipes/\<рецепт\>.rb.
      * Для каждого включенного рецепта при его наличии.
    * При отсутствии рецептов генерируется пустой рецепт recipes/void.rb.
      * Отсутствие рецептов подразумевает одно из условий:
        * отсутствие включенных рецептов в конфигурации;
        * отсутствие файлов рецептов (recipes/\<стадия\>/\<рецепт\>.rb) для всех включенных в конфигурации рецептов.
      * Это позволяет активировать атрибуты, объявленные в данном cookbook'е.
* Установка [dimod](#dimod).
  * Атрибуты.
    * Файл attributes/\<стадия\>.rb устанавливается в attributes/\<стадия\>.rb.
    * Файл attributes/common.rb устанавливается в attributes/common.rb.
  * Файлы.
    * Содержимое директории files/common при наличии устанавливается в директорию files/default.
    * Содержимое директории files/\<стадия\> при наличии устанавливается в директорию files/default.
    * Недопустимо наличие одинаковых имен файлов в директории стадии и common.
  * Шаблоны.
    * Содержимое директории templates/common при наличии устанавливается в директорию templates/default.
    * Содержимое директории templates/\<стадия\> при наличии устанавливается в директорию templates/default.
    * Недопустимо наличие одинаковых имен файлов в директории стадии и common.
  * Рецепты.
    * Файл recipes/\<стадия\>.rb при наличии устанавливается в recipes/\<стадия\>.rb.
    * При отсутствии рецепта генерируется пустой рецепт recipes/void.rb.
      * Это позволяет активировать атрибуты, объявленные в данном cookbook'е, при отсутствии рецепта.
* Остальные cookbook'и устанавливаются без изменений, "как есть".

### Контрольная сумма cookbook'ов стадии
Контрольная сумма cookbook'ов стадии — это контрольная сумма всех [установленных файлов cookbook'ов](#установка-стадии-cookbook’а) для данной стадии.

### Дерево cookbooks
Дерево cookbooks — это результат выполнения berks vendor в chef приложении.



---
title: Диаграмма зависимостей chef-сборщика
sidebar: doc_sidebar
permalink: chef_dependencies.html
folder: definition
---

![Chef dependencies](https://docs.google.com/drawings/d/1zAAmxIqpfONBp9u3kZvd_KpCukvH_htvkd--OWtWt54/pub?w=1440&h=1080)