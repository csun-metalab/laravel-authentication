# Laravel Directory Authentication
Composer package for Laravel 5.0 and above to allow for directory-based authentication.

This package adds the ability to perform both local database and LDAP-based authentication.

Once the user has been authenticated via the directory service (such as LDAP) a local database lookup is performed in order to resolve a user model instance that is accessible through `Auth::user()`.

If you wish to use only local database authentication (but still leverage the functionality of the package) instead, please look at the [Laravel Directory Authentication (Database Only)](https://github.com/csun-metalab/laravel-directory-authentication/blob/dev/README_DB.md) Readme.

**NOTE:** authorization was added in Laravel 5.1 and therefore there is a breaking change in the `MetaUser` class. If you need to use this package for Laravel 5.0, please use any version of this package prior to `1.6.0`.

## Table of Contents

* [Installation](#installation)
* [Required Environment Variables](#required-environment-variables)
* [Optional Environment Variables](#optional-environment-variables)
* [The HandlerLDAP class](#the-handlerldap-class)
* [The MetaUser Class](#the-metauser-class)
* [Creating a Custom User Class](#creating-a-custom-user-class)
* [Authenticating](#authenticating)
* [Masquerading](#masquerading)
* [Adding New LDAP Records](#adding-new-ldap-records)
* [Modifying Existing LDAP Records](#modifying-existing-ldap-records)
* [Modifying LDAP User Passwords](#modifying-ldap-user-passwords)

## Installation

**NOTE:** There are different configuration entries to change further down based on the Laravel version you are using.

### Composer, Environment, and Service Provider

#### Composer

To install from Composer, use the following commands in this order:

```
composer require tiesa/ldap:dev-master
composer require csun-metalab/laravel-directory-authentication
```

There are two commands because `tiesa/ldap` cannot be in the `require` clause for this package as that dependency only exposes a `dev-master` release and no versioned releases. Due to this, it evaluates to a `minimum-stability` of `dev` and cannot be included as a dependency of a package going into a project with the `minimum-stability` of `stable`.

#### Environment

Now, add the following line(s) to your `.env` file:

```
LDAP_HOST=
LDAP_BASE_DN=
```

You may also elect to add the following optional line(s) to your `.env` file to customize the functionality further:

```
LDAP_VERSION=3
LDAP_ALLOW_NO_PASS=false
LDAP_DN=
LDAP_PASSWORD=

LDAP_SEARCH_USER_ID=employeeNumber
LDAP_SEARCH_USERNAME=uid
LDAP_SEARCH_MAIL=mail
LDAP_SEARCH_MAIL_ARRAY=mailLocalAddress
LDAP_SEARCH_USER_QUERY=

LDAP_DB_USER_ID_PREFIX=
LDAP_DB_RETURN_FAKE_USER=false

LDAP_OVERLAY_DN=

LDAP_ADD_BASE_DN=
LDAP_ADD_DN=
LDAP_ADD_PW=

LDAP_MODIFY_METHOD=self
LDAP_MODIFY_BASE_DN=
LDAP_MODIFY_DN=
LDAP_MODIFY_PW=
```

#### Service Provider

Next, add the service provider to your `providers` array in `config/app.php` in Laravel as follows:

```
'providers' => [
   //...

   CSUNMetaLab\Authentication\Providers\AuthServiceProvider::class,

   // You can also use this based on Laravel convention:
   // 'CSUNMetaLab\Authentication\Providers\AuthServiceProvider',

   //...
],
```

### Configuration (Laravel 5.2 and Above)

Next, change the `driver` to `ldap` and add the full classname of the user model to use for database lookups in the `users` array within `config/auth.php` as follows:

```
'providers' => [
    'users' => [
        // LDAP authentication method
        'driver' => 'ldap',

        // this can be any subclass of the CSUNMetaLab\Authentication\MetaUser class
        // or the MetaUser class itself since it works out of the box
        'model' => CSUNMetaLab\Authentication\MetaUser::class,
    ],
],
```

### Configuration (Laravel 5.0 and 5.1)

Next, change the `driver` to `ldap` and change the `model` attribute to be the full classname of the user model to use for database lookups within `config/auth.php` as follows:

```
// LDAP authentication method
'driver' => 'ldap',

// this can be any subclass of the CSUNMetaLab\Authentication\MetaUser class
// or the MetaUser class itself since it works out of the box 
'model' => 'CSUNMetaLab\Authentication\MetaUser',
```

### Publish Configuration

Finally, run the following Artisan command to publish the configuration:

```
php artisan vendor:publish
```

## Required Environment Variables

You added two environment variables to your `.env` file that control the connection to the LDAP server as well as its searching subtree.

### LDAP_HOST

This is the hostname or IP address of the LDAP server.

### LDAP_BASE_DN

This is the base DN under which all people to be searched for reside. This may be something like the following:

`ou=People,ou=Auth,o=Organization`

Someone under this base DN may therefore exist with the following record:

`uid=person,ou=People,ou=Auth,o=Organization`

**NOTE:** The environment variables of `LDAP_BASE_DN` and `LDAP_OVERLAY_DN` are incompatible. Please use either one or the other. Ideally, you would use `LDAP_OVERLAY_DN` if your LDAP server makes use of an overlay to create a logical root for multiple subtrees. Otherwise, use `LDAP_BASE_DN` for the search base.

## Optional Environment Variables

There are several optional environment variables that may be added to customize the functionality of the package even further.

### LDAP_VERSION

The version of LDAP to use when performing operations. The default is `3`.

### LDAP_ALLOW_NO_PASS

True to turn off user password validation (and therefore use the admin DN and password for searching for people). When false, binding and searching is done with the username and password passed to `Auth::attempt()`.

Default is `false`.

### LDAP_DN

The admin DN to use when binding and searching for people. This is only used when user password validation is turned off (so searching can still happen).

### LDAP_PASSWORD

The admin password to use when binding and searching for people. This is only used when user password validation is turned off (so searching can still happen).

### LDAP_SEARCH_USER_ID

The field to use when looking-up a person in LDAP by their user ID; this is typically a numeric field.

Default is `employeeNumber`. This is the field value that will be used when checking for a user in the associated data model and database table/view.

This can also be the same as the `LDAP_SEARCH_USERNAME` value if you want to perform both the LDAP and database lookups with the same value.

### LDAP_SEARCH_USERNAME

The field to use when looking-up a person in LDAP by their username; this is typically the POSIX ID.

Default is `uid`. This is the value that will be used to perform the search operation as the username passed to the call to `Auth::attempt()`.

If password validation is turned on, this is also the username that will be used for the bind operation when combined with the base DN.

### LDAP_SEARCH_MAIL

The field to use when looking-up a person in LDAP by their email address.

Default is `mail`.

### LDAP_SEARCH_MAIL_ARRAY

The field to use when looking-up a person in LDAP by all valid email addresses and aliases; this is typically an array attribute.

Default is `mailLocalAddress`.

### LDAP_SEARCH_USER_QUERY

Optional search query that will replace the default query executed by the package during user directory searches. If not specified, the query that will be used is the equivalent of `(|(uid=%s)(mail=%s)(mailLocalAddress=%s))` depending on the values of `LDAP_SEARCH_USERNAME`, `LDAP_SEARCH_MAIL`, and `LDAP_SEARCH_MAIL_ARRAY`.

If specified, this query needs to be a `vsprintf()`-compatible string and use `%s` as the placeholder for the search value.

### LDAP_DB_USER_ID_PREFIX

Optional prefix before the value of the employee ID primary key in the associated database table/view.

Default is blank (no prefix).

For example, LDAP might store the employee ID as numeric (`XXXXXXXXX`) but your database stores it as a textual value prepended with `members:` (ex: `members:XXXXXXXXX`). You would then set this value to `members:` and your database lookups would work.

### LDAP_DB_RETURN_FAKE_USER

Determines whether to return an actual user instance if the user was found in the directory but not in the database.

If `true`, a user instance will be returned with LDAP attributes that can then be used to [create the user in the database](#authentication-with-provisioning).

If `false`, the authentication attempt will fail outright if the user is not in the database because `Auth::attempt()` will return `false`.

Default is `false`.

### LDAP_OVERLAY_DN

Overlay DN to give a consistent logical root for the search, add and modify subtrees in the directory.

Default is a blank string.

**NOTE:** The environment variables of `LDAP_BASE_DN` and `LDAP_OVERLAY_DN` are incompatible. Please use either one or the other. Ideally, you would use `LDAP_OVERLAY_DN` if your LDAP server makes use of an overlay to create a logical root for multiple subtrees. Otherwise, use `LDAP_BASE_DN` for the search base.

### LDAP_ADD_BASE_DN

The base DN that will be used for adding objects to a subtree.

If this value is left blank, the value of `LDAP_BASE_DN` will be used instead.

Default is a blank string.

### LDAP_ADD_DN

The admin DN to use when adding objects to the `LDAP_ADD_BASE_DN` subtree.

If this value is left blank, the value of `LDAP_DN` will be used instead.

Default is a blank string.

### LDAP_ADD_PW

The password to use when adding objects to the `LDAP_ADD_BASE_DN` subtree.

If this value is left blank, the value of `LDAP_PASSWORD` will be used instead.

Default is a blank string.

### LDAP_MODIFY_METHOD

The method that will be used for modifying objects in the subtree from the `LDAP_MODIFY_BASE_DN` value. Allowed values are `self` and `admin`.

If the value is `self` then the binding user would be able to modify his own attributes in the directory.

If the value is `admin` then the bind used would be made up of the combination of `LDAP_MODIFY_DN` and `LDAP_MODIFY_PW`.

Default is `self`.

### LDAP_MODIFY_BASE_DN

The base DN that will be used for modifying objects in a subtree. If this value is left blank, the value of `LDAP_ADD_BASE_DN` will be used instead.

Default is a blank string.

### LDAP_MODIFY_DN

The admin DN to use when modifying objects in the `LDAP_MODIFY_BASE_DN` subtree.

If this value is left blank, the value of `LDAP_ADD_DN` will be used instead.

Default is a blank string.

### LDAP_MODIFY_PW

The password to use when modifying objects in the `LDAP_MODIFY_BASE_DN` subtree.

If this value is left blank, the value of `LDAP_ADD_PW` will be used instead.

Default is a blank string.

## The `HandlerLDAP` Class

The core LDAP functionality is provided by the `CSUNMetaLab\Authentication\Handlers\HandlerLDAP` class. This contains the connect, bind, and searching features used during authentication.

This class can also be used on its own to provide general LDAP searching functionality as well and give you a consistent interface to execute lookup operations.

You can return an instance of `HandlerLDAP` by executing a call to its factory class of `CSUNMetaLab\Authentication\Factories\HandlerLDAPFactory` as follows:

`$ldap = HandlerLDAPFactory::fromDefaults();`

You will now have an instance of the class loaded with the default configuration from `config/ldap.php`. If you wanted to search for someone outside of authentication purposes you could do the following, for example:

```
// this assumes the $ldap variable already holds an instance of HandlerLDAP

// you can optionally set a new base DN to use during the connection and bind
// $ldap->setBaseDN("ou=Administrators,ou=Auth,o=Organization");

// you can now connect and subsequently bind to the directory in one of two ways:
$ldap->connect(); // will use the values of LDAP_DN and LDAP_PASSWORD
//$ldap->connect($username, $password); // will use explicit credentials

// you can also set a new base DN to use prior to a search too
// $ldap->setBaseDN("ou=Managers,ou=Auth,o=Organization");

// now that you have a connection, you can perform any valid search query within
// the base DN specified in the configuration
$result = $ldap->searchByQuery("uid=manager");

// you can then retrieve the relevant attributes that you want
$name = $ldap->getAttributeFromResults($result, "displayName");
$email = $ldap->getAttributeFromResults($result, "mail");
if(!empty($name) && !empty($email)) {
  return "Manager {$name} has the email of {$email}";
}

```

## The `MetaUser` Class

This package comes with the `CSUNMetaLab\Authentication\MetaUser` class that is configured to work properly with the directory authentication methods. It also supports [masquerading as another user](#masquerading) right out of the box with zero additional configuration.

It provides baseline implementations of the methods to look-up a user by both an identifier in the database as well as the combination of identifier and "Remember Me" token. Only the `findForAuth()` method is invoked automatically upon successful authentication; the other `findForAuthToken()` method is provided for convenience if you are implementing "Remember Me" functionality in your application.

This class expects a local database table called `users` with a primary key of `user_id`. You are free to use this class directly as long as you meet those two requirements.

## Creating a Custom User Class

It is recommended that at the minimum you create a class that extends from `CSUNMetaLab\Authentication\MetaUser` since that will give you greater control over the authentication and database functionality.

### Simple Subclass

A simple subclass is the following:

```
<?php

namespace App\Models;

use CSUNMetaLab\Authentication\MetaUser;

class User extends MetaUser
{
  protected $fillable = ['user_id', 'first_name', 'last_name', 'display_name', 'email'];

  // this must be set for models that do not use an auto-incrementing PK
  public $incrementing = false;
}

?>
```

This class still uses the `users` table but also defines the primary key as non auto-incrementing. In addition, it defines a `fillable` array and allows `User` instances to be created via mass-assignment.

It still, however, relies on the implementations of `findForAuth()` and `findForAuthToken()` that are present in the `MetaUser` class.

### Comprehensive Subclass

**IMPORTANT:** If the primary key for your model is anything other than `user_id` you need to implement a comprehensive subclass, specifically the `findForAuth()` and `findForAuthToken()` methods. The `$identifier` parameter will be the value of the primary key for the model instance when authentication checks are performed (such as when the `auth` middleware is used). Otherwise, your initial login will work but the session values will be incorrect so no subsequent login checks on the next request will go through.

A more comprehensive subclass could be the following:

```
<?php

namespace App\Models;

use CSUNMetaLab\Authentication\MetaUser;

class User extends MetaUser
{
  protected $fillable = ['user_id', 'first_name', 'last_name', 'display_name', 'email'];
  protected $primaryKey = "uid";
  protected $table = "people";

  // this must be set for models that do not use an auto-incrementing PK
  public $incrementing = false;

  // implements MetaAuthenticatableContract#findForAuth
  public static function findForAuth($identifier) {
    return self::where('uid', '=', $identifier)
      ->where('status', 'Active')
      ->first();
  }

  // implements MetaAuthenticatableContract#findForAuthToken
  public static function findForAuthToken($identifier, $token) {
    return self::where('uid', '=', $identifier)
      ->where('remember_token', '=', $token)
      ->where('status', 'Active')
      ->first();
  }
}

?>
```

A couple of additional things are happening in this subclass:

1. Both the table and primary keys have been changed to use custom values
2. The two post-authentication methods have been overridden to perform status checks to ensure only active users are allowed to use the application

## Authenticating

Authenticating with either the `MetaUser` or your custom subclass works just like regular database authentication in Laravel.

This package makes a distinction between whether a user exists only in the directory or in both the directory and the database.

### Authentication Only

#### Without Returning a Fake User Instance

If `LDAP_DB_RETURN_FAKE_USER` is set to `false`, you could invoke `Auth::attempt()` in the following way:

```
$creds = ['username' => 'admin', 'password' => '123'];

if(Auth::attempt($creds)) {
  // valid user in the directory and the database, so proceed into the application!
  redirect('home');
}
else
{
  // not a valid user (either in the directory or the database) so return back
  // to the login page with an error
  redirect('login')->withErrors([
    'Invalid username or password'
  ]);
}
```

#### With Returning a Fake User Instance

If `LDAP_DB_RETURN_FAKE_USER` is set to `true`, you could invoke `Auth::attempt()` in the following way:

```
$creds = ['username' => 'admin', 'password' => '123'];

if(Auth::attempt($creds)) {
  // valid user in the directory, so let's check to see whether the user is
  // valid within the database too
  if(Auth::user()->isValid) {
    // user also exists within the database, so proceed into the application!
    redirect('home');
  }
  else
  {
    // user does not exist within the database, so go ahead and return back
    // to the login screen with an error
    Auth::logout();
    redirect('login')->withErrors([
      'Local user record could not be found'
    ]);
  }
}
else
{
  // not a valid user in the directory, so go ahead and return back to the
  // login screen with an error
  redirect('login')->withErrors([
    'Invalid username or password'
  ]);
}
```

### Authentication with Provisioning

**NOTE:** Authentication with provisioning only functions if `LDAP_DB_RETURN_FAKE_USER` has been set to `true`.

Because a distinction is made between valid directory and valid database users, you have the option of provisioning the user in your local database if he exists within the directory but not yet within the database.

The user instance represented by `Auth::user()` also gets an array of attributes back from LDAP. This array can be accessed as `searchAttributes`. It contains the following key/value pairs:

* `uid` (matches the value of search attribute `LDAP_SEARCH_USERNAME`)
* `user_id` (matches the value of search attribute `LDAP_SEARCH_USER_ID`)
* `first_name` (matches the value of `givenName`)
* `last_name` (matches the value of `sn`)
* `display_name` (matches the value of `display_name`)
* `email` (matches the value of `LDAP_SEARCH_MAIL`)

You can now change your authentication procedure from above to be the following:

```
$creds = ['username' => 'admin', 'password' => '123'];

if(Auth::attempt($creds)) {
  // valid user in the directory, so let's check to see whether the user is
  // valid within the database too
  if(Auth::user()->isValid) {
    // user also exists within the database, so proceed into the application!
    redirect('home');
  }
  else
  {
    // user does not exist within the database, so go ahead and provision the
    // record and perform an automatic login; we are assuming the user instance
    // uses a model called User here
    $attrs = Auth::user()->searchAttributes;
    $user = User::create([
      'user_id' => $attrs['user_id'],
      'first_name' => $attrs['first_name'],
      'last_name' => $attrs['last_name'],
      'display_name' => $attrs['display_name'],
      'email' => $attrs['email'];
    ]);

    // perform the automatic login and the redirect
    Auth::login($user);
    redirect('home');
  }
}
else
{
  // not a valid user in the directory, so go ahead and return back to the
  // login screen with an error
  redirect('login')->withErrors([
    'Invalid username or password'
  ]);
}
```

## Masquerading

This package also supports the ability for the logged-in user to become another user out of the box. This is especially useful in situations where an admin-level user may need to enter another user's account in order to triage and solve a problem directly.

### Become Another User

In order to become another user, it's as simple as finding the other user and then switching the logged-in user. The previous user is maintained in the session so switching back can be seamless.

```
// findOrFail is used here with a custom User instance but you can use any
// subclass of MetaUser that you would like or MetaUser itself
$switchUser = User::findOrFail('employee');

if(Auth::user()->masqueradeAsUser($switchUser)) {
  // successfully masquerading!
}
else
{
  // masquerade attempt failed
}
```

### Am I Masquerading?

It may be useful to determine whether the user reported by `Auth::user()` is actually a masqueraded user:

```
if(Auth::user()->isMasquerading()) {
  // I am acting as someone else
}
else
{
  // it's the original logged-in user
}
```

You can also retrieve the instance of the masquerading user (the original user that logged-in) in the following way:

```
if(Auth::user()->isMasquerading()) {
  $originalUser = Auth::user()->getMasqueradingUser();
  return "This account is really " . $originalUser->display_name;
}
else
{
  // not masquerading
  return "Not masquerading";
}
```

### Stop Masquerading

Finally, it's simple to stop masquerading and return to your original user account.

```
if(Auth::user()->stopMasquerading()) {
  // I have returned to my original user account
}
else
{
  // this account was not masquerading
}
```

Calls to `Auth::user()` will now once again report the original logged-in user.

## Adding New LDAP Records

This package will attempt to use the information configured in the following three `.env` values first when performing add operations:

* `LDAP_ADD_BASE_DN`: the subtree where new entries will be added. If this is left empty, the value of `LDAP_BASE_DN` will be used instead.
* `LDAP_ADD_DN`: the admin DN to use when adding entries. If this is left empty, the value of `LDAP_DN` will be used instead.
* `LDAP_ADD_PW`: the admin password to use when adding entries. If this is left empty, the value of `LDAP_PASSWORD` will be used instead.

You can add new records to an LDAP subtree using the `HandlerLDAP` class:

```
use CSUNMetaLab\Authentication\Factories\HandlerLDAPFactory;
use CSUNMetaLab\Authentication\Factories\LDAPPasswordFactory;

public function addNewObject() {
  $name = "New User";
  $email = "newuser@example.com";
  $pw = "1234";

  $pwhash = LDAPPasswordFactory::SSHA($pw); // SSHA hash
  $uid = 'ex_' . bin2hex(random_bytes(4)); // random UID

  // retrieve a new HandlerLDAP instance and connect to the configured
  // host
  $ldap = HandlerLDAPFactory::fromDefaults();
  $ldap->connect();

  // set up the attributes to be added to the new record
  $nameArr = explode(" ", $name);
  $attrs = [
    'objectClass' => 'inetOrgPerson',
    'uid' => $uid,
    'mail' => $email,
    'displayName' => $name,
    'cn' => $name,
    'sn' => (!empty($nameArr[1]) ? $nameArr[1] : "Example"),
    'givenName' => $nameArr[0],
    'userPassword' => $pwhash,
  ];

  // add the object to the add subtree
  $success = $ldap->addObject($uid, $attrs);
  if($success) {
    return "Successfully registered account {$name}! UID: {$uid}";
  }

  return "Could not register {$name}. UID: {$uid}. Please try again";
}
```

## Modifying Existing LDAP Records

This package will attempt to use the information configured in the following four `.env` values first when performing modify operations:

* `LDAP_MODIFY_METHOD`: if `self`, then a user can bind as himself and modify his own attributes. If `admin` then the value of `LDAP_MODIFY_DN` and `LDAP_MODIFY_PW` will be used when performing the modification bind.
* `LDAP_MODIFY_BASE_DN`: the subtree where entries will be modified. If this is left empty, the value of `LDAP_ADD_BASE_DN` will be used instead.
* `LDAP_MODIFY_DN`: the admin DN to use when modifying entries. If this is left empty, the value of `LDAP_ADD_DN` will be used instead.
* `LDAP_MODIFY_PW`: the admin password to use when modifying entries. If this is left empty, the value of `LDAP_ADD_PW` will be used instead.

You can modify existing records in an LDAP subtree using the `HandlerLDAP` class:

```
use CSUNMetaLab\Authentication\Factories\HandlerLDAPFactory;
use CSUNMetaLab\Authentication\Factories\LDAPPasswordFactory;

public function modifyObject() {
  // this lets a logged-in user modify their own attributes due to the bind
  // as "self" as opposed to the LDAP_MODIFY_DN and LDAP_MODIFY_PW values
  // from the .env file
  $cur_pw = "1234";

  $email = "newuser@example.com";
  $newName = "Different Name";
  $newNameArr = explode(" ", $newName);

  // retrieve a new HandlerLDAP instance and connect to the configured
  // host
  $ldap = HandlerLDAPFactory::fromDefaults();
  $ldap->connect();

  // get the matching object for the logged-in user and resolve its DN
  $obj = $ldap->searchByAuth($email);
  $dn = $ldap->getAttributeFromResults($obj, 'dn');

  // modification method of "self": make sure the user has the correct
  // password by performing another bind; this will not be executed if
  // the admin modify DN and password are being used instead
  if(config('ldap.modify_method') == "self") {
    $ldap->connectByDN($dn, $cur_pw);
  }

  // set a new name for the user
  $success = $ldap->modifyObject($dn, [
    'displayName' => $newName,
    'cn' => $newName,
    'givenName' => $newNameArr[0],
    'sn' => $newNameArr[1],
  ]);

  if($success) {
    return "Successfully changed the name for {$email}! DN: {$dn}";
  }

  return "Could not change the name for {$email}. DN: {$dn}";
}
```

## Modifying LDAP User Passwords

While you can perform a `modifyObject()` with the `userPassword` attribute and follow similar steps as those in [Modifying Existing LDAP Records](#modifying-existing-ldap-records), the `HandlerLDAP` class also provides a convenience method for changing user passwords.

The password will be generated as a SSHA hash using a cryptographically-secure salt based on the output of `openssl_random_pseudo_bytes()` using a four-byte string:

```
public function modifyUserPassword() {
  // this lets a logged-in user modify their own attributes due to the bind
  // as "self" as opposed to the LDAP_MODIFY_DN and LDAP_MODIFY_PW values
  // from the .env file
  $cur_pw = "1234";

  $email = "newuser@example.com";
  $new_pw = "2345";

  // retrieve a new HandlerLDAP instance and connect to the configured
  // host
  $ldap = HandlerLDAPFactory::fromDefaults();
  $ldap->connect();

  // get the matching object for the logged-in user and resolve its DN
  $obj = $ldap->searchByAuth($email);
  $dn = $ldap->getAttributeFromResults($obj, 'dn');

  // modification method of "self": make sure the user has the correct
  // password by performing another bind; this will not be executed if
  // the admin modify DN and password are being used instead
  if(config('ldap.modify_method') == "self") {
    $ldap->connectByDN($dn, $cur_pw);
  }

  // set a new password for the user
  $success = $ldap->modifyObjectPassword($dn, $new_pw);

  if($success) {
    return "Successfully changed the password for {$email}! DN: {$dn}";
  }

  return "Could not change the password for {$email}. DN: {$dn}";
}
```