Authentication
==============

`The official Apigility documentation <https://apigility.org/documentation/auth/intro>`_
describes the lightweight authentication infrastructure included with Apigility and how it is intended
to be extended.  Extending this infrastructure is exactly what is required for Doctrine
in Apigility.

When building an API application the central authentication is usually done through an
OAuth2 implicit and/or authorization_code grant type.  To support authorization you must
redirect users to a login page when a user tries to authenticate through these routes.

`api-skeletons/zf-oauth2-doctrine-identity <https://github.com/API-Skeletons/zf-oauth2-doctrine-skeletons>`_
replaces the default AuthenticatedIdentity provided by Apigility.  This class
``->getIdentity()`` returns the AccessToken granted through OAuth2.  From the Access
Token you can fetch the authenticated User entity and OAuth2 Client entity.

Authentication should occur every time a user tries to access a route they are not authorized for
and the user is not already Authenticated.  To trigger Authentication you must have authorization
setup for your resources.
