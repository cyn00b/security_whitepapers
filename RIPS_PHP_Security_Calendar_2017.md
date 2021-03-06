## RIPSTECH PRESENTS PHP SECURITY CALENDAR 2017

### Day 1 - Wish List
Can you spot the vulnerability?

```php
class Challenge {
    const UPLOAD_DIRECTORY = './solutions/';
    private $file;
    private $whitelist;

    public function __construct($file) {
        $this->file = $file;
        $this->whitelist = range(1, 24);
    }

    public function __destruct() {
        if (in_array($this->file['name'], $this->whitelist)) {
            move_uploaded_file(
                $this->file['tmp'],
                self::UPLOAD_DIRECTORY . $this->file['name']
            );
        }
    }
}

$challenge = new Challenge($_FILES['solution']);
```

The challenge contains an arbitrary file upload vulnerability in line 13. The operation in_array() is used in line 12 to check if the file name is a number. However, it is type-unsafe because the third parameter is not set to 'true'. Hence, PHP will try to type-cast the file name to an integer value when comparing it to the array $whitelist (line 8). As a result it is possible to bypass the whitelist by prepending a value in the range of 1 and 24 to the file name, for example "5backdoor.php". The uploaded PHP file then leads to code execution on the web server.


### Day 2 - Twig
Can you spot the vulnerability?

```php
// composer require "twig/twig"
require 'vendor/autoload.php';

class Template {
    private $twig;

    public function __construct() {
        $indexTemplate = '<img ' .
            'src="https://loremflickr.com/320/240">' .
            '<a href="{{link|escape}}">Next slide »</a>';

        // Default twig setup, simulate loading
        // index.html file from disk
        $loader = new Twig\Loader\ArrayLoader([
            'index.html' => $indexTemplate
        ]);
        $this->twig = new Twig\Environment($loader);
    }

    public function getNexSlideUrl() {
        $nextSlide = $_GET['nextSlide'];
        return filter_var($nextSlide, FILTER_VALIDATE_URL);
    }

    public function render() {
        echo $this->twig->render(
            'index.html',
            ['link' => $this->getNexSlideUrl()]
        );
    }
}

(new Template())->render();
```

The challenge contains a cross-site scripting vulnerability in line 26. There are two filters that try to assure that the link that is passed to the <a> tag is a genuine URL. First, the filter_var() function in line 22 checks if it is a valid URL. Then, Twig's template escaping is used in line 10 that avoids breaking out of the href attribute.

The vulnerability can still be exploited with the following URL: ?nextSlide=javascript://comment%250aalert(1).
The payload does not involve any markup characters that would be affected by Twig's escaping. At the same time, it is a valid URL for filter_var(). We used a JavaScript protocol handler, followed by a JavaScript comment introduced with // and then the actual JS payload follows on a newline. When the link is clicked, the JavaScript payload is executed in the browser of the victim.


### Day 3 - Snow Flake
Can you spot the vulnerability?

```php
function __autoload($className) {
    include $className;
}

$controllerName = $_GET['c'];
$data = $_GET['d'];

if (class_exists($controllerName)) {
    $controller = new $controllerName($data);
    $controller->render();
} else {
    echo 'There is no page with this name';
}

class HomeController {
    private $data;

    public function __construct($data) {
        $this->data = $data;
    }

    public function render() {
        if ($this->data['new']) {
            echo 'controller rendering new response';
        } else {
            echo 'controller rendering old response';
        }
    }
}
```

In this code are two security bugs. A file inclusion vulnerability is triggered by the call of class_exists() in line 8. Here, the existance of a user supplied class name is checked. This automatically invokes the custom autoloader in line 1 in case the class name is unknown which will try to include unknown classes. An attacker can abuse this file inclusion by using a path traversal attack. The lookup for the class name ../../../../etc/passwd will leak the passwd file. The attack only works until version 5.3 of PHP.

But there is a second bug that also works in recent PHP versions. In line 9, the class name is used for a new object instantiation. The first argument of its constructor is under the attackers control as well. Arbitrary constructors of the PHP code base can be called. Even if the code itself does not contain a vulnerable constructor, PHP's built-in class SimpleXMLElement can be used for an XXE attack that also leads to the exposure of files. A real world example of this exploit can be found in our blog post.


### Day 4 - False Beard
Can you spot the vulnerability?

