Override Default Entities
=========================

This example shows how to override default entities and modify them to use BIGINT.
The same procedure is used for any other time default entities will be overridden.

The default foreign keys and primary keys in ``zf-oauth2-doctrine`` are standard
integers.  Many times databases will plan for the future and use BIGINT for these
keys.  It's not possible to create referential integrity between an INTEGER and a
BIGINT.  It is possible to quickly implement BIGINT integers in ``zf-oauth2-doctrine``.

First you'll need a new module for the modified xml.  Call it ``OAuth2``.  Next copy
the ``zf-oauth2-doctrine/config/orm/*.xml`` into your new
module ``OAuth2/config/orm`` (create this directory).

Edit the xml and change all `integers` to `bigint`.  Next edit your
`oauth2.doctrine-orm.global.php` file and set `'enable_default_entities' => false`

Now in OAuth2/config/module.config.php add the Doctrine config and alias the xml::

    'doctrine' => array(
        'driver' => array(
            'oauth2_driver' => array(
                'class' => 'Doctrine\\ORM\\Mapping\\Driver\\XmlDriver',
                'paths' => array(
                    0 => __DIR__ . '/orm',
                ),
            ),
            'orm_default' => array(
                'class' => 'Doctrine\\ORM\\Mapping\\Driver\\DriverChain',
                'drivers' => array(
                    'ZF\\OAuth2\\Doctrine\\Entity' => 'oauth2_driver',
                ),
            ),
        ),
    ),
