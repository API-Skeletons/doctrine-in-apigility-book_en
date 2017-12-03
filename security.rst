Security
========


Authentication and Authorization
--------------------------------

To secure your API and create security for users to log into your site please follow the 
`instructions on zf-mvc-auth User Differentiation <https://github.com/TomHAnderson/apigility-documentation/blob/e0491084c70b11f29409f48fb454dd45c0ec1658/auth/user-differentiation.md>`_
(note this documentation points to a pull request and will be updated once this pull request is accepted).


Query Providers
---------------

A query provider is a class which provides a Doctrine QueryBuilder to the DoctrineResource in ``zfcampus\zf-apigility-doctrine``.
This prepared QueryBuilder is then used to fetch the entity or collection through the Doctrine Object Manager.  The same Query Provider
may be used for querying an entity or collection because when querying an entity the id from the route is assigned to the QueryBuilder
after it is fetched from the Query Provider.  For every verb (GET, POST, PATCH, etc.) your API handles through a Doctrine resource a
Query Provider may be assigned.  

Query Providers are used for security and for extending the functionality of the QueryBuilder object they provide.  For instance,
given a User API resource for which only the user who owns a resource may PATCH the resource, a QueryBuilder object can assign an
``andWhere`` parameter to the QueryBuilder to specify that only the current user may fetch the resoruce::

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
(that is, to an inverse and to a owner entity for every relationship) you can add complicated joins to your Query Provider::

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

