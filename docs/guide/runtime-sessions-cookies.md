Sessions and Cookies
====================

Sessions and cookies allow data to be persisted across multiple user requests. In plain PHP, you may access them
through the global variables `$_SESSION` and `$_COOKIE`, respectively. Yii encapsulates sessions and cookies as objects
and thus allows you to access them in an object-oriented fashion with additional nice enhancements.


## Sessions <a name="sessions"></a>

Like [requests](runtime-requests.md) and [responses](runtime-responses.md), you can get access to sessions via
the `session` [application component](structure-application-components.md) which is an instance of [[yii\web\Session]],
by default.


### Opening and Closing Sessions <a name="opening-closing-sessions"></a>

To open and close a session, you can do the following:

```php
$session = Yii::$app->session;

// check if a session is already open
if ($session->isActive) ...

// open a session
$session->open();

// close a session
$session->close();

// destroys all data registered to a session.
$session->destroy();
```

You can call the [[yii\web\Session::open()|open()]] and [[yii\web\Session::close()|close()]] multiple times
without causing errors. This is because internally the methods will first check if the session is already opened.


### Accessing Session Data <a name="access-session-data"></a>

To access the data stored in session, you can do the following:

```php
$session = Yii::$app->session;

// get a session variable. The following usages are equivalent:
$language = $session->get('language');
$language = $session['language'];
$language = isset($_SESSION['language']) ? $_SESSION['language'] : null;

// set a session variable. The following usages are equivalent:
$session->set('language', 'en-US');
$session['language'] = 'en-US';
$_SESSION['language'] = 'en-US';

// remove a session variable. The following usages are equivalent:
$session->remove('language');
unset($session['language']);
unset($_SESSION['language']);

// check if a session variable exists. The following usages are equivalent:
if ($session->has('language')) ...
if (isset($session['language'])) ...
if (isset($_SESSION['language'])) ...

// traverse all session variables. The following usages are equivalent:
foreach ($session as $name => $value) ...
foreach ($_SESSION as $name => $value) ...
```

> Info: When you access session data through the `session` component, a session will be automatically opened
if it has not been done so before. This is different from accessing session data through `$_SESSION`, which requires
an explicit call of `session_start()`.

When working with session data that are arrays, the `session` component has a limitation which prevents you from
directly modifying an array element. For example,

```php
$session = Yii::$app->session;

// the following code will NOT work
$session['captcha']['number'] = 5;
$session['captcha']['lifetime'] = 3600;

// the following code works:
$session['captcha'] = [
    'number' => 5,
    'lifetime' => 3600,
];

// the following code also works:
echo $session['captcha']['lifetime'];
```

You can use one of the following workarounds to solve this problem:

```php
$session = Yii::$app->session;

// directly use $_SESSION (make sure Yii::$app->session->open() has been called)
$_SESSION['captcha']['number'] = 5;
$_SESSION['captcha']['lifetime'] = 3600;

// get the whole array out first, modify it and then save it back
$captcha = $session['captcha'];
$captcha['number'] = 5;
$captcha['lifetime'] = 3600;
$session['captcha'] = $captcha;

// use ArrayObject instead of array
$session['captcha'] = new \ArrayObject;
...
$session['captcha']['number'] = 5;
$session['captcha']['lifetime'] = 3600;

// store array data by keys with common prefix
$session['captcha.number'] = 5;
$session['captcha.lifetime'] = 3600;
```

For better performance and code readability, we recommend the last workaround. That is, instead of storing
an array as a single session variable, you store each array element as a session variable which shares the same
key prefix with other array elements.


### Custom Session Storage <a name="custom-session-storage"></a>

The default [[yii\web\Session]] class stores session data as files on the server. Yii also provides the following
session classes implementing different session storage:

