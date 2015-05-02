.. index::
   single: Rotte; Caricatore di rotte personalizzato

Creare un caricatore di rotte personalizzato
============================================

Un caricatore di rotte personalizzato permette di aggiungere rotte a un'applicazione, senza 
per esempio doverle includere un file Yaml. Questo può essere comodo quando si 
ha un bundle, ma non si vogliono aggiungere manualmente le rotte per quel bundle
a ``app/config/routing.yml``. Questo può essere in special modo importante 
quando si vuole rendere il bundle riusabile o quando lo si è reso open source, in quanto
questo rallenterebbe il processo di installazione e lo renderebbe più incline agli errori.

Alternativamente, si può usare un caricatore personalizzato di rotte, quando si vuole che le proprie 
rotte siano generate o trovate automaticamente in base a qualche convenzione o schema.
Un esempio è `FOSRestBundle`_, nel quale le rotte sono generate in base ai nomi
dei metodi Action nei controllori.

.. note::

    Ci sono molti bundle che usano i propri caticatori per gestire 
    casi come quello descritto qui sopra, ad esempio
    `FOSRestBundle`_, `JMSI18nRoutingBundle`_, `KnpRadBundle`_ e `SonataAdminBundle`_.

Caricare le rotte
-----------------

Le rotte in una applicazione Symfony vengono caricate
da :class:`Symfony\\Bundle\\FrameworkBundle\\Routing\\DelegatingLoader`.
Questo caticatore usa svariati altri caricatori (delegati) per caricare risorse di 
tipi differenti, per esempio file Yaml o annotazioni ``@Route`` e ``@Method`` 
nei file dei controllori. I caricatori specializzati implementano 
:class:`Symfony\\Component\\Config\\Loader\\LoaderInterface`
e quindi hanno due metodi importanti:
:method:`Symfony\\Component\\Config\\Loader\\LoaderInterface::supports`
e :method:`Symfony\\Component\\Config\\Loader\\LoaderInterface::load`.

Osserviamo queste linee del file ``routing.yml`` in AcmeDemoBundle della Standard
Edition:

.. code-block:: yaml

    # src/Acme/DemoBundle/Resources/config/routing.yml
    _demo:
        resource: "@AcmeDemoBundle/Controller/DemoController.php"
        type:     annotation
        prefix:   /demo

Quando il caricatore principale li analizza, interpella ogni caricatore delegato e ne chiama
il metodo :method:`Symfony\\Component\\Config\\Loader\\LoaderInterface::supports`
con la risorsa data (``@AcmeDemoBundle/Controller/DemoController.php``) e
il tipo (``annotation``) come parametri. Quando uno dei caricatori restituisce ``true``,
il suo metodo :method:`Symfony\\Component\\Config\\Loader\\LoaderInterface::load` 
verrà chiamato e dovrà restituire una classe di tipo :class:`Symfony\\Component\\Routing\\RouteCollection`
contenente oggetti di tipo :class:`Symfony\\Component\\Routing\\Route`

Creare un caricatore 
--------------------

Per caricare rotte da qualche fonte personalizzata (ossia da qualcosa di diverso da annotazioni, 
file Yaml o XML), occorre creare un caricatore. Questo caricatore
deve implementare l'interfaccia :class:`Symfony\\Component\\Config\\Loader\\LoaderInterface`.

Il caricatore d'esempio qui sotto supporta il caricamento di risorse di tipo
``extra``. Il tipo ``extra`` non è importante, si può inventare qualsiasi tiipo di risorsa
si voglia. Il nome della risorsa non viene concretamente utilizzato nell'esempio::

    namespace Acme\DemoBundle\Routing;

    use Symfony\Component\Config\Loader\LoaderInterface;
    use Symfony\Component\Config\Loader\LoaderResolverInterface;
    use Symfony\Component\Routing\Route;
    use Symfony\Component\Routing\RouteCollection;

    class ExtraLoader implements LoaderInterface
    {
        private $loaded = false;

        public function load($resource, $type = null)
        {
            if (true === $this->loaded) {
                throw new \RuntimeException('Non aggiungere due volte il caricatore "extra"');
            }

            $routes = new RouteCollection();

            // prepara una nuova rotta
            $path = '/extra/{parameter}';
            $defaults = array(
                '_controller' => 'AcmeDemoBundle:Demo:extra',
            );
            $requirements = array(
                'parameter' => '\d+',
            );
            $route = new Route($path, $defaults, $requirements);

            // aggiunge la nuova rotta all'insieme di rotte:
            $routeName = 'extraRoute';
            $routes->add($routeName, $route);

            $this->loaded = true;

            return $routes;
        }

        public function supports($resource, $type = null)
        {
            return 'extra' === $type;
        }

        public function getResolver()
        {
            // necessario, ma può essere vuoto, a meno che non si vogliano caricare altre risorse
            // se lo si, è più facile usando la classe Loader base (vedere sotto)
        }

        public function setResolver(LoaderResolverInterface $resolver)
        {
            // come sopra
        }
    }

