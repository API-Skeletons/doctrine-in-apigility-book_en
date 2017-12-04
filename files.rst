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


.. role:: raw-html(raw)
   :format: html

.. note::
  Authored by Tom H Anderson of `API Skeletons <https://apiskeletons.com>`_.
  All rights reserved.  :raw-html:`<form style="display: inline" action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top"><input type="hidden" name="cmd" value="_s-xclick"><input type="hidden" name="hosted_button_id" value="WHR95HM3DMYAQ"><input type="image" src="https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif" border="0" name="submit" alt="PayPal - The safer, easier way to pay online!"><img alt="" border="0" src="https://www.paypalobjects.com/en_US/i/scr/pixel.gif" width="1" height="1"></form>`
  if you find this book useful.


:raw-html:`<script async src="https://www.googletagmanager.com/gtag/js?id=UA-64198835-2"></script><script>window.dataLayer = window.dataLayer || [];function gtag(){dataLayer.push(arguments);}gtag('js', new Date());gtag('config', 'UA-64198835-2');</script>`
