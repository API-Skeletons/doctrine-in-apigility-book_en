Security
========


Authentication
--------------

Knowing which user is logging into Apigility is not as strait forward as you may guess.  The Basic and Digest
authorization mechanisms rely on a .htpasswd file to store user credentials.  This is not how the majority
of web sites work.

For this document we will assume a User table inside a database with username and password fields.  This
is not a viable data store for Basic and Digest authentication.  This is viable for a Password Grant Type
using OAuth2.  However using the Password Grant Type is highly frowned upon because it gives the site which
builds the login form access to the user credentials and that is not what OAuth2 is about.

We need to authenticate a user before proceeding with an Implicit or Authorization Code grant type in order to
assign the generated Access Token to that authenticated user for future API calls beyond OAuth2.  Here is where
traditional Authentication is useful.  We need to secure the ``ZF\OAuth2`` resource prefix to authenticate via an
old-fashioned login page and here `zfcampus/zf-mvc-auth <https://github.com/zfcampus/zf-mvc-auth>`_ comes in.

As mentioned in this Auth section ``zf-mvc-auth`` can force a configured Authorization adapter to a given API.  The
mechanism `zf-mvc-auth` uses to accomplish this is configuration of resource prefixes to Authentication Types.  When you
create a custom Authentication Adapter you define the Authentication Type it supports.  Let's begin with the configuration

.. code-block:: php

  <?php
    'zf-mvc-auth' => [
        'authentication' => [
            'map' => [
                'DbApi\\V1' => 'oauth2',
                'ZF\\OAuth2' => 'session',
            ],

            'adapters' => [
                'oauth2' => [
                    'adapter' => 'ZF\MvcAuth\Authentication\OAuth2Adapter',
                    'storage' => [
                        'adapter' => 'pdo',
                        'dsn' => 'mysql:host=localhost;dbname=oauth2',
                        'username' => 'username',
                        'password' => 'password',
                        'options' => [
                            1002 => 'SET NAMES utf8', // PDO::MYSQL_ATTR_INIT_COMMAND
                        ],
                    ],
                ],

                'session' => [
                    'adapter' => 'Application\\Authentication\\Adapter\\SessionAdapter',
                ],
            ],
        ],
    ],

With this configuration we have two adapters and they are each mapped to the section of the application we want them to secure.
The ``oauth2`` adapter will be ignored since we're dedicated to finding a user to assign an Access Token to.


Creating an Authentication Adapter
----------------------------------

Adapters must implement `ZF\MvcAuth\Authentication\AdapterInterface <https://github.com/TomHAnderson/zf-mvc-auth/blob/master/src/Authentication/AdapterInterface.php>`_
This interface includes

* ``public function provides()`` - This function will return the Authentication Type(s) this adapter supports.  For our example it will be `session`.
* ``public function matches($type)`` - <sic> (from code) Attempt to match a requested authentication type against what the adapter provides.
* ``public function getTypeFromRequest(Request $request)`` - Still looking for Authentication Types this allows more generic matching based on the request.
* ``public function preAuth(Request $request, Response $response)`` - A helper function ran before ``authenticate``
* ``public function authenticate(Request $request, Response $response, MvcAuthEvent $mvcAuthEvent)`` - Do an authentication attempt

For our examples we will use a route ``/login`` where any unauthenticated user who does not have their credentials stored in the session
and is trying to access a resource under ``ZF\OAuth2`` will be routed to.  This route will show the login page, let the user post to it,
and if successful it will set the userid into the session where our adapter will be looking for it.  When a user
successfully authenticates with this adapter they will be assigned an ``Application\Identity\UserIdentity``

