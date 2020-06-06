---
layout:     post
title:      Аутентификация через социальные сети с AngularJS и hello.js
date:       2014-10-13
summary:    Заметка о том, как реализовать аутентификацию на клиенте через социальные сети при помощи AngularJS и hello.js.
tags: AngularJS hello.js NodeJS oAuth
---

Существует ряд решений для задачи аутентификации через социальные сети. Например, можно использовать [PassportJS](http://passportjs.org/). PassportJS дает большую гибкость и кучу возможностей и тд, но мне показалось что все должно решаться гораздо проще и хотелось бы по-меньше использовать бэкенд. Забегая вперед, отмечу что без бэкенда вовсе, к сожалению, не получится обойтись.

И так, будем использовать ```hello.js``` для решения поставленной задачи.

[Hello.js](http://adodson.com/hello.js/) это клиентский JavaScript SDK для аутентификации через сервисы, использующие ```oAuth2``` и ```oAuth```, например twitter, github, facebook и тд.

Мы будем использовать только twitter, поэтому сначала нужно создать приложение, которое будет получать доступ к данным пользователя. Как это сделать можно узнать [тут](http://iag.me/socialmedia/how-to-create-a-twitter-app-in-8-easy-steps/).

Для провайдеров, которые используют oAuth или oAuth2 с Explicit Grant, в процессе аутентификации должен присутствовать ```secret key```, а тк библиотека клиентская, не очень хорошо хранить ключ на клиенте и передавать его в открытом виде. Об этой проблеме уже позаботились за нас и ```hello.js``` дает возможность использовать промежуточный сервис через прокси. Этот сервис будет получать ```secret key``` и реализовывать ```handshake``` механизм для получения ```access_token```. Подробнее [тут](http://adodson.com/hello.js/#oauth-proxy).

_Кстати сказать, можно не писать свой сервис для этого, а использовать [прокси](https://auth-server.herokuapp.com/), который автор hello.js предоставляет для этих целей. Но это означает, что нужно будет засветить свой secret key третьему лицу, плюс, не очень хорошо полагаться на сторонний сервис (который, как минимум, может быть недоступен), разве что наша цель просто "поиграться" с библиотекой и забыть.._


И так..

_Дальше можно не читать, а просто забрать исходники приложения [тут](https://github.com/dmaslov/auth-example)_

На клиенте у нас будет [AngularJS](https://angularjs.org/) и hello.js, на бэкенде [Express](http://expressjs.com/) + [oauth-shim](https://github.com/MrSwitch/node-oauth-shim). Соберем все [gulp](http://gulpjs.com/)'ом.


Так будет выглядеть структура нашего проекта:

![project_structure](http://i.imgur.com/LrY1Vvn.png)


package.json для установки зависимостей для серверной части и консольных скриптов:

{% gist dmaslov/3d16aa90284ead1d2db6 %}


bower.json для установки зависимостей для клиентской части:

{% gist dmaslov/8ec0e672eeffd23c109a %}


gulpfile.js будет просто собирать js файлы в один, минифицировать и копировать в /public/js:

{% gist dmaslov/785ad3c99919c9a93412 %}


Сразу рассмотрим код серверной части, он не так интересен. Финальный вариант:

{% gist dmaslov/88b6f6c7d5c4e176241a %}


Из интересного тут только ```oauth-shim```, остальное нужно чтобы просто работало приложение

Подключаем
{% highlight javascript linenos %}
oauthshim = require('oauth-shim')
{% endhighlight %}

Инициализируем (тут мы вписываем clientId и clientSecret из настроек нашего твиттер приложения)

{% highlight javascript linenos %}
oauthshim.init({
  'clientID' : 'clientSecret'
});
{% endhighlight %}


Вызываем (Все запросы от клиента к прокси будут вызывать oauthshim)

{% highlight javascript linenos %}
app
.get('/proxy', oauthshim.request)
{% endhighlight %}


Клиентская часть нашего приложения будет состоять из одной директивы и фабрики, в которой будет находится основная логика. Сразу привожу финальный вид ```app.js```, ниже разберем код подробней:

{% gist dmaslov/2b0bc0ad89af3f12502a %}



Шаблон директивы делится на две части. Если пользователь не авторизирован, показываем ему кнопку аутентификации, иначе - кнопку выхода. Шаблон меняется в зависимости от значения переменной ```isAuthorized``` в скоупе директивы:

{% gist dmaslov/a772e7d43f623e1109e4 %}



index.html нашего приложения:

{% gist dmaslov/a4de767cd8bdbf659b1d %}



### Первый запуск:

в консоле переходим в корень нашего приложения и выполняем команды:


Установка серверных зависимостей

{% highlight bash %}
npm install
{% endhighlight %}


Установка клиентских зависимостей

{% highlight bash %}
bower install
{% endhighlight %}


Сборка клиента

{% highlight bash %}
gulp
{% endhighlight %}


Все три утилиты должны быть предварительно установленны глобально в системе.

Дальше запустим сервер и посмотрим что получилось:

{% highlight bash %}
npm start
{% endhighlight %}


Переходим в браузере по адресу ```localhost:8080``` и, если все предыдущие действия прошли успешно, видим кнопку авторизации:

![App](http://i.imgur.com/BauGdm0.png)



Разберем код ```app.js``` и workflow нашего приложение.


### Сценарий первый (пользователь не авторизован):

Пользователь кликает по кнопке, срабатывает метод ```authCtrl.login('twitter')```, который вызывает функцию ```login``` фабрики, отвечающую за аутентификацию и передает в нее имя провайдера (в нашем случае это twitter)

{% highlight javascript linenos %}
authCtrl.login = function(provider){
  AuthFactory.login(provider);
}
{% endhighlight %}


Далее эта функция инициализирует библиотеку ```hello.js``` и вызывает метод ответственный за аутентификацию. Когда аутентификация завершится, вернем ```response``` обратно в контроллер:

{% highlight javascript linenos %}
function login(provider){
  init();
  hello(provider).login().then(function(response) {
    $rootScope.$broadcast('auth', response);
  });
}
{% endhighlight %}


В функции ```init``` указываем адрес прокси и адрес переадресации после аутентификации:

{% highlight javascript linenos %}
function init(){
  if(!isInited){
    hello.init(
    {twitter : 'twitter_client_id_here'},
    {
      redirect_uri: '/redirect',
      oauth_proxy: '/proxy'
    });
    isInited = !isInited;
  }
}
{% endhighlight %}


Далее прокси получает запрос и обрабатывает его

{% highlight javascript linenos %}
app
.get('/proxy', oauthshim.request)
{% endhighlight %}


Происходит ```handshake``` и мы получаем ```access_token```, после чего происходит редирект

{% highlight javascript linenos %}
.get('/redirect', function(req, res){
  res.sendFile('close.html', {root: app.get('views')});
})
{% endhighlight %}


В файле ```close.html``` просто подключается наш js файл. ```Hello.js``` сам закроет окно после аутентификации

{% gist dmaslov/d0671afc58eb9780ee8d %}


После успешной аутентификации, метод ```hello(provider).login()``` вернет нам ответ с информацией об аутентификации:

![response](http://i.imgur.com/akwNoqE.png)


Далее мы получаем ```response``` в методе ```link``` директивы, если получен ```access_token```, то переключаем переменную ```isAuthorized``` в ```true``` и узнаем имя пользователя, для отображения в шаблоне:

{% highlight javascript linenos %}
scope.$on('auth', function(event, response) {
  if(response.authResponse.access_token){
    scope.authCtrl.getUserDetails();
    scope.$apply(function(){
      scope.isAuthorized = true;
    });
  }
});
{% endhighlight %}

{% highlight javascript linenos %}
vm.getUserDetails = function() {
  var authData = AuthFactory.getAuthResponse('twitter');
  if(authData){
    vm.userName = authData.screen_name;
    $scope.isAuthorized = true;
  }
};
{% endhighlight %}


Поскольку теперь ```isAuthorized === true``` в шаблоне будет отображена кнопка выхода, вместо кнопки аутентификации:

![logged_in](http://i.imgur.com/tEiXeTQ.png)


### Сценарий второй (пользователь авторизован):

Пользователь заходит на страницу, в методе ```link``` мы проверяем есть ли данные о пользователе, если есть - получаем имя пользователя, переключаем переменную ```isAuthorized``` в ```true```. Поскольку ```isAuthorized === true``` в шаблоне будет отображена кнопка выхода, вместо кнопки аутентификации.

{% highlight javascript linenos %}
function link(scope, element, attrs) {
  scope.authCtrl.getUserDetails();
  ...
}
{% endhighlight %}

{% highlight javascript linenos %}
vm.getUserDetails = function() {
  var authData = AuthFactory.getAuthResponse('twitter');
  if(authData){
    vm.userName = authData.screen_name;
    $scope.isAuthorized = true;
  }
};
{% endhighlight %}


Пользователь кликает на кнопку ```logout```, вызывается соответствующий метод контроллера директивы, который вызывает метод фабрики, ответственный за logout. ```hello.js``` удаляет данные о пользователе и мы переключаем переменную ```isAuthorized``` в ```false```:

{% highlight javascript linenos %}
vm.logout = function(provider) {
  AuthFactory.logout(provider);
  $scope.isAuthorized = false;
};
{% endhighlight %}


Поскольку ```isAuthorized === false``` в шаблоне будет отображена кнопка входа, вместо кнопки выхода.



Кстати сказать, ```hello.js``` использует ```LocalStorage``` для хранения данных.


Исходники этого приложения можно забрать [тут](https://github.com/dmaslov/auth-example).
