Design Summary
==============

This summary gives an overview of a very good design pattern for using Doctrine in Apigility.  Each section will have it's own page and
is included here to give the developer a birds-eye view of the pattern.


Files
-----

Doctrine in Apigility creates two files when a new resource is assigned to an API.  These are

.. code-block:: php

  <?php
  [Resource]Collection.php
  [Resource]Resource.php

As an example for an Artist resource these files full path will be

.. code-block:: php

  <?php
  module/DbApi/src/DbApi/V1/Rest/Artist/ArtistCollection.php
  module/DbApi/src/DbApi/V1/Rest/Artist/ArtistResource.php

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
may read

.. code-block:: php

    <?php
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

Now when the QueryBuilder is ran inside the `DoctrineResource <https://github.com/zfcampus/zf-apigility-doctrine/blob/master/src/Server/Resource/DoctrineResource.php>`_
the id for the user passed to the patch will be appended to the QueryBuilder.  If the id does not belong to the current user then the
QueryBuilder will return no results and a 404 will be thrown to the user trying to edit a record which is not theirs.

More complicated examples **rely on your metadata being complete**.  If your metadata defines joins to and from every join (that is, to an inverse and to a owner entity for every relationship) you can add complicated joins to your Query Provider

.. code-block:: php

    <?php
    $queryBuilder
        ->innerJoin('row.performance', 'performance')
        ->innerJoin('performance.artist', 'artist')
        ->innerJoin('artist.artistGroup', 'artistGroup')
        ->andWhere($queryBuilder->expr()->isMemberOf(':user', 'artistGroup.user'))
        ->setParameter('user', $this->getAuthentication()->getIdentity()->getUser())
        ;


Hydrators
---------

If you're unfamiliar with hydrators
`read Zend Framework's manual on Hydrators <https://framework.zend.com/manual/2.4/en/modules/zend.stdlib.hydrator.html>`_
then
`read Doctrine's manual on Hydrators <https://github.com/doctrine/DoctrineModule/blob/master/docs/hydrator.md>`_
then
`read phpro/zf-doctrine-hydration-module <https://github.com/phpro/zf-doctrine-hydration-module>`_

Hydrators in Doctrine in Apigility are handled by
`phpro/zf-doctrine-hydration-module <https://github.com/phpro/zf-doctrine-hydration-module>`_.
Familiarity with this module is very important to understanding how to extend hydrators without creating special case
hydrators.  Doctrine in Apigility uses an Abstract Factory to create hydrators.

**There should be no need to create your own hydrators.**  That bold statement is true because we're taking a white-gloved approach to
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

Here is the ArtistDefault filter

.. code-block:: php

    <?php
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


Hydrator Strategies
-------------------

The module `API-Skeletons/zf-doctrine-hydrator <https://github.com/API-Skeletons/zf-doctrine-hydrator>`_
provides all the hydrator strategies you will need.  More information on these strategies in `hydration <hydration>`_.


max_depth
---------

Because Doctrine hydrators can extract relationships the default response from a Doctrine in Apigility Resource will include an ``_embedded`` section with the extracted entities and their ``_embedded`` and so on.  **For special cases only** does
`zfcampus/zf-hal <https://github.com/zfcampus/zf-hal>`_ have a `max_depth parameter <https://apigility.org/documentation/modules/zf-hal#key-metadata_map>`_.  This special case is not intended to correct issues with HATEOAS in Doctrine in Apigility.  When you encounter
a cyclic association in Doctrine in Apigility the correct way to handle it is using Hydrator Strategies and Filters.


HATEOAS
-------

Hypertext as the engine of application state is the goal of serving data from Doctrine in Apigility.  Creating a response with no
dead ends.  That is, anytime a reference is made to another entity or collection and that resource is not part of the response there
will be an http self link to that resource.  This way a requesting application can fetch all data associated with a resource
even if it takes more than one request.

