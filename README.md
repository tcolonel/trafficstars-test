# Тестовое задание для trafficstars.com

Язык написания: PHP.

## Задание #1

##### Исходные данные:

Есть логин форма с двумя полями логин, пароль и кнопкой submit.
Нужно написать 2 скрипта для ее обработки. HTML составлять не нужно.
База данных MySQL представлена таблицей users с полями id, login, password.

1. Написать скрипт для обработки логин формы, который можно взломать SQL инъекцией.

2. Написать скрипт для обработки логин формы, который защищен от подобного взлома.

Скрипт должен возвращать значение true/false - авторизован пользователь или нет. Можно просто сохранить значение в какой-нибудь переменной, этого будет достаточно.

Код сопроводить заметкой как можно взломать первый скрипт.
Что помогает избежать взлома во втором скрипте.


##### Выполнение задания #1:

*Скрипт #1:*

    <?
    $login = (isset($_POST['login']) ? $_POST['login'] : '');
    $passwd = (isset($_POST['password']) ? $_POST['password'] : '');
    
    if(!empty($login) && !empty($passwd))
    {
      $_SESSION['AUTH_USER'] = authUser($login, $passwd);
    }
    else
    {
      ...
    }
    
    function authUser($login, $passwd)
    {
      $mysqli = new mysqli("localhost", "user", "password", "database");
      if($mysqli->connect_errno)
      {
        echo "Не удалось подключиться к MySQL: (" . $mysqli->connect_errno . ") " . $mysqli->connect_error;
      }
      
      $query = "select id from users where login = '$login' and password = '$passwd'";
      if($result = $mysqli->query($query))
      {
        /*
        while($row = $result->fetch_assoc())
        {
          ...
        }
        //*/
        $result->close();
        return true;
      } 
      else
      {
        return false;
      }
      
      $mysqli->close();
    }
    ?>

Данный скрипт уязвим к SQL-injection. Например, можно в $login / $passwd передать такой текст:

    'union select '1', concat(login||'-'||password) as name from users;--
таким образом мы изменим запрос и получим список всех юзеров с их паролями. В данной ситуации поле деятельности не ограничено.
 
 
 
*Скрипт #2:*

    <?
    $uLogin = (isset($_POST['login']) ? $_POST['login'] : '');
    $uPasswd = (isset($_POST['password']) ? $_POST['password'] : '');
    
    1. if(!empty($uLogin) && !empty($uPasswd))
    2. if(!empty($uLogin) && is_string($uLogin) &&  !empty($uPasswd) && is_string($uPasswd))
    {
      $_SESSION['AUTH_USER'] = authUser($uLogin, $uPasswd);
    }
    else
    {
      ...
    }
    
    function authUser($uLogin, $uPasswd)
    {
      $mysqli = new mysqli("localhost", "user", "password", "database");
      if($mysqli->connect_errno)
      {
        echo "Не удалось подключиться к MySQL: (" . $mysqli->connect_errno . ") " . $mysqli->connect_error;
      }
      
      $uLogin   = mysql_real_escape_string($uLogin);
      $uPasswd  = mysql_real_escape_string($uPasswd);
      $query    = "select id from users where login = ? and password = ?";
      
      if($result = $mysqli->prepare($query))
      { 
        $result->bind_param("ss", $uLogin, $uPasswd); 
        $result->execute(); 
        $result->bind_result($login, $passwd); 
        /*
        while($result->fetch())
        {
        }
        //*/
        $result->close();
        return true;
      }
      else
      {
        return false;
      }
      
      $mysqli->close();
    }
    ?>

В данном скрипте для избежания взлома мы использовали:

1. mysql_real_escape_string (_следует помнить, что применять её можно только после установления соединения с базой_)
2. "подготовленными выражениями" (prepared statements)
3. проверку входящих переменных на соотв. типы: _2. if(!empty($uLogin) && **is_string($uLogin)** &&  !empty($uPasswd) && **is_string($uPasswd)**)_