.. code-block:: php

  <?php
    namespace Application\Authentication\Adapter;

    use ZF\MvcAuth\Authentication\AdapterInterface;
    use Zend\Http\Request;
    use Zend\Http\Response;
    use ZF\MvcAuth\Identity\IdentityInterface;
    use ZF\MvcAuth\MvcAuthEvent;
    use Zend\Session\Container;
    use Application\Identity;

    final class SessionAdapter implements
        AdapterInterface,
    {
        public function provides()
        {
            return [
                'session',
            ];
        }

        public function matches($type)
        {
            return $type == 'session';
        }

        public function getTypeFromRequest(Request $request)
        {
            return false;
        }

        public function preAuth(Request $request, Response $response)
        {
        }

        public function authenticate(Request $request, Response $response, MvcAuthEvent $mvcAuthEvent)
        {
            $session = new Container('webauth');

            if ($session->auth) {
                $userIdentity = new Identity\UserIdentity($session->auth);
                $userIdentity->setName('user');

                return $userIdentity;
            }

            // Force login for all other routes
            $mvcAuthEvent->stopPropagation();
            $session->redirect = $request->getUriString();
            $response->getHeaders()->addHeaderLine('Location', '/login');
            $response->setStatusCode(302);
            $response->sendHeaders();

            return $response;
        }
    }

To use this authentication adapter you must assign it to the DefaultAuthenticationListener

.. code-block:: php

  <?php
    namespace Application;

    use ZF\MvcAuth\Authentication\DefaultAuthenticationListener;
    use Zend\ModuleManager\Feature\BootstrapListenerInterface;
    use Zend\EventManager\EventInterface;

    class Module implements
        BootstrapListenerInterface
    {
        public function onBootstrap(EventInterface $e)
        {
            $app = $e->getApplication();
            $container = $app->getServiceManager();

            // Add Authentication Adapter for session
            $defaultAuthenticationListener = $container->get(DefaultAuthenticationListener::class);
            $defaultAuthenticationListener->attach(new Authentication\AuthenticationAdapter());
        }
    }

The ``Application\Identity\UserIdentity`` requires a ``getId()`` function or public id property to return the user id of the
authenticated user.  This will be used by ``zfcampus/zf-oauth2`` to assign the user to ``AccessToken``,
``AuthorizationCode``, and ``RefreshToken`` using the ``ZF\OAuth2\Provider\UserId`` server manager alias.

The Basic and Digest authentication can assign the user because they read the .htpasswd file.  For OAuth2
the user must be fetched using the ``ZF\OAuth2\Provider\UserId`` alias.  You may create your own provider for
a custom method of fetching an id.

This is the default

.. code-block:: php

  <?php
    'service_manager' => [
        'aliases' => [
            'ZF\OAuth2\Provider\UserId' => 'ZF\OAuth2\Provider\UserId\AuthenticationService',
        ],
    ],

With this alias in place the OAuth2 server will store the userid and assign it to the Identity during future requests.
The ``getId()`` or ``id`` property of the provider
of the identity will be used to assign to OAuth2.  When an OAuth2 resource is requested with a Bearer token the user
will be fetched from the database and assigned to the AuthenticatedIdentity.

Here is an example ``UserIdentity``

.. code-block:: php

  <?php
    namespace Application\Identity;

    use ZF\MvcAuth\Identity\IdentityInterface;
    use Zend\Permissions\Rbac\AbstractRole as AbstractRbacRole;

    final class UserIdentity extends AbstractRbacRole implements IdentityInterface
    {
        protected $user;
        protected $name;

        public function __construct(array $user)
        {
            $this->user = $user;
        }

        public function getAuthenticationIdentity()
        {
            return $this->user;
        }

        public function getId()
        {
            return $this->user['id'];
        }

        public function getUser()
        {
            return $this->getAuthenticationIdentity();
        }

        public function getRoleId()
        {
            return $this->name;
        }

        // Alias for roleId
        public function setName($name)
        {
            $this->name = $name;
        }
    }


Authorization
-------------

With our adapter in place it will not secure the ZF\OAuth2 routes because they are by default secured with the
``ZF\MvcAuth\Identity\GuestIdentitiy``.  So we need to add Authorization to the application:

First we'll extend the onBootstrap we just created

.. code-block:: php

  <?php
    public function onBootstrap(EventInterface $e)
    {
        $app = $e->getApplication();
        $container = $app->getServiceManager();

        // Add Authentication Adapter for session
        $defaultAuthenticationListener = $container->get(DefaultAuthenticationListener::class);
        $defaultAuthenticationListener->attach(new Authentication\AuthenticationAdapter());

        // Add Authorization
        $eventManager = $app->getEventManager();
        $eventManager->attach(
            MvcAuthEvent::EVENT_AUTHORIZATION,
            new Authorization\AuthorizationListener(),
            100
        );
    }

