.. role:: raw-html(raw)
   :format: html

.. note::
  Freely contributed by Tom H Anderson of `API Skeletons <https://apiskeletons.com>`_.
  All rights reserved.  :raw-html:`<form style="display: inline" action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top"><input type="hidden" name="cmd" value="_s-xclick"><input type="hidden" name="hosted_button_id" value="WHR95HM3DMYAQ"><input type="image" src="https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif" border="0" name="submit" alt="PayPal - The safer, easier way to pay online!"><img alt="" border="0" src="https://www.paypalobjects.com/en_US/i/scr/pixel.gif" width="1" height="1"></form>`
  if you find this book useful.


Skeleton Application
====================

This section describes how to build a Doctrine in Apigility application from the Apigility Skeleton.

First clone the Apigility Skeleton module::

  git clone git@github.com:zfcampus/zf-apigility-skeleton

`cd` into the zf-apigility-skeleton and require these repositories::

  composer require zfcampus/zf-apigility-doctrine
  composer require doctrine/doctrine-orm-module
  composer require zendframework/zend-mvc-console

When prompted select the ``config/modules.config.php`` file to add the new module with the exception of the ``ZF\Apigility\Doctrine\Admin``
which should be added to the ``config/development.config.php.dist``.

The direction to include ``zend-mvc-console`` is debatable.  This module will no longer be maintained in the future but the discussion of
how to replace it is outside the scope of this book.


Db Module
---------

For the entities in my application I create a module called simply 'Db'.  Your mileage may vary as this is often given the namespace
of the application or company name.  At any rate this will be the module where your XML annotations and entity classes are stored.

You will be using the command line to create your entities.  It should be rare to create a custom function inside an entity as these
usually belong in the
`Repository <http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/working-with-objects.html#custom-repositories>`_
for the entity instead.  Following this rule your entities can be managed by the command line tool ``orm:generate-entities``.  This command
takes an argument of the path to create the entities.  If you give it the path ``module/Db/src`` it will create the entities in the
``module/Db/src/Db/Entity directory based on the namespace of the entities.  This shows that we need the subdir `Db` under `src`.  In
recent Zend Framework work the autoloading of modules has been moved to the composer.json file and you can still use that but you must be
sure to have the correct directory structure for the code generation.

Inside the Db module create a ``config`` directory and inside that directory create an ``orm`` directory.  This is where you will export
your entity metadata to from Skipper.


config/autoload/local.php
-------------------------

You'll need to create a configuration for Doctrine in your local.php.  This file is used so each deployment can have independent
configuration.  Here is an example file::

.. code-block:: php

    <?php
    return array(
        'doctrine' => array(
            'connection' => array(
                'orm_default' => array(
                    'driverClass' => 'Doctrine\\DBAL\\Driver\\PDOMySql\\Driver',
                    'params' => array(
                        'user' => 'root',
                        'password' => '123',
                        'host' => 'mysql',
                        'dbname' => 'etreedb',
                        'port' => '3306',
                        'charset' => 'utf8',
                        'collate' => 'utf8_general_ci',
                    ),
                ),
            ),
        ),
    );


Export Metadata and Create Entities
-----------------------------------

For a brand new ERD export the XML to the ``module/Db/config/orm`` directory then run::

  php public/index.php orm:generate-entities module/Db/src

and  your PHP skeleton code will be done.


Recommended Extension Repositories
==================================


Migrations and Fixtures
-----------------------

This is a new topic to many developers so I'm including this special note to you, reader, to become familiar with them.
`migrations <https://github.com/doctrine/migrations>`_ and `fixtures <https://github.com/API-Skeletons/zf-doctrine-data-fixture>`_


Doctrine QueryBuilder
---------------------

Implementing the repository `zfcampus/zf-doctrine-querybuilder <https://github.com/zfcampus/zf-doctrine-querybuilder>`_
is the most important extension for Doctrine in Apigility.  This repository allows your clients to create complex queries and sorting
on individual resources.   For instance if you give a user access to an ``Performance`` resource and that resources returns performances
then ``zf-doctrine-querybuilder`` will allow a client to return only a subset of the data they have access to, for instance just
performances from a given state.  Implementation is covered in `Doctrine QueryBuilder <querybuilder>`_.


Doctrine Repository Plugins
---------------------------

`API-Skeletons/zf-doctrine-repository <https://github.com/API-Skeletons/zf-doctrine-repository>`_
provides a method to override the default ``Repository`` factory for Doctrine and implements a plugin architecture which can be used
in lieu of dependency injection into repositories.  This repository provides a clean method for interacting with external resources from
within a repository and its use is strongly encouraged.


Doctrine Hydrators
------------------

Covered also in `hydrators <hydrators>`_ is `API-Skeletons/zf-doctrine-hydrator <https://github.com/API-Skeletons/zf-doctrine-hydrator>`_.
This repository includes three hydrator plugins which are used to create a fluent HATEOAS HAL API response.


OAuth2 for Doctrine in Apigility
--------------------------------

OAuth2 is implemented with several repositories, each building on the last.  The first is
`API-Skeletons/zf-oauth2-doctrine <https://github.com/API-Skeletons/zf-oauth2-doctrine>`_ which provies the metadata to attach OAuth2
entities to your existing schema via a dynamic hook to your User entity.

`API-Skeletons/zf-oauth2-doctrine-console <https://github.com/API-Skeletons/zf-oauth2-doctrine-console>`_ provies console routes for
managing ``zf-oauth2-doctrine`` resources.

`API-Skeletons/zf-oauth2-doctrine-identity <https://github.com/API-Skeletons/zf-oauth2-doctrine-identity>`_ should have been a part of
``zf-oauth2-doctrine`` from the beginning.  That being said, this repository replaces the ``AuthenticatedIdentity`` of
``zfcampus/zf-mvc-auth`` with an identity which contains access to the ``AccessToken``, ``User``, ``Client``, and ``AuthorizationService``.  This allows you to inject the ``AuthenticationService`` into your classes then access the identity via
``$authorizationService->getIdentity()`` then get the User class via ``->getUser()``.  The result of all this is a cleaner way to work
with ORM objects only throughout your application.

`API-Skeletons/zf-oauth2-doctrine-permissions-acl <https://github.com/API-Skeletons/zf-oauth2-doctrine-permissions-acl>`_ uses the
identity from ``zf-oauth2-doctrine-identity`` to create ACL permissions on your resources.  This module cleanly provides integration
with ``zfcampus/zf-mvc-auth`` and is covered in `authorization <authorization>`_.