.. note::

    Accertarsi che il controllore specifito esista realmente.

Adesso definire un servizio per l'``ExtraLoader``:

.. configuration-block::

    .. code-block:: yaml

        services:
            acme_demo.routing_loader:
                class: Acme\DemoBundle\Routing\ExtraLoader
                tags:
                    - { name: routing.loader }

    .. code-block:: xml

        <?xml version="1.0" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="acme_demo.routing_loader" class="Acme\DemoBundle\Routing\ExtraLoader">
                    <tag name="routing.loader" />
                </service>
            </services>
        </container>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Definition;

        $container
            ->setDefinition(
                'acme_demo.routing_loader',
                new Definition('Acme\DemoBundle\Routing\ExtraLoader')
            )
            ->addTag('routing.loader')
        ;

Si noti il tag ``routing.loader``. Tutti i servizi con questo tag saranno marcati
come potenziali caricatori di rotte e aggiunti come router specializzati alla classe
:class:`Symfony\\Bundle\\FrameworkBundle\\Routing\\DelegatingLoader`.

Usare un Custom Loader
~~~~~~~~~~~~~~~~~~~~~~

Se non è stato fatto niente di diverso, il caricatore di rotte *non* sarà interpellato.
Occorre solo aggiungere qualche riga extra alla configurazione del router.

.. configuration-block::

    .. code-block:: yaml

        # app/config/routing.yml
        AcmeDemoBundle_Extra:
            resource: .
            type: extra

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <routes xmlns="http://symfony.com/schema/routing"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/routing http://symfony.com/schema/routing/routing-1.0.xsd">

            <import resource="." type="extra" />
        </routes>

    .. code-block:: php

        // app/config/routing.php
        use Symfony\Component\Routing\RouteCollection;

        $collection = new RouteCollection();
        $collection->addCollection($loader->import('.', 'extra'));

        return $collection;

La parte importante qui è la chiave ``type``. Il suo valore deve essere "extra".
Questo è il tipo supportato dal nostro ``ExtraLoader`` e questo farà sì che venga chiamato il suo 
metodo ``load()`` . La chiave ``resource`` è ininfluente per ``ExtraLoader``,
quindi la impostiamo a ".".

.. note::

    Le rotte definite usando dei caricatori di rotte personalizzati vengono automaticamente messe in cache 
    dal framework. Quindi, ogni volta che si cambia qualcosa nella 
    classe del caricatore, non dimenticare di cancellare la cache.

Caricatori più avanzati
-----------------------

Nella maggior parte dei casi è meglio non implementare direttamente
:class:`Symfony\\Component\\Config\\Loader\\LoaderInterface`,
ma estendere la classe :class:`Symfony\\Component\\Config\\Loader\\Loader`.
Questa classe sa come usare un :class:`Symfony\\Component\\Config\\Loader\\LoaderResolver`
per caricare le risorse di routing secondarie.

Ovviamente è ancora necessario implementare i metodi
:method:`Symfony\\Component\\Config\\Loader\\LoaderInterface::supports`
e :method:`Symfony\\Component\\Config\\Loader\\LoaderInterface::load`.
Ogni volta che si carica un'altra risorsa, per esempio un file di configurazione di rotte in 
Yaml, si può richiamare il metodo
:method:`Symfony\\Component\\Config\\Loader\\Loader::import` ::

    namespace Acme\DemoBundle\Routing;

    use Symfony\Component\Config\Loader\Loader;
    use Symfony\Component\Routing\RouteCollection;

    class AdvancedLoader extends Loader
    {
        public function load($resource, $type = null)
        {
            $collection = new RouteCollection();

            $resource = '@AcmeDemoBundle/Resources/config/import_routing.yml';
            $type = 'yaml';

            $importedRoutes = $this->import($resource, $type);

            $collection->addCollection($importedRoutes);

            return $collection;
        }

        public function supports($resource, $type = null)
        {
            return $type === 'advanced_extra';
        }
    }

.. note::

    Il nome della risorsa e il tipo della configurazione di routing importata
    possono essere qualsiasi cosa che sia normalmente supportata dal caricatore di 
    configurazioni di routing (Yaml, XML, PHP, annotation, ecc.).

.. _`FOSRestBundle`: https://github.com/FriendsOfSymfony/FOSRestBundle
.. _`JMSI18nRoutingBundle`: https://github.com/schmittjoh/JMSI18nRoutingBundle
.. _`KnpRadBundle`: https://github.com/KnpLabs/KnpRadBundle
.. _`SonataAdminBundle`: https://github.com/sonata-project/SonataAdminBundle