And we need to create the AuthorizationListener we just configured

.. code-block:: php

  <?php
    namespace Application\Authorization;

    use ZF\MvcAuth\MvcAuthEvent;

    final class AuthorizationListener
    {
        public function __invoke(MvcAuthEvent $mvcAuthEvent)
        {
            $authorization = $mvcAuthEvent->getAuthorizationService();

            // Deny from all
            $authorization->deny();

            $authorization->addResource('Application\Controller\IndexController::index');
            $authorization->allow('guest', 'Application\Controller\IndexController::index');

            $authorization->addResource('ZF\OAuth2\Controller\Auth::authorize');
            $authorization->allow('user', 'ZF\OAuth2\Controller\Auth::authorize');
        }
    }

Now when a request is made for an implicit grant type through ``ZF\OAuth2`` our new Authentication Adapter will see the user
is not authenticated and store the user's requested url and redirect them to login where, after successfully logging in
they will be directed back to the oauth2 request.  The user will be granted access to the ``ZF\OAuth2\Controller\Auth::authorize``
resource and they will be assigned an Access Token.


Query Providers
---------------

A query provider is a class which provides a Doctrine QueryBuilder to the DoctrineResource in ``zfcampus\zf-apigility-doctrine``.
This prepared QueryBuilder is then used to fetch the entity or collection through the Doctrine Object Manager.  The same Query Provider
may be used for querying an entity or collection because when querying an entity the id from the route is assigned to the QueryBuilder
after it is fetched from the Query Provider.  For every verb (GET, POST, PATCH, etc.) your API handles through a Doctrine resource a
Query Provider may be assigned.

Query Providers are used for security and for extending the functionality of the QueryBuilder object they provide.  For instance,
given a User API resource for which only the user who owns a resource may PATCH the resource, a QueryBuilder object can assign an
``andWhere`` parameter to the QueryBuilder to specify that only the current user may fetch the resoruce

.. code-block:: php

  <?php
    final class UserPatch extends AbstractQueryProvider
    {
        public function createQuery(ResourceEvent $event, $entityClass, $parameters)
        {
            $queryBuilder = $this->getObjectManager()->createQueryBuilder();
            $queryBuilder
                ->select('row')
                ->from($entityClass, 'row')
                ->andWhere($queryBuilder->expr()->eq('row.user', ':user'))
                ->setParameter('user', $this->getAuthentication()->getIdentity()->getUser())
                ;

           return $queryBuilder;
        }
    }

The entity class we are ``select()`` from in the QueryBuilder will always be aliased as ``row``.  This is the only data which should be
returned from a QueryBuilder as a complete Doctrine object.

More complicated examples **rely on your metadata being complete**.  If your metadata defines joins to and from every join
(that is, to an inverse and to a owner entity for every relationship) you can add complicated joins to your Query Provider

.. code-block:: php

  <?php
    $queryBuilder
        ->innerJoin('row.performance', 'performance')
        ->innerJoin('performance.artist', 'artist')
        ->innerJoin('artist.artistGroup', 'artistGroup')
        ->andWhere($queryBuilder->expr()->isMemberOf(':user', 'artistGroup.user'))
        ->setParameter('user', $this->getAuthentication()->getIdentity()->getUser())
        ;


Query Create Filters
--------------------

Query Create Filters are the homolog to Query Providers but for POST requests only.  These are intended to inspect the data the user is
POSTing and if anything is incorrect to return an ``ApiProblem``.  These are not intended to correct the data.  **If an API receives data
which is incorrect it should reject the data, not try to fix it.**


.. role:: raw-html(raw)
   :format: html

.. note::
  Authored by `API Skeletons <https://apiskeletons.com>`_.  All rights reserved.


:raw-html:`<script async src="https://www.googletagmanager.com/gtag/js?id=UA-64198835-2"></script><script>window.dataLayer = window.dataLayer || [];function gtag(){dataLayer.push(arguments);}gtag('js', new Date());gtag('config', 'UA-64198835-2');</script>`
