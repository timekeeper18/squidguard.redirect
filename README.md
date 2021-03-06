## SQUIDGUARD: НАСТРАИВАЕМ ПЕРЕНАПРАВЛЕНИЕ ЗАБЛОКИРОВАННОГО РЕСУРСА НА СТРАНИЦУ БЛОКИРОВКИ
После установки связки Squid+squidGuard для URL-фильтрации на основе блэклистов перед вами встали несколько вопросов:

* Как обрабатывать запросы к заблокированным блэклистами доменам?
* Если перенаправлять их на какой-либо домен, то как предоставить пользователям возможность обратной связи (например, при помощи удобной отсылки запроса на исключение домена из блэклиста)?

Мы начали эксперименты. Создали страницу для обработки заблокированных URL-ей, на которую решили перенаправлять запросы к доменам из блэклиста. Этой странице в качестве параметра мы передаем запрашиваемый домен, а она предоставляем пользователю ответ об ограничении доступа и возможность не согласиться с решением системы.

Для редиректа на эту страничку мы создали в файле конфигурации следующие правила контроля доступа (ACL):
>#ACL RULES:<br>
>acl {<br>
> default {<br>
   >      pass white_dest !digincore_bl !in-addr all<br>
   >      redirect  http://blocked.mysite.com/?url=%u<br>
   >   }
>}

Т.е. при помощи переменной %u (подробнее можно почитать на сайте разработчиков squidGuard) мы передаем нашей странице блокировки запрашиваемый пользователем URL. Но оказалось, что при прямом редиректе,  в адресной строке браузера по прежнему будет находится изначально запрашиваемый адрес сайта, а не blocked.mysite.com, что приводит к некорректному отображению картинок и непредсказуемой работе функционала страницы.

Встала задача как перенаправить не просто контент страницы браузеру, но и адрес нового сайта. В ходе исследований мы пришли к такому решению. 

Создаем файл /var/www/blocked.php со следующим содержанием:

 > header("Location:  http://blocked.mysite.com/?url=".$_GET['u']);

Что проделывает данный скрипт: он получает введенный URL ($_GET['u']) и подставляет его в качестве параметра в страницу финального назначения (http://blocked.mysite.com/?url=), куда и происходит редирект.

Далее, изменяем настройки ACL в конфиге squidGuard, а именно, убираем прямой редирект на сайт обработки URL и заменяем его редиректом на созданный нами файл blocked.php:
>#ACL RULES:<br>
>acl {<br>
>   default {<br>
>      pass white_dest !digincore_bl !in-addr all<br>
>      redirect http://localhost/blocked.php?u=%u<br>
>   }<br>
>}<br>

Таким образом, мы используем для перенаправления промежуточный файл, при этом браузеру предоставляется новый адрес и отображение финальной страницы становится корректным. Например, при попытке зайти на ресурс sex.com, в адресной строке браузера появится следующая ссылка http://blocked.mysite.com/?url=http://sex.com/

Еще одно изменение, которое поможет этой схеме работать стабильней - необходимо ввести адрес домена обработки (blocked.mysite.com) в белый список (white list) и расположить его в домашней директории баз squidGuard (в файле конфигурации она хранится в переменной dbhome). Создать группу в разделе “DESTINATION CLASSES” (у нас она называется white_dest) и разрешить доступ ко всем доменам из этой группы. В разделе pass указано, что пропускать white_dest.
