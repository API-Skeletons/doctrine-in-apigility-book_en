Introduction
============

The landscape of API strategies grows every day.  In the field of PHP there are
strategies from simple REST-only resources to fully
`Richardson Maturity Model Level 3 <https://martinfowler.com/articles/richardsonMaturityModel.html>`_
API engines.  Apigility falls into the Level 3 category.

An API will serve data and Doctrine in Apigility tries, at it's core, to map Doctrine entities
to API resources.  So a Doctrine enabled resource in Apigility will provide GET, POST, PUT, PATCH, and DELETE.
It also enables the ORM entity relationships to embed related data in a HAL (Hypertext Application Language)
response.  This allows complex joins between entities to be represented in the API response.


When should you use Doctrine in Apigility?
------------------------------------------

When you have a Doctrine project with properly mapped associations, metadata, between entities Doctrine in Apigility
will give you a powerful head-start toward building an API.  Correct metadata is absolutly core to building an API
with this tool.  To help design your ORM `Skipper <https://skipper18.com>`_ is strongly recommended.

You can use Doctrine in Apigility to serve pieces of your schema by filtering with hydrator strategies or you can
serve your entire schema in an "Open-schema API" where relationships between entities are fully explored in the HAL
_embedded data.

If you're familiar with the benefits of ORM and will use it in your project and you require a fully-fledged
API engine than this API strategy may what you're looking for.