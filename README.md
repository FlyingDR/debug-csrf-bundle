Debug bundle
============

This bundle provides several services that are mean to simplify developing process of Symfony 2 applications by emulating certain security related features in a case if request is running under debugger.

Debugger detection
------------------
Debugger detection is handled by ```DebuggerDetectorListener```, **Xdebug** and **Zend Debugger** are recognized at this moment.

If some service needs to know if request is running under debugger - it should implement ```DebuggerStatusSubscriberInterface``` to get request status as soon as it will be determined.

CSRF token validation emulation
-------------------------------
When debugging form submissions - it may be useful to disable CSRF token validation under debugger while having CSRF validation enabled.

CSRF token validation emulation is controlled by configuration:
```yaml
debug:
    csrf:
        # true to enable CSRF token validation emulation, false to disable it completely
        enabled: true
        # true to allow use of CSRF token validation emulation permanently, 
        # false to enable it only when running under debugger
        permanent: false
        # Status of emulated CSRF token validation
        token_validation_status: true
```
Unless enabled permanently - validation emulation is disabled automatically for production environment and can also be disabled in development environment. When enabled - it will substitute real CSRF validation with configured value if request was running under debugger. For normal requests all CSRF validation will be passed to real CSRF token manager unless use of emulation is forced by enabling ```permanent``` configuration option.

Debug authentication provider
-----------------------------
Debug authentication provider can be used to transparently authenticate user that is required for debugging / development purposes. Example use case is debugging of some requests that are resides into secured area of application without need to hack application's configuration all the time (slow, boring and error prone).

It is recommended to read [Symfony book](http://symfony.com/doc/current/book/security.html) chapter about security and [Symfony Cookbook recipe](http://symfony.com/doc/current/cookbook/security/custom_authentication_provider.html) about custom authentication providers before using this feature.

### Installation

To use debug authentication provider you're required to perform some changes into your code and configuration.

#### 1. Create your own token builder

Security token is the key component of security in Symfony, you can read more about it [here](http://symfony.com/doc/current/cookbook/security/custom_authentication_provider.html#the-token). Most of tasks related to creating custom authentication provider are handled by this bundle, but since token is too much specific for each particular application - it is your task to generate it. Luckily it is pretty easy. Token builder should implement ```TokenBuilderInterface```, but you can also use ```AbstractTokenBuilder``` as a base in most cases. You need to implement ```build()``` method that receives ```Request``` object and needs to create and return security token, required for your application. Simplest implementation may look something like this:
```php
public function build(Request $request)
{
    // More about token configuration later
    $config = $this->getTokenConfig();
    $token = new UsernamePasswordToken($config['username'], '', 'debug', $config['roles']);
    return $token;
}
```
however real implementation may involve receiving user object from database or some other place. In a case if you use [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle) in your application - you may find it useful to inject their ```UserManager``` into class and take user from it.

Token builder can be used as a simple class, but it is preferable to register it as a service, especially if your token building logic depends on information from other services.

#### 2. Register debug authentication provider

After implementing token builder you need to register debug authentication provider into your security configuration. Registration is explained [here](http://symfony.com/doc/current/cookbook/security/custom_authentication_provider.html#id1) and may look like this:
```yaml
security:
    firewalls:
        main:
            pattern: ^/
            debug:
                token_builder: my.token_builder.service.id
            # Rest of your firewall definition
```
It is important to register debug authentication provider **before** real authentication providers that are used into your application so it will be able to provide debug security token generated by you and by this disable other security mechanisms.

#### 3. Configure debug authentication provider

Debug authentication provider has following configuration options:
 - ```token_builder``` - Either service id or class name of your token builder. This option is required.
 - ```token_config``` - Arbitrary configuration information for your token builder. It is passed directly to your token builder via ```setTokenConfig``` method. For example you can pass username of user, you want to authenticate.
 - ```enabled``` - Allows you to completely disable or force this feature without any additional change. By default it is enabled for debug environments.
 - ```permanent``` - Set to ```true``` to enable user authentication substitution even if request is not running under debugger. May be useful for development purposes if you don't want to authenticate yourself all the time. Defaults to ```false``` meaning that debug authentication provider will only activate itself if request is running under debugger.
 - ```auth_provider``` and ```auth_listener``` options defines services of authentication provider and listener respectively and usually should not be changed.
