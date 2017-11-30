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

Inside the Db module create a ``config` directory and inside the directory create an ``orm`` directory.  This is where you will export
your entity metadata to from Skipper.


config/autoload/local.php
-------------------------

You'll need to create a configuration for Doctrine in your local.php.  This file is used so each deployment can have independent
configuration.  Here is an example file::

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


Migrations and Fixtures
-----------------------

This is a new topic to many developers so I'm including this special note to you, reader, to become familiar with them.
`migrations <https://github.com/doctrine/migrations>`_ and `fixtures <https://github.com/API-Skeletons/zf-doctrine-data-fixture>`_

  
