Hydrators
=========

If you're unfamiliar with hydrators
`read Zend Framework's manual on Hydrators <https://framework.zend.com/manual/2.4/en/modules/zend.stdlib.hydrator.html>`_
then
`read Doctrine's manual on Hydrators <https://github.com/doctrine/DoctrineModule/blob/master/docs/hydrator.md>`_
then
`read phpro/zf-doctrine-hydration-module <https://github.com/phpro/zf-doctrine-hydration-module>`_

Doctrine in Apigility uses an Abstract Factory to create hydrators.  **There should be no need to create your own hydrators.**  That bold statement is true because we're taking a white-gloved approach to
data handling.  By using Hydrator Strategies and Filters we can fine tune the configuration for each hydrator used for a Doctrine entity
assigned to a resource.

`phpro/zf-doctrine-hydration-module <https://github.com/phpro/zf-doctrine-hydration-module>`_ makes working with hydrators easy by
moving each field which could be hydrated into Doctrine in Apigility's configuration file.  The only configuration we need to concern
ourselves with is ``strategies`` and ``filters``

.. code-block:: php

  <?php
    'doctrine-hydrator' => array(
        'DbApi\\V1\\Rest\\Artist\\ArtistHydrator' => array(
            'entity_class' => 'Db\\Entity\\Artist',
            'object_manager' => 'doctrine.entitymanager.orm_default',
            'by_value' => true,
            'filters' => array(
                'artist_default' => array(
                    'condition' => 'and',
                    'filter' => 'DbApi\\V1\\Hydrator\\Filter\\ArtistFilter',
                ),
            ),
            'strategies' => array(
                'performance' => 'ZF\\Doctrine\\Hydrator\\Strategy\\CollectionLink',
                'artistGroup' => 'ZF\\Doctrine\\Hydrator\\Strategy\\CollectionLink',
                'artistAlias' => 'ZF\\Doctrine\\Hydrator\\Strategy\\CollectionLink',
            ),
            'use_generated_hydrator' => true,
        ),


Hydrator Filters
----------------

A hydrator filter returns a boolean for whether the field passed to the filter should be rendered or not.  They are used for removing
fields, associations, and collections from a hydrating entity where that data should not be passed through the API.  For instance
a ``User`` entity may contain a ``password`` field used for authentication.  While this field is used inside the application is has no
business being returned as part of a ``User`` resource of the API

.. code-block:: php

  <?php
    namespace DbApi\V1\Hydrator\Filter;

    use Zend\Hydrator\Filter\FilterInterface;

    class UserFilter implements
        FilterInterface
    {
        public function filter($field)
        {
            $excludeFields = [
                'password',
            ];

            return (! in_array($field, $excludeFields));
        }
    }

Hydrator filters are attched to hydrators through configuration which is part of the Doctrine in Apigility configuration

.. code-block:: php

  <?php
    'doctrine-hydrator' => array(
        'DbApi\\V1\\Rest\\User\\UserHydrator' => array(
            ...
            'filters' => array(
                'user_filter' => array(
                    'condition' => 'and',
                    'filter' => 'DbApi\\V1\\Hydrator\\Filter\\UserFilter',
                ),
            ),
        ),

It is recommended to only use one hydrator filter per hydrator.


Hydrator Strategies
-------------------

A hydrator strategy may be attached to any field, association, or collection which is derived by hydrating an entity.
`API-Skeletons/zf-doctrine-hydrator <https://github.com/API-Skeletons/zf-doctrine-hydrator>`_ has three hydration strategies and rather
than create a long article about how to create your own strategies it is the recommendation of this boot that you only use one of
these three strategies for your hydrated data.

There is a pitfall to using strategies; especially when a strategy extracts a collection.  An entity which is a member of a collection
which is extracted as part of a strategy for a parent entity will (should) have a reference back to the parent entity.  This creates
a cyclic relationship.  Often developers turn to the ``max_depth`` parameter of ``zf-hal`` to correct this but this approach is really
hack and should be avoided.  Instead of trying to limit the depth replace the reference to the parent entity in the collection with
an ``EntityLink``; that is, just provide a link to the canonical resource rather than the whole extracted entity.

Using hydrator strategies you can create an elegant response for your API.  A good strategy for applying Hydrator Strategies is to
create your API resource through the Apigility UI then fetch an entity through the API.  You'll see every relationship for the entity
often as an empty class ``{}``.  For each of these empty classes, often they are collections, assign a hydrator strategy.  Don't try to
over-do it; you don't need to return the entire database with each request; just make sure the requesting client can get to any data
which is related to the resource.  It's ok if a client makes 2 or 3 requests to get all thier data.


.. role:: raw-html(raw)
   :format: html

.. note::
  Authored by Tom H Anderson of `API Skeletons <https://apiskeletons.com>`_.
  All rights reserved.  :raw-html:`<form style="display: inline" action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top"><input type="hidden" name="cmd" value="_s-xclick"><input type="hidden" name="hosted_button_id" value="WHR95HM3DMYAQ"><input type="image" src="https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif" border="0" name="submit" alt="PayPal - The safer, easier way to pay online!"><img alt="" border="0" src="https://www.paypalobjects.com/en_US/i/scr/pixel.gif" width="1" height="1"></form>`
  if you find this book useful.
