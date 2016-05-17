# Анализ группы ВК: часть 1, данные
Юрий Исаков  
Апрель 2016  

***

Отчет в трех частях о работе с данными группы ВКонтакте с помощью языка R. [Первая часть](part1.md) описывает получение и обработку данных, [вторая](part2.md) -- их визуализацию, а [третья часть](part3.md) посвящена моделированию. В качестве анализируемой группы выступает [беговое сообщество Воронежа](https://vk.com/runningvrn).

***


В первой части используем API ВКонтакте и загрузим данные об участниках сообщества, а также информацию об активностях пользователей на стене группы.

### 1. Начало

Для работы с API ВКонтакте мы будем использовать OAuth-авторизацию. Приведенный код использовался в RStudio на OS X El Capitan. Для других операционных систем, возможно, потребуется задействовать другие библиотеки.


```r
Sys.setlocale('LC_ALL','utf-8') # если нет проблем с кодировкой, отключите эту строчку
```

```
## [1] "C/utf-8/C/C/C/C"
```

```r
library(RCurl)
library(httr)
library(RJSONIO)
library(lubridate)
library(dplyr)
```

### 2. Настройка соединения

Прежде всего надо [зарегистрировать](https://vk.com/dev) свое приложение как Standalone. После регистрации приложения заходим в раздел **Настройки** узнаем свои `client_id` и `client_secret`, сохраняем их соответствующие переменные вместе с именем приложения `app_name` в файл *secret_keys.R*. В поле `Адрес сайта` вводим адрес, который в консоли R выдает функция `oauth_callback()` и прописываем *localhost* в поле `Базовый домен`. Теперь можно проходить аутентификацию. Напишем функцию, которая будет возвращать строку с токеном:


```r
get_access_token <- function(){
    accessURL <- "https://oauth.vk.com/access_token"
    authURL <- "https://oauth.vk.com/authorize"
    vk <- oauth_endpoint(authorize = authURL,
                         access = accessURL)
    myapp <- oauth_app(app_name, client_id, client_secret)
    ig_oauth <- oauth2.0_token(vk, myapp,  
                               type = "application/x-www-form-urlencoded",
                               cache=FALSE)
    my_session <-  strsplit(toString(names(ig_oauth$credentials)), '"')
    access_token <- paste0('access_token=', my_session[[1]][4])
    
    access_token
}
```

Теперь для авторизации нам надо выполнить две команды. Параметр `access_token`, который нужно указывать в некоторых случаях будет находиться в одноименной переменной. После успешного прохождения аутентификации в браузере появится сообщение: *Authentication complete. Please close this page and return to R*. 


```r
source('secret_keys.R')
# файл выглядит как-то так,
# client_id <- "123456"
# client_secret <- "секретик"
# app_name <- "мое_приложение"
access_token <-get_access_token()
```

Теперь мы можем отправлять запросы используя [методы API](https://vk.com/dev/methods) и получать информацию в формате JSON. Например:


```r
fromJSON(getURL('https://api.vk.com/method/groups.getById?group_id=rommikh'))
```

```
## $response
## $response[[1]]
## $response[[1]]$gid
## [1] 108370559
## 
## $response[[1]]$name
## [1] "Роман Михайлов. Мысли и Загоны."
## 
## $response[[1]]$screen_name
## [1] "rommikh"
## 
## $response[[1]]$is_closed
## [1] 0
## 
## $response[[1]]$type
## [1] "page"
## 
## $response[[1]]$photo
## [1] "http://cs629512.vk.me/v629512597/28987/FZw1VCwd6G0.jpg"
## 
## $response[[1]]$photo_medium
## [1] "http://cs629512.vk.me/v629512597/28986/FflZLcIxhQQ.jpg"
## 
## $response[[1]]$photo_big
## [1] "http://cs629512.vk.me/v629512597/28985/jW40wpOChPY.jpg"
```


### 3. Загрузка списка участников группы

Ниже приведен код двух функций, которые используются для получения списка участников группы. Певрвая, `get_members()` имеет два аргумента: идентификатор группы и порядок сортировки (необязательный аргумент), который нужен в случае, если запрос адресуется группе, в которой у нас нет прав модератора (см. комментарий в коде). Также, заданы все возможные поля участников из которых будут выбраны нужные функцией `members2df()`. Эта функция возвращает датафрейм из переданного ей списка. Здесь используется перебор по всем элементам списка. Хотя этот метод не самый быстрый, он нам вполне подходит — количество пользователей невелико, и он наглядный, нет необходимости подбирать индексы и имена признаков в наших данных. Стоить заметить, что этот метод вернет только первую тысячу участников, т.к. у исследуемой группы на данный момент порядка 630, то пока он нам подходит. Ниже нам придется загружать и длинные списки (для получения всех постов на стене группы), а так же как преобразовывать данные для стран и городов.


```r
get_members <- function(group_domain, sort = 'sort=time_asc') {
    # формируем строку запроса
    # sort <- 'sort=id_asc' # если нет модераторских прав
    # у нас есть права, поэтому можем загружать в порядке вступления
    fields <- 'fields=sex,bdate,city,country,photo_50,photo_100,photo_200_orig,photo_200,photo_400_orig,photo_max,photo_max_orig,online,online_mobile,lists,domain,has_mobile,contacts,connections,site,education,universities,schools,can_post,can_see_all_posts,can_see_audio,can_write_private_message,status,last_seen,relation,relatives,counters'
    api <- paste0('https://api.vk.com/method/groups.getMembers?group_id=', group_domain)
    request <- paste(api, fields, sort, access_token, sep='&')
    # получаем данные в формате JSON
    members_list <- fromJSON(getURL(request))
    # преобразуем список в data.frame
    members <- members2df(members_list$response$users)

    members
}

members2df <- function(members){
    # создаем датафрейм, в который будем записывать данные
    df <- data.frame(uid = rep(0,length(members)))
    i <- 0
    for (member in members) {
        i <- i + 1
        df$uid[i] <- member$uid # id пользователя
        df$first_name[i] <- member$first_name # имя
        df$last_name[i] <- member$last_name # фамилия
        df$sex[i] <- member$sex # пол
        df$bdate[i] <- ifelse(is.null(member$bdate), NA, 
                              ifelse(nchar(member$bdate)<6,
                                    as.character(dmy(paste0(member$bdate,'.1904'))),
                                    as.character(dmy(member$bdate))))  # дата рождения
        df$city_id[i] <- ifelse(is.null(member$city), NA, member$city) # город
        df$country_id[i] <- ifelse(is.null(member$country), NA, member$country) # страна
        df$university[i] <- ifelse(is.null(member$university_name), NA,
                                   ifelse(member$university_name=='', NA, 
                                          member$university_name)) # ВУЗ
        df$deactivated[i] <- ifelse(is.null(member$deactivated), 'active', 
                                    member$deactivated) # живой ли аккаунт пользователя?
    }
    
        df
}
```

Итак, теперь мы можем получить данные о пользователях, которые сохраним в таблицу `members`. Также получим список уникальных стран и городов наших участников.

```r
group_domain <- 'running_vrn'
members <- get_members(group_domain)
members_countries <- unique(members$country_id)
members_cities <- unique(members$city_id)
```

### 4. Загрузка дополнительных данных

В ответ на запрос о городах и странах данные приходят в таком формате:


```r
fromJSON(getURL('https://api.vk.com/method/database.getCountriesById?country_ids=211,61'))
```

```
## $response
## $response[[1]]
## $response[[1]]$cid
## [1] 211
## 
## $response[[1]]$name
## [1] "Французская Полинезия"
## 
## 
## $response[[2]]
## $response[[2]]$cid
## [1] 61
## 
## $response[[2]]$name
## [1] "Гваделупа"
```

В этом случае мы можем пребразовывать список списков в датафрейм используя векторные выражения и следующую хитрую функцию. В результирующей переменной придется переименовать колонки.


```r
lists2df = function(ll){
    as.data.frame(do.call(rbind, lapply(lapply(ll, unlist), "[",
                                        unique(unlist(c(sapply(ll,names)))))),
                  stringsAsFactors=FALSE)
}
```

Теперь мы можем записать страны и города участников в переменные `countries` и `cities` соответственно, передав функции `lists2df()` результат запроса в формате JSON. Перед тем, как собрать все данные окончательно, мы переименуем столбцы этих двух переменных таким образом, чтобы названия были как в переменной `members`.


```r
# страны
request_countries <- paste0('https://api.vk.com/method/database.getCountriesById?country_ids=', 
                            paste(members_countries, collapse=","))
countries <- lists2df(fromJSON(getURL(request_countries))$response)
# города
request_cities <- paste0('https://api.vk.com/method/database.getCitiesById?city_ids=', 
                         paste(members_cities, collapse=","))
cities <- lists2df(fromJSON(getURL(request_cities))$response)
```


### 5. Загрузка постов со стены группы

Воспользуемся методом `wall.getById` для чтения всех постов в группе (*заметим, этот способ не оптимальный, см. дальше*). Методу надо сообщать идентификаторы постов, которые имеют вид `-<group_id>_<post_id>`. Т.к. нам неизвестны списки постов, то придется перебирать все подряд. Сразу получить все посты за один запрос не удастся, поэтому будем перебирать их пачками по 100. В случае, если поста с данным идентификатором не существует, ничего возвращено не будет. Все посты в группе начинаются с 1. Вычислить последний можно прочитав 2  последних поста и сравнив их идентификаторы (последний пост может оказаться старым и закрепленным, поэтому для уверенности надо прочитать и последний) — для этого можно воспользоваться методом `wall.get` (здесь у них индексы 2 и 3):


```r
two_last_posts <- fromJSON(getURL('https://api.vk.com/method/wall.get?domain=runningvrn&count=2'))$response
id_max <- max(two_last_posts[[2]]$id, two_last_posts[[3]]$id)
```

В функции `get_wall_posts()` мы формируем список идентификаторов постов, которые потом пытаемся получить. Посты читаются пачками, размер которой определен как необязательный параметр `id_step`. После получения списка постов они приводятся в требуемый вид функцией `wall2df()`.


```r
# пауза при между последовательными запросами
sleep_time <- .34
# сохраняем записи c id_min по id_max в датафрейм
get_wall_posts <- function(id_min, id_max, id_step=100){
    # устанавливаем основные параметры вызова
    extended <- paste0('extended=', 0)
    copy_depth <- paste0('copy_history_depth=', 1)
    # загружаем посты пачками
    id_lo=id_min;id_hi=id_min+id_step-1
    cat(id_min,'-',id_max,': ') # вывод текущей позиции, чтобы не грустить
    while (id_lo < id_max) {
        cat(min(id_hi, id_max), '. ')# вывод текущей позиции, чтобы не грустить
        posts_range <- id_lo:id_hi # диапазон в текущей пачке
        posts <- paste0('posts=', paste0('-', group_id, '_', posts_range, 
                                        collapse=','))
        # используем версию 4.9
        # можно без access_token (изменится поле whodidthis)
        # request <- paste('https://api.vk.com/method/wall.getById?v=4.9',
        #                  posts, extended, copy_depth, sep='&')
        request <- paste('https://api.vk.com/method/wall.getById?v=4.9',
                         posts, extended, copy_depth, access_token, sep='&')
        posts_list <- fromJSON(getURL(request))
        # если пачка первая, то создаем датафрейм
        if (id_lo == id_min) 
            df <- wall2df(posts_list$response)
        # а если нет, то дополняем следующей пачкой
        else 
            df <- rbind(df, wall2df(posts_list$response))
        # пауза, чтобы запросы не были слишком частыми
        if (id_hi < id_max) Sys.sleep(sleep_time)
        # индексы для новой пачки
        id_lo <- id_lo+id_step
        id_hi <- id_hi+id_step
    }
    
    df
}

# сохраняем посты из wall в датафрейм
wall2df <- function(wall){
    # создаем data.frame в который будем записывать данные
    df <- data.frame(uid=rep(0, length(wall)))
    i <- 0
    # перебираем все посты
    for (wall_post in wall){
        i <- i + 1
        df$uid[i] <- wall_post$id # id поста
        df$author[i] <- wall_post$from_id # автор поста
        df$whodidthis[i] <- ifelse(is.null(wall_post$created_by),
                                   ifelse(is.null(wall_post$signer_id),
                                          NA, wall_post$signer_id),
                                   wall_post$created_by) # автор репоста, если указан
        df$type[i] <- wall_post$post_type # пост/репост
        df$comments[i] <- wall_post$comments[["count"]] # кол-во комментариев
        df$likes[i] <- wall_post$likes[["count"]] # кол-во лайков
        df$reposts[i] <- wall_post$reposts[["count"]] # кол-во репостов
        df$date[i] <- wall_post$date # дата поста
        df$text[i] <- wall_post$text # текст поста
    }
    # преобразуем дату в нужный формат
    df$date <- as.POSIXct(df$date, origin="1970-01-01", 
                          tz='Europe/Moscow')
    
    df
}
```

Итак, сохраним все посты группы:


```r
group_id <- 89497660 # id нашей группы
id_min <- 1
posts <- get_wall_posts(id_min, id_max)
```

Опять же заметим, что этот способ не подходит для сообществ с большой историей. Чтение таким методом займет какое-то время из-за перерывов между чтением пачек. Это не критично в нашем случае, потому как постов довольно мало (меньше 500 на момент написания). В целом, наиболее логичным же представляется способ комбинацией типов запросов. Например, в нашем случае на 100 идентификаторов возвращается порядка 20 реально существующих постов. Используя же метод `wall.get` можно читать по 100 штук, однако здесь возможны трудности -- например, во время чтения может быть создан новый пост или откреплен верхний, а чтение этим методом происходит сверху вниз, таким образом, пачки "поедут". Но благодаря комбинации методов API можно составить список идентификаторов всех реально существующих постов, в этом случае можно прилично уменьшить время чтения.

### 6. Лайки, комментарии и снова лайки

Мы хотим узнать, кто ставит отметки "мне нравится", комментирует посты. Также, соберем идентификаторы тех пользователей, кто ставит лайки комментариям. Функция `get_likers_commenters()` возвращает список, где для каждого поста на стене указаны поставившие лайк к посту, прокомментировавшие его и поставившие лайк к комментариям (в полях `likers`, `commenters`, `comments_likers` соответственно). Функция возвращает первые сто лайков и комментариев, что не ограничивает, у нас и столько нет.


```r
get_likers_commenters <- function(posts){
    posts_likers_commenters <- list()
    cat('1-', dim(posts)[1], ': ', sep='')
    for (i in 1:dim(posts)[1]){
        # получаем пользователей, лайкнувших пост
        request_likers <- paste0('https://api.vk.com/method/likes.getList?owner_id=-',
                                 group_id, '&type=post&item_id=', posts$uid[i])
        likers <- fromJSON(getURL(request_likers))$response$users
        # получаем пользователей, прокомментировавших пост
        request_comments <- paste0('https://api.vk.com/method/wall.getComments?v=5.50&owner_id=-',
                                   group_id, '&post_id=', posts$uid[i])
        comments <- fromJSON(getURL(request_comments))
        # список для комментаторов
        commenters <- c()
        # список идентификаторов постов
        comments_ids <- c()
        # список для поставивших лайки комментарию 
        comments_likers <- c()
        # прокомментировал ли кто-то пост?
        if (comments$response$count){
            commenters <- sapply(comments$response$items, 
                                 function(comment) comment$from_id)
            comments_ids <- sapply(comments$response$items, 
                                   function(comment) comment$id)
            # теперь пройдемся по всем комментариям, чтобы собрать лайкеров
            for (comment_id in comments_ids) {
                request_comments_likers <- paste0(
                    'https://api.vk.com/method/likes.getList?owner_id=-',
                    group_id, '&type=comment&item_id=', 
                    comment_id)
                comments_likers = c(comments_likers, 
                                    unlist(fromJSON(getURL(request_comments_likers))$response$users))
            }
        }
        # заполняем поля идентификаторами пользователей
        posts_likers_commenters[[i]] <- list(likers = likers,
                                             commenters = commenters,
                                             comments_likers = comments_likers)
        # скрашиваем томительное ожидание
        if( i %% 25 == 0) cat(i, ' . ')
        # на моем маке проблема с SSLRead при частых запросах ;(
        if( i %% 200 == 0) Sys.sleep(10)
    }
    posts_likers_commenters
}

posts_likers_commenters <- get_likers_commenters(posts)
```

Получив последий список с данными об активности пользователей на стене можно собирать все данные вместе 

### 7. Объединение данных

На последнем шаге объединим наши данные в таблицы `members` и `posts`, которые будем использовать для построения визуализации и моделирования.

Подготовим таблицы с городами и странами — для этого надо переименовать столбцы по образу таблицы `members`. Затем преобразуем идентификаторы из строк в числа и заполним пропуски для пользователей, не указавших свои географические данные. Наконец, введем переменную возраст. При создании, пользователям, не указавшим год рождения он устанавливался как 1904. Поэтому после вычисления возраста отсеем пользоватей с возрастом больше 100 лет.


```r
# переименуем столбцы
countries <- rename(countries, country_id=cid, country = name)
cities <- rename(cities, city_id=cid, city = name)
# преобразуем значения идентификаторов в численный тип
countries$country_id <- as.integer(countries$country_id)
cities$city_id <- as.integer(cities$city_id)
# подставим названия городов и стран
members <- left_join(members, cities, by = 'city_id')
members <- left_join(members, countries, by = 'country_id')
# заполним пропуски
members$country[is.na(members$country)] <- 'не указана'
members$city[is.na(members$city)] <- 'не указан'
# введем колонку с возрастом пользователей и уберем значения > 100 лет
members$age <- floor(as.numeric(difftime(now(), members$bdate, units = 'days'))/365.25)
members$age[members$age > 100] <- NA
```

Перенесем всю данные из переменной `posts_likers_commenters` в таблицу `posts`. При этом каждая ячейка в новых столбцах будет списком.


```r
# добавим информацию об активности к каждому посту
posts$likers <- sapply(posts_likers_commenters, function(plc) plc$likers)
posts$commenters <- sapply(posts_likers_commenters, function(plc) plc$commenters)
posts$comments_likers <- sapply(posts_likers_commenters, function(plc) plc$comments_likers)
```

Таким образом, вся нужная информация перенесена в таблицы `members` и `posts` и готова для использования. Сохраним ее в один файл для последующей работы.


```r
save(list = c('posts', 'members'), file = "runningvrn.RData", envir = .GlobalEnv)
```

### 8. Заключение

Итак, мы получили все необходимые исходные данные для анализа. Для справедливости стоит заметить, что для групп с большими количеством участников и активностью методы надо доработать. Хороший способ представляют собственные методы на VKScript.

Посмотрим заголовки полученных таблиц:


```r
load("runningvrn.RData")
# география участников сообщества
head(members[,-3], 3)
```

```
##         uid first_name sex      bdate city_id country_id university
## 1  14557170     Сергей   2 1991-02-11       0          1        ВГУ
## 2 101040354     Сергей   2 1904-02-24      42          1        ВГУ
## 3  93517209    Алексей   2 1994-06-03      42          1      ВГИФК
##   deactivated      city country age
## 1      active не указан  Россия  25
## 2      active   Воронеж  Россия  NA
## 3      active   Воронеж  Россия  21
```

```r
head(posts, 3)
```

```
##   uid    author whodidthis type comments likes reposts                date
## 1   2 -89497660         NA post        0    24       2 2015-03-12 23:50:09
## 2   3 -89497660         NA post       12    17       8 2015-03-13 11:04:37
## 3  17 -89497660   14557170 post       13    19       4 2015-03-14 10:52:10
##                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       text
## 1                                                                                                                                                                                                                               Друзья!<br>Мы создаем "Клуб любителей бега в городе Воронеж".<br>Каковы его цели:<br>1. Объединить людей, любящих бегать или желающих научиться.<br>2. Создать площадку для общения и обмена опытом и ,конечно, совместных тренировок.<br>3. Предоставить возможность заниматься с тренером и получить необходимые знания.
## 2                                                                                                                                                                                                                                                                                                                                                                  Друзья!<br>Завтра первый сбор и тренировка!<br>Встреча в 8.00 утра в СК "Олимпик" 14.03 суббота! <br>Сбор за шлагбаумом у красного здания с надписью "Subway".<br>Для связи 89805562423
## 3 Ребят, спасибо всем, кто сегодня пришел!Сбор был в 8, в 8.05 был старт,так что те, кто опоздали - в следующий раз не опаздывайте!)Вышло три группы: ребята пробежавшие 13 км, 6 км, и девочки,которые тоже что то, но пробежали!В связи с этим:<br>1. В следующий раз мы сделаем разделение на несколько групп!\U0001f3c3\U0001f3c3\U0001f3c3<br>2.Время и количество сборов тоже обсудим в группе,я думаю, 2-3 пробежки в неделю будет.<br>Девочки, которые спрашивали, как правильно бежать, дышать,одеваться и т.д. - все это будем постить в группе!
##                                                                                                                                                                                                                                                 likers
## 1 312499475, 2383410, 27426538, 116583986, 43224637, 16038695, 2375685, 31297393, 53200996, 132013462, 4906974, 13325384, 14557170, 23071780, 13148927, 41000815, 327715051, 215613672, 169312826, 21164907, 193199838, 22733010, 295507857, 101040354
## 2                                                                      31297393, 16452689, 37612440, 32838148, 14557170, 170847804, 28653573, 14867663, 193199838, 54937370, 59069686, 89453366, 198508425, 112710793, 195566974, 191661180, 101040354
## 3                                                     61756904, 49111265, 77397282, 35009292, 31297393, 13235227, 19073490, 116583986, 293992804, 7019172, 36217267, 18039690, 12921515, 16215566, 198508425, 101040354, 124076145, 28653573, 14557170
##                                                                                                commenters
## 1                                                                                                    NULL
## 2 231369078, 16215566, 231369078, 16215566, 101040354, 16215566, 101040354, 193199838, 43021214, 14557170
## 3         12795268, 14557170, 59069686, 14557170, 7846755, 14557170, 7846755, 14557170, 7846755, 14557170
##                          comments_likers
## 1                                   NULL
## 2                                   NULL
## 3 4041680, 14557170, 16215566, 198508425
```



