A very good example of a practical response of HATEOAS can be found in the README for `API-Skeletons/zf-doctrine-hydrator <https://github.com/API-Skeletons/zf-doctrine-hydrator>`_

The data returned from each resource is the data for that resource' entity.  You should not try to add data to a response which is
not naturally hydrated.  However, there may be times when computed data is required as part of a response.  This is covered in detail in `HATEOAS <hateoas>`_.


An Example
----------

Finally here is an example created by applying the rules listed above and the details listed in this book.  You'll see this performance
has an embedded artist as well as links to every place in the API a client may wish to go to next.  It is not the job of the API to
decide where to go next.  The job of the API is to serve data and give directions for where a client may go

.. code-block:: json

    {
      "performanceDate": "1995-02-21",
      "venue": "Delta Center",
      "city": "Salt Lake City",
      "state": "UT",
      "set1": "Salt Lake City\nFriend Of The Devil\nWang Dang Doodle\nTennessee Jed\nBroken Arrow\nBlack Throated Wind*\nSo Many Roads\nThe Music Never Stopped",
      "set2": "Foolish Heart \u0026gt;\nSamba In The Rain\nTruckin\u0027 \u0026gt;\nI Just Wanna Make Love To You \u0026gt;\nThat Would Be Something \u0026gt;\nDrums \u0026gt;\nSpace \u0026gt;\nVisions Of Johanna \u0026gt;\nSugar Magnolia\n\nEncore: \nLiberty",
      "set3": " ",
      "description": "* Weir on acoustic, First Salt Lake City. First Want To Make Love To You since 10\/8\/84, First Visions 4\/22\/86.  Salt Lake City from Weir\u0027s solo album Heaven Help the Fool\n\nThis show was originally entered with the year 1995 which does not match the year shown in the date above. Please submit a correction or confirmation of the performance date if you are able.",
      "lastUpdateAt": {
        "date": "2016-08-01 12:41:18.000000",
        "timezone_type": 3,
        "timezone": "UTC"
      },
      "createdAt": {
        "date": "2001-07-10 22:15:08.000000",
        "timezone_type": 3,
        "timezone": "UTC"
      },
      "year": 1995,
      "title": "",
      "isApproved": true,
      "id": 2333,
      "performanceGroup": null,
      "_embedded": {
        "performanceCorrection": {
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/performance-correction?filter%5B0%5D%5Bfield%5D=performance\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2333"
            }
          }
        },
        "performanceLink": {
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/performance-link?filter%5B0%5D%5Bfield%5D=performance\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2333"
            }
          }
        },
        "source": {
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/source?filter%5B0%5D%5Bfield%5D=performance\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2333"
            }
          }
        },
        "userPerformance": {
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/user-performance?filter%5B0%5D%5Bfield%5D=performance\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2333"
            }
          }
        },
        "artist": {
          "name": "Grateful Dead",
          "icon": "\/images\/gdskullsmall.gif",
          "createdAt": {
            "date": "2001-07-10 22:15:08.000000",
            "timezone_type": 3,
            "timezone": "UTC"
          },
          "abbreviation": "gd",
          "isTradable": true,
          "description": "",
          "id": 2,
          "artistLink": {},
          "_embedded": {
            "artistAlias": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/artist-alias?filter%5B0%5D%5Bfield%5D=artist\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2"
                }
              }
            },
            "performance": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/performance\/2333?filter%5B0%5D%5Bfield%5D=artist\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2"
                }
              }
            },
            "user": {
              "username": "toma",
              "email": "toma@etree.org",
              "name": "Tom Anderson",
              "createdAt": {
                "date": "1999-09-15 00:00:00.000000",
                "timezone_type": 3,
                "timezone": "UTC"
              },
              "rules": "\u003Cp\u003E\r\n\tWelcome to my site. I hope you find it useful.\u003Cbr \/\u003E\r\n\t\u003Cbr \/\u003E\r\n\tYou can contact the db team at etreedb@googlegroups.com\u003C\/p\u003E\r\n",
              "isActiveTrading": true,
              "city": "San Francisco",
              "state": "CA",
              "postalCode": null,
              "description": "",
              "lastUpdateAt": {
                "date": "2017-05-21 16:24:02.000000",
                "timezone_type": 3,
                "timezone": "UTC"
              },
              "id": 1,
              "_embedded": {
                "source": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/source?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "sourceComment": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/source-comment?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFamily": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-family?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFamilyExtended": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-family-extended?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFeedback": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-feedback?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFeedbackPost": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-feedback?filter%5B0%5D%5Bfield%5D=postUser\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userList": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-list?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userPerformance": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-performance?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "media": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/media?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userWantlist": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/performance\/2333?filter%5B0%5D%5Bfield%5D=wantlistUser\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "role": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/role?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                }
              },
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user\/1"
                }
              }
            },
            "lastUser": {
              "username": "toma",
              "email": "toma@etree.org",
              "name": "Tom Anderson",
              "createdAt": {
                "date": "1999-09-15 00:00:00.000000",
                "timezone_type": 3,
                "timezone": "UTC"
              },
              "rules": "\u003Cp\u003E\r\n\tWelcome to my site. I hope you find it useful.\u003Cbr \/\u003E\r\n\t\u003Cbr \/\u003E\r\n\tYou can contact the db team at etreedb@googlegroups.com\u003C\/p\u003E\r\n",
              "isActiveTrading": true,
              "city": "San Francisco",
              "state": "CA",
              "postalCode": null,
              "description": "",
              "lastUpdateAt": {
                "date": "2017-05-21 16:24:02.000000",
                "timezone_type": 3,
                "timezone": "UTC"
              },
              "id": 1,
              "_embedded": {
                "source": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/source?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "sourceComment": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/source-comment?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFamily": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-family?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFamilyExtended": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-family-extended?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFeedback": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-feedback?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userFeedbackPost": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-feedback?filter%5B0%5D%5Bfield%5D=postUser\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userList": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-list?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userPerformance": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/user-performance?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "media": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/media?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "userWantlist": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/performance\/2333?filter%5B0%5D%5Bfield%5D=wantlistUser\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                },
                "role": {
                  "_links": {
                    "self": {
                      "href": "http:\/\/docker.api.etreedb.org\/role?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=1"
                    }
                  }
                }
              },
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user\/1"
                }
              }
            },
            "artistGroup": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/artist-group?filter%5B0%5D%5Bfield%5D=artist\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=2"
                }
              }
            }
          },
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/artist\/2"
            }
          }
        },
        "user": {
          "username": "aikox2",
          "email": "aiko",
          "name": "aikox2",
          "createdAt": {
            "date": "2004-01-24 18:15:06.000000",
            "timezone_type": 3,
            "timezone": "UTC"
          },
          "rules": "\u003Cp\u003E\r\n\tHey Now,\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tThis list is for my personal reference.\u0026nbsp; I do not \u0026nbsp;trade via postal mail.\u0026nbsp;\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tThis is a work in progress; I have hundreds of shows that have yet to be added to the list.\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tDisclaimer:\u0026nbsp; This list\u0026nbsp;contains\u0026nbsp;shows that are commercially available, as well as shows by artists who do not allow trading.\u0026nbsp; These shows are included for reference\u0026nbsp;only, and are not available for trade.\u0026nbsp; No shows are available for sale.\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tThe \u0026quot;I Was There\u0026quot; list are shows I attended.\u0026nbsp; There are shows on this list that I do not have recordings of.\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tThe \u0026quot;ALL\u0026quot; list is large and thus loads slowly; you may want to select a sub-list from the drop-down menu (i.e.: DVD, GD, WSP, PHIL, ABB, CLAPTON, JAZZ, etc.)\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tA zero disc count means that I have not yet updated that info.\u0026nbsp; If it is on my list, I have the show.\u0026nbsp; If there is no media type designated, it is audio CDR.\u0026nbsp; Audio source may be aud (audience microphone), SBD, FM or RIP (commercial CD backup copy).\u0026nbsp; If no source is listed, it predates my adding this info.\u0026nbsp; All audio is lossless sourced except for a handful of shows that are MP3 sourced and so indicated.\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tAll DVDs are videos.\u0026nbsp; All DVDs are so designated.\u0026nbsp; I do not own any DVD audio.\u0026nbsp; If the Media field does not specify DVD, it is an audio CDR that I have not added the media type to yet.\u0026nbsp; These predate my collecting video.\u0026nbsp; Video source may be aud (audience camera), PRO (multi-camera, not broadcast), TV (proshot for broadcast), WEB (proshot for webstream) or RIP (commercial DVD backup copy).\u0026nbsp;\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tIf a show is listed twice on my\u0026nbsp;a list, that means I have an audio CDR version and a video DVD version, or multiple sources of the same show.\u0026nbsp;\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\tThough many of the DVDs indicate they are PAL, not all PAL DVDs have been so designated.\u0026nbsp;\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\taikox2\u003C\/p\u003E\r\n\u003Cp\u003E\r\n\t\u0026nbsp;\u003C\/p\u003E\r\n",
          "isActiveTrading": true,
          "city": "",
          "state": "NC",
          "postalCode": null,
          "description": null,
          "lastUpdateAt": {
            "date": "2017-11-11 21:01:42.000000",
            "timezone_type": 3,
            "timezone": "UTC"
          },
          "id": 78828,
          "_embedded": {
            "source": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/source?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "sourceComment": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/source-comment?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userFamily": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user-family?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userFamilyExtended": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user-family-extended?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userFeedback": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user-feedback?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userFeedbackPost": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user-feedback?filter%5B0%5D%5Bfield%5D=postUser\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userList": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user-list?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userPerformance": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/user-performance?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "media": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/media?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "userWantlist": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/performance\/2333?filter%5B0%5D%5Bfield%5D=wantlistUser\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            },
            "role": {
              "_links": {
                "self": {
                  "href": "http:\/\/docker.api.etreedb.org\/role?filter%5B0%5D%5Bfield%5D=user\u0026filter%5B0%5D%5Btype%5D=eq\u0026filter%5B0%5D%5Bvalue%5D=78828"
                }
              }
            }
          },
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/user\/78828"
            }
          }
        },
        "wantlistUser": {
          "_links": {
            "self": {
              "href": "http:\/\/docker.api.etreedb.org\/user?filter%5B0%5D%5Bfield%5D=userWantlist\u0026filter%5B0%5D%5Btype%5D=ismemberof\u0026filter%5B0%5D%5Bvalue%5D=2333"
            }
          }
        }
      },
      "_links": {
        "self": {
          "href": "http:\/\/docker.api.etreedb.org\/performance\/2333"
        }
      }
    }

.. role:: raw-html(raw)
   :format: html

.. note::
  Authored by Tom H Anderson of `API Skeletons <https://apiskeletons.com>`_.
  All rights reserved.  :raw-html:`<form style="display: inline" action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top"><input type="hidden" name="cmd" value="_s-xclick"><input type="hidden" name="hosted_button_id" value="WHR95HM3DMYAQ"><input type="image" src="https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif" border="0" name="submit" alt="PayPal - The safer, easier way to pay online!"><img alt="" border="0" src="https://www.paypalobjects.com/en_US/i/scr/pixel.gif" width="1" height="1"></form>`
  if you find this book useful.


:raw-html:`<script async src="https://www.googletagmanager.com/gtag/js?id=UA-64198835-2"></script><script>window.dataLayer = window.dataLayer || [];function gtag(){dataLayer.push(arguments);}gtag('js', new Date());gtag('config', 'UA-64198835-2');</script>`
