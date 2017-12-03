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
ourselves with is ``strategies`` and ``filters``::

    'doctrine-hydrator' => array(
        'DbApi\\V1\\Rest\\Artist\\ArtistHydrator' => array(
            'entity_class' => 'Db\\Entity\\Artist',
            'object_manager' => 'doctrine.entitymanager.orm_default',
            'by_value' => true,
            'filters' => array(
                'artist_default' => array(
                    'condition' => 'and',
                    'filter' => 'DbApi\\Hydrator\\Filter\\ArtistDefault',
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
business being returned as part of a ``User`` resource of the API::

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
    
Hydrator filters are attched to hydrators through configuration which is part of the Apigility configuration::

    'doctrine-hydrator' => array(
        'DbApi\\V1\\Rest\\User\\UserHydrator' => array(
            ...
            'filters' => array(
                'artist_default' => array(
                    'condition' => 'and',
                    'filter' => 'DbApi\\V1\\Hydrator\\Filter\\UserFilter',
                ),
            ),          
        ),

It is recommended to only use one hydrator filter per hydrator.  
