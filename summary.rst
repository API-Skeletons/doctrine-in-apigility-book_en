Design Summary
==============

This summary gives an overview of a very good design pattern for using Doctrine in Apigility.  Each section will have it's own page and
is included here to give the developer a birds-eye view of the pattern.


Files
-----

Doctrine in Apigility creates two files when a new resource is assigned to an API.  These are::
  
  [Resource]Collection.php
  [Resource]Resource.php

As an example for an Artist resource these files full path will be::

  module/DbApi/src/DbApi/V1/Rest/Artist/ArtistCollection.php
  module/DbApi/src/DbApi/V1/Rest/Artist/ArtistResource.php'

There should never be a need to modify these files.  Let me repeat, these files are not intended to override the ancestor objects.  They
exist here as part of the depenency injection strategy Doctrine in Apigility uses.  Again, **DO NOT** modify these files.


Security
--------

The design for Doctrine in Apigility expects a two-layered security strategy.  The first layer is ACL (or RBAC if you prefer and are dedicated)
and the second layer is Query Providers.  ACL Authorization is handled by Apigility and Query Providers are handled by Doctrine in Apigility.


ACL Security
------------

Doctrine in Apigility expects you to implement the Authorization created with 
`zfcampus/zf-mvc-auth <https://github.com/zfcampus/zf-mvc-auth>`_ for your project.  This probably means implementing ACL in your 
application and assigning roles to the differnet HTTP verbs each role can access.  For instance a DELETE verb may only be available
to an administrator.  This is explained in detail in `Authorization <authorization>`_.


Query Provider Security
-----------------------

For any resource where the access to the resource is limited a Query Provider should be created.  Query Providers are small classes
which return a Doctrine QueryBuilder object.  By default the QueryBuilder contains only the entity assigned to the resource the user
is requesting.  By extending the QueryBuilder with filters and joins the query will return filtered data based on a particular user or 
security permission of the user the QueryBuilder, when ran, will produce SQL that adds new security to the resource.

For instance, if a UserResource is secured by ACL to only USER roles but each user can only PATCH to their own entity the Query Provider
may read::

    final class UserPatch extends AbstractQueryProvider
    {
        public function createQuery(ResourceEvent $event, $entityClass, $parameters)
        {
            $queryBuilder = $this->getObjectManager()->createQueryBuilder();
            $queryBuilder
                ->select('row')
                ->from($entityClass, 'row')
                ->andWhere(row.user = :user)
                ->setParameter('user', $this->getAuthentication()->getIdentity()->getUser())
                ;

           return $queryBuilder;
        }
    }

Now when the QueryBuilder is ran inside the `DoctrineResource <https://github.com/zfcampus/zf-apigility-doctrine/blob/master/src/Server/Resource/DoctrineResource.php>`_
the id for the user passed to the patch will be appended to the QueryBuilder.  If the id does not belong to the current user then the
QueryBuilder will return no results and a 404 will be thrown to the user trying to edit a record which is not theirs.

More complicated examples **rely on your metadata being complete**.  If your metadata defines joines to and from every join (that is, to an inverse and to a owner entity for every relationship) you can add complicated joins to your Query Provider::

    $queryBuilder
        ->innerJoin('row.performance', 'performance')
        ->innerJoin('performance.artist', 'artist')
        ->innerJoin('artist.artistGroup', 'artistGroup')
        ->andWhere($queryBuilder->expr()->isMemberOf(':user', 'artistGroup.user'))
        ->setParameter('user', $this->getAuthentication()->getIdentity()->getUser())
        ;


HATEOAS, Hydrators, and Hydrator Strategies & Filters
-------------------------------------------

If you're unfamiliar with hydrators 
`read Zend Framework's manual on Hydrators <https://framework.zend.com/manual/2.4/en/modules/zend.stdlib.hydrator.html>`_ 
then 
`read Doctrine's manual on Hydrators <https://github.com/doctrine/DoctrineModule/blob/master/docs/hydrator.md>`_

Because Doctrine hydrators can extract relationships the default response from a Doctrine in Apigility Resource will include an ``_embedded`` section with the extracted entities and their ``_embedded`` and so on.  **For special cases only** does 
`zfcampus/zf-hal <https://github.com/zfcampus/zf-hal>`_ have a max_depth parameter <https://apigility.org/documentation/modules/zf-hal#key-metadata_map>`_.  This special case is not intended to correct issues with HATEOAS in Doctrine in Apigility.  When you encounter
a cyclic association in Doctrine in Apigility the correct way to handle is it using Hydrator Strategies and Filters.

Hydrators in Doctrine in Apigility are handled by `phpro/zf-doctrine-hydration-module <https://github.com/phpro/zf-doctrine-hydration-module>`_.  Familiarity with this module is very important to understanding how to extend hydrators without creating special case hydrators.  Doctrine in Apigility uses an Abstract Factory to create hydrators.  

**There should be no need to create your own hydrators.**  That bold statement is true because we're taking a white-gloved approach to 
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

Here is the ArtistDefault filter::

    namespace DbApi\Hydrator\Filter;

    use Zend\Hydrator\Filter\FilterInterface;

    class ArtistDefault implements
        FilterInterface
    {
        public function filter($field)
        {
            $excludeFields = [
                'artistMergeKeep',
                'artistMergeMerge',
            ];

            if (in_array($field, $excludeFields)) {
                return false;
            }

            return true;
        }
    }
    
This should be quite obvious; fields are excluded from being hydrated (or extracted) based on the filter.

Next are Hydrator Strategies.  The module `API-Skeletons/zf-doctrine-hydrator <https://github.com/API-Skeletons/zf-doctrine-hydrator>`_
provides all the hydrator strategies you will need.  More information on these strategies in `hydration <hydration>`_.


