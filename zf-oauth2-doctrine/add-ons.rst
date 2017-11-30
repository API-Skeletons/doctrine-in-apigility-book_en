Add-Ons
=======

`api-skeletons/zf-oauth2-doctrine-console <https://github.com/API-Skeletons/zf-oauth2-doctrine-console>`_
--------------------

Console routes to manage OAuth2 entities.  Add and list Clients, Scopes, and more from the command line.


`api-skeletons/zf-oauth2-doctrine-identity <https://github.com/API-Skeletons/zf-oauth2-doctrine-identity>`_
--------------------

This should have been a part of zf-oauth2-doctrine from the beginning but because zf-oauth2-doctrine is so mature identity mapping has been implemented in this add-on repository.  This will change the AuthenticatedIdentity of [zfcampus/zf-mvc-auth](https://github.com/zfcampus/zf-mvc-auth) to a Doctrine enabled identity with quick access to the User, Client, and AccessToken entities.

`api-skeletons/zf-oauth2-doctrine-permissions-acl <https://github.com/API-Skeletons/zf-oauth2-doctrine-permissions-acl>`_
----------------------

ACL Permissions and Authenticated Identity management.  This uses interfaces rather than defining entities to create guards on resources based on [api-skeletons/zf-oauth2-doctrine-identity](https://github.com/API-Skeletons/zf-oauth2-doctrine-identity)


`bushbaby/zf-oauth2-doctrine-mutatetablenames <https://github.com/basz/zf-oauth2-doctrine-mutatetablenames>`_
------------------------

 Allows configuration of the table names that this module uses without overriding default entities.
