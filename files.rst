Files
=====

When a Doctrine resoruce is created through the Apigility UI there are two new files created.  For a resource named Artist these files
are stored in the API module the resource was created in::

  V1\Rest\Artist\ArtistCollection.php
  V1\Rest\Artist\ArtistResource.php
 
For the strategy this book defines it is important you do not try to extend these files or modify them in any way.  They exist because
they could be modified but allow me to repeat:  do not modify these files.  Any reason you may find to modify these files can be 
accomplished through other means such as a Query Provider or subscribing to one of the many events which Doctrine in Apigility throws.


New Files
---------

You will create new files as you build your API.  It is best to put these new files inside the ``V1\`` directory inside your API.
Examples are ``V1\Query\Provider``, ``V1\Query\CreateFilter``, ``V1\Hydrator\Strategy``, ``V1\Hydrator\Filter``.
