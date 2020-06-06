---
layout:     post
title:      macos+php-fpm+php5_module
date:       2013-10-21
summary:    Заметка о том, как собрать php 5.x с возможностью работы в FastCGI режиме и режиме модуля к apache на примере php 5.4 в mac os x.
tags: homebrew macos php php-fpm
---


#### Дано:

Mac os x 10.8 с входящем в сборку по-умолчанию apache и php 5.3.15.

#### Задача:

Собрать php 5.x с возможностью работы в FastCGI режиме и режиме модуля к apache на примере php 5.4 в mac os x.


Подробно об установке php 5.4 в mac os x через ```homebrew``` написано [здесь](https://github.com/josegonzalez/homebrew-php).

Проверяем варианты установки php54 командой ```brew options php54```.

![screenshot_1](http://i.imgur.com/7jItDKd.png)

Стандартная формула установки предлагает сконфигурировать php с ключом ```--with-fpm```. Этот вариант не подходит, так как исключает возможность запуска в виде модуля к apache (implies ```--without-apache```).

Подправим формулу установки (Если в будущем мы решим обновить пакет php54, то все изменения, описанные тут, надо будет проделать снова).

Открываем в редакторе файл ```/usr/local/Library/Formula/php54.rb``` и дописываем ключ ```--enable-fpm```.

{% gist dmaslov/4b18215197dfb74b9fa1 %}

Сохраняем и устанавливаем/переустанавливаем пакет php54 командой ```brew install/reinstall php54```.

Теперь наш php54 может работать как модуль к apache, так и в режиме fastCGI, для проверки открываем httpd.conf находим подключение libphp5.so
```LoadModule php5_module libexec/apache2/libphp5.so```

и меняем его на

```#LoadModule php5_module libexec/apache2/libphp5.so```

```LoadModule php5_module /usr/local/Cellar/php54/5.4.14/libexec/apache2/libphp5.so```

Перезапускаем apache ```sudo apachectl restart```
Видим что теперь php может работать в fastCGI режиме ```--enable-fpm```.

![screenshot_2](http://i.imgur.com/FNUNPwY.png)

У нас теперь два php-fpm. Для php 5.3.15 лежит тут ```/usr/sbin/php-fpm```, для php 5.4.14 лежит тут ```/usr/local/Cellar/php54/5.4.14/sbin/php-fpm```.

Конфиги лежат соответственно тут ```/private/etc/php-fpm.conf``` и тут ```/usr/local/etc/php/5.4/php-fpm.conf```.

Для переключения между версиями php я использую следующие действия:

#### Для nginx:

Делаем alias в bash_profile для php 5.4.14 php-fpm.

```alias php-fpm54="/usr/local/Cellar/php54/5.4.14/sbin/php-fpm"```

Перезапускаем консоль и теперь запускаем или php-fpm, или php-fpm54. Запускаем nginx. Готово.

![screenshot_3](http://i.imgur.com/DTnHOsP.png)

![screenshot_4](http://i.imgur.com/6jjlEY3.png)

#### Для apache:

открываем httpd.conf комментируем нужную строчку

```#LoadModule php5_module libexec/apache2/libphp5.so```

```LoadModule php5_module /usr/local/Cellar/php54/5.4.14/libexec/apache2/libphp5.so```

перезапускаем apache. Готово.

![screenshot_6](http://i.imgur.com/41CoXst.png)

#### Для CLI:

редактируем bash_profile, добавляем строчку

```PATH="$(brew --prefix josegonzalez/php/php54)/bin:$PATH"```

перезапускаем консоль. Готово.

![screenshot_5](http://i.imgur.com/1PAGr2Z.png)

Все..