Так же следует обратить внимание на директивы **magic_quotes_gpc** и **magic_quotes_runtime**. Среди причин, по которым не стоит полагаться на "волшебные кавычки", есть ещё одна - к "волшебным кавычкам" относится еще **magic_quotes_sybase**. Мало того, что она вместо слеша добавляет кавычку - так она ещё и отменяет действие **magic_quotes_gpc**. Если каким-то чудом обе эти директивы имеют статус 'on', то последняя не сработает! То есть, полагаясь на "волшебные кавычки", мы в этом случае получим все прелести неправильно составленных запросов. Вообще, чисто теоретически, надо учитывать наличие этой директивы, поскольку она преподносит ещё и такой сюрприз, как... изменение поведения функций addslashes и stripslashes! Если magic_quotes_sybase = on, то эти функции начинают вместо слеша добавлять и удалять одинарную кавычку соответственно.


## Задание #2

Есть страница на сайте, которую мы хотим защитить от взлома. Для этого мы делаем фильтрацию по юзер агенту. Есть спец программы, которые делают перебор всех возможных способов взлома SQL инъекцией. Значение юзер агента у софта hacker.

Написать скрипт, который проверяет, что страница (ссылка) отдает пустой контент, когда мы заходим на нее с таким юзер агентом (hacker).

##### Выполнение задания #2:

Не совсем понял суть задачи, поэтому реализации будет 2.

*Реализация #1 (отдача пустой страницы при заходе на нее с user-agent: hacker):*

    <?
    function checkUserAgent()
    {
      $uAgent = $_SERVER['HTTP_USER_AGENT'];
      
      if(preg_match('/hacker/i', $uAgent))
        return true;
      else
        return false;
    }
    
    function checkUserAgentAnother()
    {
      $uAgent = $_SERVER['HTTP_USER_AGENT'];
      
      if(strlen(strstr($uAgent, "hacker")) > 0)
        return true;
      else
        return false;      
    }
    ?>

далее на странице проверяем:

    <?
    1. if(!checkUserAgent()) echo '';
    2. if(!checkUserAgentAnother()) echo '';
    else
    {
      // тело страницы
    }
    ?>

Можно так же использовать готовые решения, например:

1. Функцию **get_browser** (пытается определить возможности браузера пользователя производя поиск информации о браузере в файле browscap.ini). Для работы этой функции необходимо, чтобы в установке browscap в настройках php.ini был установлен корректный путь к файлу browscap.ini в вашей системе. Данная функция может сильно тормозить систему - при частом обращении к ней.
2. https://github.com/donatj/PhpUserAgent
3. https://github.com/ornicar/php-user-agent
4. https://github.com/jenssegers/Laravel-Agent


*Реализация #2 (проверка страницы с user-agent: hacker):*
    
    <?
    function getPageContent($url, $debug = false)
	  {
		  if($url === false) die('Задайте URL, он не может быть пустым.');
		
    	$ch = curl_init();
		  $timeout = 10;
		  curl_setopt($ch, CURLOPT_URL, $url);
		  curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
		  curl_setopt($ch, CURLOPT_USERAGENT, 'hacker/2.1 (+http://www.vasya-pupkin.com/bot.html)');
		  curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, $timeout);
		  #curl_setopt($ch, CURLOPT_REFERER, 'http://www.vasya-pupkin.com/');
		  curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 0);
		  curl_setopt($ch, CURLOPT_ENCODING, 'gzip,deflate');
		  curl_setopt($ch, CURLOPT_TIMEOUT, 10);
		  $data = curl_exec($ch);
		  $answer = curl_getinfo($ch);
		  curl_close($ch);
		  
		  if($debug)
		  {
		    ...
  		}
		  
		  if($answer['http_code'] == '200')
		  {
			  if(strlen($data) > 0) return $data;
			  else return false;
		  }
		  else return false;
	  }
	  ?>

Пример использования:

    <?
    $content = getPageContent('http://www.vasya-pupkin.com/bot.html');
    ?>

В случае возникновения ошибки или пустой страницы функция возвращает **false**.


## Комментарии:

Можно пользоваться stackoverflow, google, php.net и прочими сайтами. Побольше комментариев к коду приветствуется. Почему можно взломать, как и прочее.
