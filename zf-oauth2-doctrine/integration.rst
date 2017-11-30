zf-doctrine-apigility Integration
=================================

Validate zf-apigility-doctrine resources
----------------------------------------

To validate the OAuth2 session with Query Create Filters and Query Providers implement
``ZF\OAuth2\Doctrine\OAuth2ServerInterface`` and use ``ZF\OAuth2\Doctrine\OAuth2ServerTrait``.
Then call ``$result = $this->validateOAuth2($scope);`` in the filter function.
