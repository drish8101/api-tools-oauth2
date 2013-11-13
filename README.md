Zf-OAuth2
=========

ZF2 module for [OAuth2](http://oauth.net/2/) authentication.

This module uses the [oauth2-server-php](https://github.com/bshaffer/oauth2-server-php)
library by Brent Shaffer to provide OAuth2 support.

Installation
------------

You can install using:

```
curl -s https://getcomposer.org/installer | php
php composer.phar install
```

Configuration
-------------

This module uses a PDO database to manage the OAuth2 information (users,
client, token, etc).  The database structure is stored in `data/db_oauth2.sql`.

```sql
CREATE TABLE oauth_clients (
    client_id VARCHAR(80) NOT NULL,
    client_secret VARCHAR(80) NOT NULL,
    redirect_uri VARCHAR(2000) NOT NULL,
    grant_tpyes VARCHAR(80),
    CONSTRAINT client_id_pk PRIMARY KEY (client_id)
);
CREATE TABLE oauth_access_tokens (
    access_token VARCHAR(40) NOT NULL,
    client_id VARCHAR(80) NOT NULL,
    user_id VARCHAR(255),
    expires TIMESTAMP NOT NULL,
    scope VARCHAR(2000),
    CONSTRAINT access_token_pk PRIMARY KEY (access_token)
);
CREATE TABLE oauth_authorization_codes (
    authorization_code VARCHAR(40) NOT NULL,
    client_id VARCHAR(80) NOT NULL,
    user_id VARCHAR(255),
    redirect_uri VARCHAR(2000),
    expires TIMESTAMP NOT NULL,
    scope VARCHAR(2000),
    CONSTRAINT auth_code_pk PRIMARY KEY (authorization_code)
);
CREATE TABLE oauth_refresh_tokens (
    refresh_token VARCHAR(40) NOT NULL,
    client_id VARCHAR(80) NOT NULL,
    user_id VARCHAR(255),
    expires TIMESTAMP NOT NULL,
    scope VARCHAR(2000),
    CONSTRAINT refresh_token_pk PRIMARY KEY (refresh_token)
);
CREATE TABLE oauth_users (
    username VARCHAR(255) NOT NULL,
    password VARCHAR(2000),
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    CONSTRAINT username_pk PRIMARY KEY (username)
);
CREATE TABLE oauth_scopes (
    type VARCHAR(255) NOT NULL DEFAULT "supported",
    scope VARCHAR(2000),
    client_id VARCHAR (80)
);
CREATE TABLE oauth_jwt (
    client_id VARCHAR(80) NOT NULL,
    subject VARCHAR(80),
    public_key VARCHAR(2000),
    CONSTRAINT client_id_pk PRIMARY KEY (client_id)
);
```

For security reason, we encrypt the fields `client_secret` (table `oauth_clients`) and `password` (table `oauth_users`) using the [bcrypt](http://en.wikipedia.org/wiki/Bcrypt) algorithm (using the class [Zend\Crypt\Password\Bcrypt](http://framework.zend.com/manual/2.2/en/modules/zend.crypt.password.html#bcrypt)).

In order to configure zf-oauth2 module for the database access, you need to copy the file
`config/oauth2.local.php.dist` in your `/config/autoload/oauth2.local.php` of your ZF2 application and edit with your DB credentials (DNS, username, password).

We also included a SQLite database in `data/dbtest.sqlite` that you can use as test environment.
In this database you will find a test client account with the client_id `testclient` and `testpass` as client_secret.
If you want to use this database you can configure your `/config/autoload/oauth2.local.php` as follow:

```
return array(
    'oauth2' => array(
        'db' => array(
            'dsn' => 'sqlite:<path to zf-oauth2 module>/data/dbtest.sqlite'
        )
    )
);
```

How to test OAuth2
------------------

To test the OAuth2 module you have to add a client_id and a client_secret into the oauth2 database. If you are using the SQLite test database you don't need to add a client_id and use the default `testclient/testpass` account.

Because we encrypt the testpass using the bcrypt algorithm you need to encrypt the password using the [Zend\Crypt\Password\Bcrypt](http://framework.zend.com/manual/2.2/en/modules/zend.crypt.password.html#bcrypt) class. We provided a simple script in `/bin/bcrypt.php` to generate the hash value of a user's password. You can use this tool from the command line, using the following syntax:

```bash
php bin/bcrypt.php testpass
```

where `testpass` is the user's password that you want to encrypt. The output of the previous command will be the hash value of the user's password, a string of 60 bytes like this:

```
$2y$14$f3qml4G2hG6sxM26VMq.geDYbsS089IBtVJ7DlD05BoViS9PFykE2
```

After the generation of the hash value of the password (client_secret) you can add a new `client_id` in the database using the following SQL statement:

```sql
INSERT INTO oauth_clients (
    client_id,
    client_secret,
    redirect_uri)
VALUES (
    "testclient",
    "$2y$14$f3qml4G2hG6sxM26VMq.geDYbsS089IBtVJ7DlD05BoViS9PFykE2",
    "http://fake/"
);
```

To test the OAuth2 module, you can use an HTTP client like [HTTPie](https://github.com/jkbr/httpie)
or [CURL](http://curl.haxx.se/).  The examples below use HTTPie and the test account `testclient/testpass`.

REQUEST TOKEN (client\_credentials)
-----------------------------------

You can request an OAuth2 token using the following HTTPie command:

```bash
http --auth testclient:testpass -f POST http://<URL of your ZF2 app>/oauth grant_type=client_credentials
```

This POST requests a new token to the OAuth2 server using the *client_credentials*
mode. This is typical in machine-to-machine interaction for application access.
If everything works fine, you should receive a response like this:

```json
{
    "access_token":"03807cb390319329bdf6c777d4dfae9c0d3b3c35",
    "expires_in":3600,
    "token_type":"bearer",
    "scope":null
}
```

*Security note:* because this POST uses basic HTTP authentication, the
`client_secret` is exposed in plaintext in the HTTP request. To protect this
call, a [TLS/SSL](http://en.wikipedia.org/wiki/Transport_Layer_Security)
connection is required.


AUTHORIZE (code)
----------------

If you have to integrate an OAuth2 service with a web application, you need to
use the Authorization Code grant type.  This grant requires an approval step to
authorize the web application. This step is implemented using a simple form that
requests the user approve access to the resource (account).  This module
provides a simple form to authorize a specific client. This form can be accessed
by a browser using the following URL:

```bash
http://<URL of your ZF2 app>/oauth/authorize?response_type=code&client_id=testclient&state=xyz
```

This page will render the form asking the user to authorize or deny the access
for the client. If they authorize the access, the OAuth2 module will reply with
an Authorization code. This code must be used to request an OAuth2 token; the
following HTTPie command provides an example of how to do that:

```bash
http --auth testclient:testpass -f POST http://<URL of your ZF2 app>/oauth grant_type=authorization_code&code=YOUR_CODE
```

Access a test resource
----------------------

When you obtain a valid token, you can access a restricted API resource. The
OAuth2 module is shipped with a test resource that is accessible with the URL
`/oauth/resource`. This is a simple resource that returns JSON data.

To access the test resource, you can use the following HTTPie command:

```bash
http -f POST http://<URL of your ZF2 app>/oauth/resource access_token=000ab5afab4cbbbda803fb9e50e7943f5e766748
# or
http http://<<URL of your ZF2 app>/oauth/resource "Authorization:Bearer 000ab5afab4cbbbda803fb9e50e7943f5e766748"
```

As you can see, the OAuth2 module supports the data either via POST, using the
`access_token` value, or using the [Bearer](http://tools.ietf.org/html/rfc6750)
authorization header.

How to protect your API using OAuth2
------------------------------------

You can protect your API using the following code (for instance, at the top of a
controller):

```php
if (!$this->server->verifyResourceRequest(OAuth2Request::createFromGlobals())) {
    // Not authorized return 401 error
    $this->getResponse()->setStatusCode(401);
    return;
}
```

where `$this->server` is an instance of `OAuth2\Server` (see the
[AuthController.php](https://github.com/zfcampus/zf-oauth2/blob/master/src/ZF/OAuth2/Controller/AuthController.php)).