```php
class Login {
    public function __construct($user, $pass) {
        $this->loginViaXml($user, $pass);
    }

    public function loginViaXml($user, $pass) {
        if (
            (!strpos($user, '<') || !strpos($user, '>')) &&
            (!strpos($pass, '<') || !strpos($pass, '>'))
        ) {
            $format = '<xml><user="%s"/><pass="%s"/></xml>';
            $xml = sprintf($format, $user, $pass);
            $xmlElement = new SimpleXMLElement($xml);
            // Perform the actual login.
            $this->login($xmlElement);
        }
    }
}

new Login($_POST['username'], $_POST['password']);
```

This challenge suffers from an XML injection vulnerability in line 13. An attacker can manipulate the XML structure and hence bypass the authentication. There is an attempt to prevent exploitation in lines 8 and 9 by searching for angle brackets but the check can be bypassed with a specifically crafted payload. The bug in this code is the automatic casting of variables in PHP. The PHP built-in function strpos() returns the numeric position of the looked up character. This can be 0 if the first character is the one searched for. The 0 is then type-casted to a boolean false for the if comparison which renders the overall constraint to true. A possible payload could look like user=<"><injected-tag%20property="&pass=<injected-tag>.


### Day 6 - Frost Pattern
Can you spot the vulnerability?


```php
class TokenStorage {
    public function performAction($action, $data) {
        switch ($action) {
            case 'create':
                $this->createToken($data);
                break;
            case 'delete':
                $this->clearToken($data);
                break;
            default:
                throw new Exception('Unknown action');
        }
    }

    public function createToken($seed) {
        $token = md5($seed);
        file_put_contents('/tmp/tokens/' . $token, '...data');
    }

    public function clearToken($token) {
        $file = preg_replace("/[^a-z.-_]/", "", $token);
        unlink('/tmp/tokens/' . $file);
    }
}

$storage = new TokenStorage();
$storage->performAction($_GET['action'], $_GET['data']);
```

This challenge contains a file delete vulnerability. The bug causing this issue is a non-escaped hyphen character (-) in the regular expression that is used in the preg_replace() call in line 21. If the hyphen is not escaped, it is used as a range indicator, leading to a replacement of any character that is not a-z or an ASCII character in the range between dot (46) and underscore (95). Thus dot and slash can be used for directory traversal and (almost) arbitrary files can be deleted, for example with the query parameters action=delete&data=../../config.php.


### Day 7 - Bells
Can you spot the vulnerability?

```php
function getUser($id) {
    global $config, $db;
    if (!is_resource($db)) {
        $db = new MySQLi(
            $config['dbhost'],
            $config['dbuser'],
            $config['dbpass'],
            $config['dbname']
        );
    }
    $sql = "SELECT username FROM users WHERE id = ?";
    $stmt = $db->prepare($sql);
    $stmt->bind_param('i', $id);
    $stmt->bind_result($name);
    $stmt->execute();
    $stmt->fetch();
    return $name;
}

$var = parse_url($_SERVER['HTTP_REFERER']);
parse_str($var['query']);
$currentUser = getUser($id);
echo '<h1>'.htmlspecialchars($currentUser).'</h1>';
```

This challenge suffers from a connection string injection vulnerability in line 4. It occurs because of the parse_str() call in line 21 that behaves very similar to register globals. Query parameters from the referrer are extracted to variables in the current scope, thus we can control the global variable $config inside of getUser() in lines 5 to 8. To exploit this vulnerability we can connect to our own MySQL server and return arbitrary values for username, for example with the referrer http://host/?config[dbhost]=10.0.0.5&config[dbuser]=root&config[dbpass]=root&config[dbname]=malicious&id=1


### Day 8 - Candle
Can you spot the vulnerability?

```php
header("Content-Type: text/plain");

function complexStrtolower($regex, $value) {
    return preg_replace(
        '/(' . $regex . ')/ei',
        'strtolower("\\1")',
        $value
    );
}

foreach ($_GET as $regex => $value) {
    echo complexStrtolower($regex, $value) . "\n";
}
```

This challenge contains a code injection vulnerability in line 4. Prior to PHP 7 the operation preg_replace() contained an eval modifier, short e. If the modifier is set, the second parameter (replacement) is treated as PHP code. We do not have a direct injection point into the second parameter but we can control the value of \\1, as it references the matched regular expression. It is not possible to escape out of the strtolower() call but since the referenced value is inside of double quotes, we can use PHP’s curly syntax to inject other function calls. An attack could look like this: /?.*={${phpinfo()}}.