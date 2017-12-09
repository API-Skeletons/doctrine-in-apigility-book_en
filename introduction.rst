Introduction
============

The landscape of API strategies grows every day.  In the field of PHP there are
strategies from simple REST-only resources to fully
`Richardson Maturity Model Level 3 <https://martinfowler.com/articles/richardsonMaturityModel.html>`_
API engines.  Apigility falls into the Level 3 category.  As a contributing author of Apigility and the primary
author of Doctrine in Apigility I've designed a method to serve ORM data though an API in a white-gloved approach
to handling the data.

What do I mean by a "white-gloved approach"?  Thanks to the large number of open source libraries for Zend Framework
and Apigility the direct manipulation of data is not usually necessary.  Certainly there will be times you'll want to
add a user to an entity before it is persisted but the majority of validation and filtering is handled inside Apigility.
Doctrine is a powerful ORM and the integration with Apigility is very clean when implemented correctly.  This book will
instruct you how to do just that.


When should you use Doctrine in Apigility?
------------------------------------------

When you have a Doctrine project with properly mapped associations (metadata) between entities Doctrine in Apigility
will give you a powerful head-start toward building an API.  Correct metadata is absolutly core to building an API
with this tool.  To help design your ORM `Skipper <https://skipper18.com>`_ is strongly recommended.
See `Entity Relationship Diagramming with Skipper <skipper>`_

You can use Doctrine in Apigility to serve pieces of your schema by filtering with hydrator strategies or you can
serve your entire schema in an "Open-schema API" where relationships between entities are fully explored in the HAL
``_embedded`` data.

If you're familiar with the benefits of ORM and will use it in your project and you require a fully-fledged
API engine than this API strategy may what you're looking for.


What makes Doctrine in Apigility different?
-------------------------------------------

Because Doctrine is an object relational mapper the relationships between entities are part of the Doctrine metadata.
So when a Doctrine entity is `extracted with a hydrator <hydrators.html>`_ the relationships are pulled from the entity too and included in the
`HAL <hateoas>`_ response as ``_embedded`` data and naturally create a `HATEOAS <hateoas>`_ response.
By using hydration strategies you may include just a
link to the canonical resource or embed the entire resource and in turn embed its resources.

Consider this response to a request for an Artist

.. code-block:: json

    {
        "id": 1,
        "name": "Soft Cell",
        "_embedded": {
            "album": [
                {
                    "id": 1,
                    "name": "Non-Stop Erotic Cabaret",
                    "_embedded": {
                        "artist": {
                            "_links": {
                                "self": "https://api/artist/1"
                            }
                        },
                        "song": {
                            "_links": {
                                "self": "https://api/song?filter%5B0%5D%5Bfield%5D=album&filter%5B0%5D%5Btype%5D=eq&filter%5B0%5D%5Bvalue%5D=1"
                            }
                        }
                    },
                    "_links": {
                        "self": "https://api/album/1"
                    }
                }
            ],
        },
        "_links": {
            "self": "https://api/artist/1"
        }
    }

The ``album`` collection for this artist is returned because a ``CollectionExtract`` hydration strategy was used.
Inside the ``album``, instead of including every ``song``, just a link to the collection filtered by the ``album``
is included.  This way a consumer of the API is directed how to fetch the data when it is not included.

Each album entry has an embedded ``artist`` but this would cause a cyclic reference so an ``EntityLink`` hydrator is
used to just give a link to the ``artist`` from within an ``album``.  This is common when using the ``CollectionExtract`` hydrator.

Within each API response when using Doctrine in Apigility there will never be a dead-end.  Any reference to an entity or collection
will be handled by a hydrator thereby making the API fully implement `HATEOAS <hateoas>`_.


.. role:: raw-html(raw)
   :format: html

.. note::
  Authored by Tom H Anderson of `API Skeletons <https://apiskeletons.com>`_.
  All rights reserved.  :raw-html:`<form style="display: inline" action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top"><input type="hidden" name="cmd" value="_s-xclick"><input type="hidden" name="hosted_button_id" value="WHR95HM3DMYAQ"><input type="image" src="https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif" border="0" name="submit" alt="PayPal - The safer, easier way to pay online!"><img alt="" border="0" src="https://www.paypalobjects.com/en_US/i/scr/pixel.gif" width="1" height="1"></form>`
  if you find this book useful.


:raw-html:`<script async src="https://www.googletagmanager.com/gtag/js?id=UA-64198835-2"></script><script>window.dataLayer = window.dataLayer || [];function gtag(){dataLayer.push(arguments);}gtag('js', new Date());gtag('config', 'UA-64198835-2');</script>`
