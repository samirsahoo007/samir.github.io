---
layout:     post
title:      autoload при помощи composer
date:       2013-10-28
summary:    Заметка о том, как реализовать autoload в PHP при помощи composer.
tags: composer php
---

Composer это менеджер пакетов для PHP. Подробно о работе с ним можно почитать [тут](http://getcomposer.org), [тут](http://habrahabr.ru/post/145946/) или [тут](http://daylerees.com/composer-primer).

С помощью composer мы можем не только собрать проект из различных библиотек, но и просто создать autoloader для своих классов.


Нам понадобится установленный ```composer```, посмотреть как установить можно [тут](http://getcomposer.org/download).
Мы можем описать правила по которым autoloader будет искать файлы классов. Рекомендуется использовать формат ```psr-0``` так как этот формат обладает большей [гибкостью](http://getcomposer.org/doc/04-schema.md#autoload).

Структура проекта следующая

![project_structure](http://i.imgur.com/pakqqbX.png)

Создаем в корне тестового проекта файл ```composer.json``` и прописываем в нем требуемые для автоподключения классы:

в формате ```psr-0```

{% gist dmaslov/4efa6dbf772ea69079ca %}

в формате ```classmap```

{% gist dmaslov/f0af8832de6d84a2786b %}

в формате ```files```

{% gist dmaslov/1660f19ecee7973a4113 %}


```includes/``` во всех листингах - это директория в которой лежат файлы с нашими классами.
 
Переходим в корень нашего проекта выполняем команду ```composer install```

![composer_log](http://i.imgur.com/7cOOkrz.png)

Теперь в корне проекта находится директория vendor с нашим autoloader. Ниже приведен код файлов, отвечающих за подключение наших классов в зависимости от формата по которому мы создавали ```composer.json```.

в формате ```psr-0```

{% gist dmaslov/2a90147e95f2d72634dd %}

в формате ```classmap```. Поскольку в ```composer.json``` было прописано подключить всю директорию, без разбора, подключились также классы, которые мы не добавляли в ```composer.json``` в других форматах.

{% gist dmaslov/33e36e02dc255e8cbd1c %}

в формате ```files```

{% gist dmaslov/de7cf6bbad8925806dd5 %}


Все что остается сделать, это подключить файл ```autoload.php```, который находится в директории ```vendor```, в нашем проекте

{% gist dmaslov/e4c286d93c69786b7079 %}


При добавлении новых классов, прописываем их в ```composer.json```, выполняем команду ```composer dump-autoload``` и они попадают в наш автозагрузчик. Все!
