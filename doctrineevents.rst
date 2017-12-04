Doctrine Events
===============

This purely Doctrine subject has a good reason to be here.  Often when a new entity is POSTed to a resource there will be default values
you want to save to the new entity.  For this reason you should use Doctrine events to catch lifecycle events to an entity and 
modify the entity as needed.

If you were to create an API where you tried to use a Query Create Filter, for example, to add default data then if you ever needed to 
create the same entity within the application the same defaults would not be applied.  Doctrine events act globally in an application.
You should use Doctrine events as often as needed.

Within a Doctrine event you have access to the Object Manager.  This in turn has access to the Repositories 
(see `API-Skeletons/zf-doctrine-repository <https://github.com/API-Skeletons/zf-doctrine-repository>`_) which may have access to plugins 
if you use the referenced module, as you should.  


Doctrine Event Subscriber Manager Factory pattern
-------------------------------------------------

Here I will give an example of a good pattern for creating and subscribing event subscribers.

First the configuration, done with a ConfigProvider

.. code-block:: php

    <?php
    namespace Db;

    use Zend\ServiceManager\Factory\InvokableFactory;

    final class ConfigProvider
    {
        /**
         * Loaded at the time the DoctrineEventSubscriber is created
         */
        public function getDoctrineEventSubscriberConfig()
        {
            return [
                'factories' => [
                    EventSubscriber\Doctrine\Artist::class
                        => InvokableFactory::class,
                ],
            ];
        }
        
        public function getDependencyConfig()
        {
            return [
                'factories' => [
                    EventSubscriber\Doctrine\DoctrineEventSubscriberManager::class =>
                        EventSubscriber\Doctrine\DoctrineEventSubscriberManagerFactory::class,
                ],
            ];
        }        
        ...
    }

This config is loaded by the factory into the manager

.. code-block:: php

    <?php
    namespace Db\EventSubscriber\Doctrine;

    use Interop\Container\ContainerInterface;
    use Db\ConfigProvider;

    final class DoctrineEventSubscriberManagerFactory
    {
        public function __invoke(
            ContainerInterface $container,
            $requestedName,
            array $options = null
        ) {
            $objectManager = $container->get('doctrine.entitymanager.orm_default');
            $configProvider = new ConfigProvider();

            $instance = new $requestedName($configProvider->getDoctrineEventSubscriberConfig());
            $instance->setServiceLocator($container);
            $instance->setObjectManager($objectManager);

            return $instance;
        }
    }

This event subscriber manager factory will create an event subscriber manager for your event subscribers.

.. code-block:: php

    <?php
    namespace Db\EventSubscriber\Doctrine;

    use Interop\Container\ContainerInterface;
    use Zend\ServiceManager\ServiceManager as ZendServiceManager;
    use DoctrineModule\Persistence\ObjectManagerAwareInterface;
    use DoctrineModule\Persistence\ProvidesObjectManager;
    use Db\ConfigProvider;

    final class DoctrineEventSubscriberManager extends ZendServiceManager implements
        ObjectManagerAwareInterface
    {
        use ProvidesObjectManager;

        private $serviceLocator;

        public function getServiceLocator()
        {
            return $this->serviceLocator;
        }

        public function setServiceLocator(ContainerInterface $serviceLocator)
        {
            $this->serviceLocator = $serviceLocator;

            return $this;
        }

        public function subscribe()
        {
            foreach ((array) $this->factories as $name => $squishedname) {
                $instance = $this->get($name);
                $instance->setAuthentication($this->getServiceLocator()->get('authentication'));

                $this->getObjectManager()->getEventManager()->addEventSubscriber($instance);
            }
        }
    }

The subscribe() function, as it creates each event subscriber, injects the zf-mvc-auth authentication so an abstract is used.

.. code-block:: php

    <?php
    namespace Db\EventSubscriber\Doctrine;

    use Zend\Authentication\AuthenticationService;

    abstract class AbstractEventSubscriber
    {
        private $authentication;

        public function getAuthentication()
        {
            return $this->authentication;
        }

        public function setAuthentication(AuthenticationService $authentication)
        {
            $this->authentication = $authentication;
        }
    }

Next you'll create your event subscribers with only one subcriber per entity.

.. code-block:: php

    <?php
    namespace Db\EventSubscriber\Doctrine;

    use Datetime;
    use Doctrine\Common\Persistence\Event\LifecycleEventArgs;
    use Doctrine\Common\EventSubscriber;
    use Doctrine\ORM\Events;
    use Db\Entity;

    final class Artist extends AbstractEventSubscriber implements
        EventSubscriber
    {
        public function getSubscribedEvents()
        {
            return [
                Events::prePersist,
                Events::preUpdate,
                Events::postUpdate,
            ];
        }

        public function prePersist(LifecycleEventArgs $args)
        {
            if (! $args->getObject() instanceof Entity\Artist) {
                return;
            }

            $args->getObject()->setUser($this->getAuthentication()->getIdentity()->getUser());
            $args->getObject()->setLastUser($this->getAuthentication()->getIdentity()->getUser());
            $args->getObject()->setCreatedAt(new Datetime());
        }

        public function preUpdate(LifecycleEventArgs $args)
        {
            if (! $args->getObject() instanceof Entity\Artist) {
                return;
            }

            $args->getObject()->setLastUser($this->getAuthentication()->getIdentity()->getUser());
        }

        public function postUpdate(LifecycleEventArgs $args)
        {
            if (! $args->getObject() instanceof Entity\Artist) {
                return;
            }

            $args->getObjectManager()
                ->getRepository(Entity\Artist::class)
                ->enqueueIndexArtist($args->getObject());
        }
    }

Notice the last call to the `enqueueIndexArtist`.  Running domain code from event subscribers if you follow the guidelines in
`How repositories in Doctrine replace the "model" layer <http://blog.tomhanderson.com/2015/10/how-repositories-in-doctrine-replace.html?q=model>`_

Finally bootstrap the module to load all the event subscribers and subscribe them:

.. code-block:: php

    <?php
    public function onBootstrap(EventInterface $e)
    {
        $sm = $e->getApplication()->getServiceManager();

        $doctrineEventSubscriberManager =
            $sm->get(EventSubscriber\Doctrine\DoctrineEventSubscriberManager::class);
        $doctrineEventSubscriberManager->subscribe();
    }


.. role:: raw-html(raw)
   :format: html

.. note::
  Authored by Tom H Anderson of `API Skeletons <https://apiskeletons.com>`_.
  All rights reserved.  :raw-html:`<form style="display: inline" action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top"><input type="hidden" name="cmd" value="_s-xclick"><input type="hidden" name="hosted_button_id" value="WHR95HM3DMYAQ"><input type="image" src="https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif" border="0" name="submit" alt="PayPal - The safer, easier way to pay online!"><img alt="" border="0" src="https://www.paypalobjects.com/en_US/i/scr/pixel.gif" width="1" height="1"></form>`
  if you find this book useful.