* [[yii\web\DbSession]]: stores session data in a database table.
* [[yii\web\CacheSession]]: stores session data in a cache with the help of a configured [cache component](caching-data.md#cache-components).
* [[yii\redis\Session]]: stores session data using [redis](http://redis.io/) as the storage medium.
* [[yii\mongodb\Session]]: stores session data in a [MongoDB](http://www.mongodb.org/).

All these session classes support the same set of API methods. As a result, you can switch to use a different
session storage without the need to modify your application code that uses session.

> Note: If you want to access session data via `$_SESSION` while using custom session storage, you must make
  sure that the session is already started by [[yii\web\Session::open()]]. This is because custom session storage
  handlers are registered within this method.

To learn how to configure and use these component classes, please refer to their API documentation. Below is
an example showing how to configure [[yii\web\DbSession]] in the application configuration to use database table
as session storage:

```php
return [
    'components' => [
        'session' => [
            'class' => 'yii\web\DbSession',
            // 'db' => 'mydb',  // the application component ID of the DB connection. Defaults to 'db'.
            // 'sessionTable' => 'my_session', // session table name. Defaults to 'session'.
        ],
    ],
];
```

You also need to create the following database table to store session data:

```sql
CREATE TABLE session
(
    id CHAR(40) NOT NULL PRIMARY KEY,
    expire INTEGER,
    data BLOB
)
```

where 'BLOB' refers to the BLOB-type of your preferred DBMS. Below are the BLOB type that can be used for some popular DBMS:

- MySQL: LONGBLOB
- PostgreSQL: BYTEA
- MSSQL: BLOB

> Note: According to the php.ini setting of `session.hash_function`, you may need to adjust
  the length of the `id` column. For example, if `session.hash_function=sha256`, you should use
  length 64 instead of 40.


### Flash Data <a name="flash-data"></a>

Flash data is a special kind of session data which, once set in one request, will only be available during
the next request and will be automatically deleted afterwards. Flash data is most commonly used to implement
messages that should only be displayed to end users once, such as a confirmation message displayed after
a user successfully submits a form.

You can set and access flash data through the `session` application component. For example,

```php
$session = Yii::$app->session;

// Request #1
// set a flash message named as "postDeleted"
$session->setFlash('postDeleted', 'You have successfully deleted your post.');

// Request #2
// display the flash message named "postDeleted"
echo $session->getFlash('postDeleted');

// Request #3
// $result will be false since the flash message was automatically deleted
$result = $session->hasFlash('postDeleted');
```

Like regular session data, you can store arbitrary data as flash data.

When you call [[yii\web\Session::setFlash()]], it will overwrite any existing flash data that has the same name.
To append new flash data to the existing one(s) of the same name, you may call [[yii\web\Session::addFlash()]] instead.
For example,

```php
$session = Yii::$app->session;

// Request #1
// add a few flash messages under the name of "alerts"
$session->addFlash('alerts', 'You have successfully deleted your post.');
$session->addFlash('alerts', 'You have successfully added a new friend.');
$session->addFlash('alerts', 'You are promoted.');

// Request #2
// $alerts is an array of the flash messages under the name of "alerts"
$alerts = $session->getFlash('alerts');
```

> Note: Try not to use [[yii\web\Session::setFlash()]] together with [[yii\web\Session::addFlash()]] for flash data
  of the same name. This is because the latter method will automatically turn the flash data into an array so that it
  can append new flash data of the same name. As a result, when you call [[yii\web\Session::getFlash()]], you may
  find sometimes you are getting an array while sometimes you are getting a string, depending on the order of
  the invocation of these two methods.


## Cookies <a name="cookies"></a>

Yii represents each cookie as an object of [[yii\web\Cookie]]. Both [[yii\web\Request]] and [[yii\web\Response]]
maintain a collection of cookies via the property named `cookies`. The cookie collection in the former represents
the cookies submitted in a request, while the cookie collection in the latter represents the cookies that are to
be sent to the user.


### Reading Cookies <a name="reading-cookies"></a>

You can get the cookies in the current request using the following code:

```php
// get the cookie collection (yii\web\CookieCollection) from "request" component
$cookies = Yii::$app->request->cookies;

// get the "language" cookie value. If the cookie does not exist, return "en" as the default value.
$language = $cookies->getValue('language', 'en');

// an alternative way of getting the "language" cookie value
if (($cookie = $cookies->get('language')) !== null) {
    $language = $cookie->value;
}

// you may also use $cookies like an array
if (isset($cookies['language'])) {
    $language = $cookies['language']->value;
}

// check if there is a "language" cookie
if ($cookies->has('language')) ...
if (isset($cookies['language'])) ...
```


### Sending Cookies <a name="sending-cookies"></a>

You can send cookies to end users using the following code:

```php
// get the cookie collection (yii\web\CookieCollection) from "response" component
$cookies = Yii::$app->response->cookies;

// add a new cookie to the response to be sent
$cookies->add(new \yii\web\Cookie([
    'name' => 'language',
    'value' => 'zh-CN',
]));

// remove a cookie
$cookies->remove('language');
// equivalent to the following
unset($cookies['language']);
```

Besides the [[yii\web\Cookie::name|name]], [[yii\web\Cookie::value|value]] properties shown in the above
examples, the [[yii\web\Cookie]] class also defines other properties to fully represent all possible information
of cookies, such as [[yii\web\Cookie::domain|domain]], [[yii\web\Cookie::expire|expire]]. You may configure these
properties as needed to prepare a cookie and then add it to the response's cookie collection.

> Note: For better security, the default value of [[yii\web\Cookie::httpOnly]] is set true. This helps mitigate
the risk of client side script accessing the protected cookie (if the browser supports it). You may read
the [httpOnly wiki article](https://www.owasp.org/index.php/HttpOnly) for more details.


### Cookie Validation <a name="cookie-validation"></a>

When you are reading and sending cookies through the `request` and `response` components like shown in the last
two subsections, you enjoy the added security of cookie validation which protects cookies from being modified
on the client side. This is achieved by signing each cookie with a hash string, which allows the application to
tell if a cookie is modified on the client side or not. If so, the cookie will NOT be accessible through the
[[yii\web\Request::cookies|cookie collection]] of the `request` component.

> Info: If a cookie fails the validation, you may still access it through `$_COOKIE`. This is because third-party
libraries may manipulate cookies in their own way, which does not involve cookie validation.

Cookie validation is enabled by default. You can disable it by setting the [[yii\web\Request::enableCookieValidation]]
property to be false, although we strongly recommend you do not do so.

> Note: Cookies that are directly read/sent via `$_COOKIE` and `setcookie()` will NOT be validated.

When using cookie validation, you must specify a [[yii\web\Request::cookieValidationKey]] that will be used to generate
the aforementioned hash strings. You can do so by configuring the `request` component in the application configuration:

```php
return [
    'components' => [
        'request' => [
            'cookieValidationKey' => 'fill in a secret key here',
        ],
    ],
];
```

> Info: [[yii\web\Request::cookieValidationKey|cookieValidationKey]] is critical to your application's security.
  It should only be known to people you trust. Do not store it in version control system.
