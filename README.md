# EntityAudit Extension for Doctrine2

This extension for Doctrine 2 is inspired by [Hibernate Envers](http://www.jboss.org/envers) and
allows full versioning of entities and their associations.

    Credits to simplethings/EntityAudit for a nice audit bundle for Symfony2 entities. This
    fork was created to allow more extensability from the Event Listeners.

## How does it work?

There are a bunch of different approaches to auditing or versioning of database tables. This extension
creates a mirroring table for each audited entitys table that is suffixed with "_audit". Besides all the columns
of the audited entity there are two additional fields:

* rev - Contains the global revision number generated from a "revisions" table.
* revtype - Contains one of 'INS', 'UPD' or 'DEL' as an information to which type of database operation caused this revision log entry.

The global revision table contains an id, timestamp, username and change comment field.

With this approach it is possible to version an application with its changes to associations at the particular
points in time.

This extension hooks into the SchemaTool generation process so that it will automatically
create the necessary DDL statements for your audited entities.

## Installation (In Symfony2 Application)

Register Bundle in AppKernel.php

    public function registerBundles()
    {
        $bundles = array(
            //...
            new Comstar\EntityAudit\ComstarEntityAuditBundle(),
            //...
        );
        return $bundles;
    }


Autoload

    'Comstar\\EntityAudit' => __DIR__.'/../vendor/bundles/',


Load extension "comstar_entity_audit" and specify the audited entities (yes, that ugly for now!)

    comstar_entity_audit:
        audited_entities:
            - MyBundle\Entity\MyEntity
            - MyBundle\Entity\MyEntity2

Call ./app/console doctrine:schema:update --dump-sql to see the new tables in the update schema queue.

Notice: EntityAudit currently only works with a DBAL Connection and EntityManager named "default".

## Installation (Standalone)

For standalone usage you have to pass the entity class names to be audited to the MetadataFactory
instance and configure the two event listeners.

    <?php
    use Doctrine\ORM\EntityManager;
    use Doctrine\Common\EventManager;
    use Comstar\EntityAudit\AuditConfiguration;
    use Comstar\EntityAudit\AuditManager;

    $auditconfig = new AuditConfiguration();
    $auditconfig->setAuditedEntityClasses(array(
        'Cosmtar\EntityAudit\Tests\ArticleAudit',
        'Comstar\EntityAudit\Tests\UserAudit'
    ));
    $evm = new EventManager();
    $auditManager = new AuditManager($auditconfig);
    $auditManager->registerEvents($evm);

    $config = new \Doctrine\ORM\Configuration();
    // $config ...
    $conn = array();
    $em = EntityManager::create($conn, $config, $evm);

## Usage

Querying the auditing information is done using a `Comstar\EntityAudit\AuditReader` instance.

In Symfony2 the AuditReader is registered as the service "comstar_entityaudit.reader":

    <?php

    class DefaultController extends Controller
    {
        public function indexAction()
        {
            $auditReader = $this->container->get("comstar_entityaudit.reader");
        }
    }

In a standalone application you can create the audit reader from the audit manager:

    <?php

    $auditReader = $auditManager->createAuditReader($entityManager);

### Find entity state at a particular revision

This command also returns the state of the entity at the given revision, even if the last change
to that entity was made in a revision before the given one:

    <?php
    $articleAudit = $auditReader->find('Comstar\EntityAudit\Tests\ArticleAudit', $id = 1, $rev = 10);

Instances created through `AuditReader#find()` are *NOT* injected into the EntityManagers UnitOfWork,
they need to be merged into the EntityManager if it should be reattached to the persistence context
in that old version.

### Find Revision History of an audited entity

    <?php
    $revisions = $auditReader->findRevisions('Comstar\EntityAudit\Tests\ArticleAudit', $id = 1);

A revision has the following API:

    class Revision
    {
        public function getRev();
        public function getTimestamp();
        public function getUsername();
    }

### Find Changed Entities at a specific revision

    <?php
    $changedEntities = $auditReader->findEntitiesChangedAtRevision( 10 );

A changed entity has the API:

    <?php
    class ChangedEntity
    {
        public function getClassName();
        public function getId();
        public function getRevisionType();
        public function getEntity();
    }

### Find Current Revision of an audited Entity

    <?php
    $revision = $auditReader->getCurrentRevision('Comstar\EntityAudit\Tests\ArticleAudit', $id = 3);

## Setting the Current Username

Each revision automatically saves the username that changes it. For this to work you have to set the username.
In the Symfony2 web context the username is automatically set to the one in the current security token.

In a standalone app or Symfony command you have to set the username to a specific value using the `AuditConfiguration`:

    <?php
    // Symfony2 Context
    $container->get('comstar_entityaudit.config')->setCurrentUsername( "beberlei" );

    // Standalone App
    $auditConfig = new \Comstar\EntityAudit\AuditConfiguration();
    $auditConfig->setCurrentUsername( "beberlei" );

## Viewing auditing

A default Symfony2 controller is provided that gives basic viewing capabilities of audited data.

To use the controller, import the routing **(dont forget to secure the prefix you set so that
only appropriate users can get access)**

    # app/config/routing.yml

    comstar_entity_audit:
        resource: "@ComstarEntityAuditBundle/Resources/config/routing.yml"
        prefix: /audit

This provides you with a few different routes:

 * comstar_entity_audit_home -- Displays a paginated list of revisions, their timestamps and the user who performed the revision
 * comstar_entity_audit_viewrevision -- Displays the classes that were modified in a specific revision
 * comstar_entity_audit_viewentity -- Displays the revisions where the specified entity was modified
 * comstar_entity_audit_viewentity_detail -- Displays the data for the specified entity at the specified revision
 * comstar_entity_audit_compare -- Allows you to compare the changes of an entity between 2 revisions

## Extending for use with Gedmo/SoftDeletable

Rather than building in a bundle requirement for Gedmo/DoctrineExtensionsBundle I decided to add instructions here for extending this bundle.
You will first need to create a new bundle in your project and extend the ComstarEntityAuditBundle through the getParent() method. You will
also need to extend the build() method so that we can add a compiler pass to override the comstar_entityaudit.log_revisions_listener with
our own version of the same.

    # Acme/AcmeDemoBundle/AcmeDemoBundle.php
    
    class AcmeDemoBundle extends Bundle
    {
        public function getParent()
        {
            return 'ComstarEntityAuditBundle';
        }
        
        public build(ContainerBuilder $container)
        {
            parent::build($container);
            $container->addCompilerPass(new OverrideServiceCompilerPass());
        }
    }
    
Now we need to create the compiler pass in the DependencyInjection directory of our AcmeDemoBundle.

    # Acme/AcmeDemoBundle/DependencyInjection/Compiler/OverrideCompilerPass.php
    
    namespace Acme\AcmeDemoBundle\DependencyInjection\Compiler;

    use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
    use Symfony\Component\DependencyInjection\ContainerBuilder;

    class OverrideServiceCompilerPass implements CompilerPassInterface
    {
        public function process(ContainerBuilder $container)
        {
            $definition = $container->getDefinition('comstar_entityaudit.log_revisions_listener');
            $definition->setClass('Acme\AcmeDemoBundle\EventListener\LogRevisionsListener');
        }
    }
    
With the service container class changed we now have to create our new event listener and we do that in the EventListener
directory of our Acme/AcmeDemoBundle.  We have to change the use statement to add the Gedmo SoftDeletableListener, update
the getSubscribedEvents() method, and add a postSoftDelete() method.

    # Acme/AcmeDemoBundle/EventListener/LogRevisionsListener.php
    
    use Gedmo\SoftDeleteable\SoftDeleteableListener;
    class LogRevisionsListener implements EventSubscriber
    {
        ...
     
        public function getSubscribedEvents()
        {
            return array(Events::onFlush, Events::postPersists, Events::postUpdate, SoftDeletableListener::POST_SOFT_DELETE);
        }
        
        public function postSoftDelete(LifecycleEventArgs $eventArgs)
        {
            // onFlush was not executed before, initialize everything
            $this->em = $eventArgs->getEntityManager();
            $this->conn = $this->em->getConnection();
            $this->uow = $this->em->getUnitOfWork();
            $this->platform = $this->conn->getDatabasePlatform();
            $this->revisionId = null; // reset revision
            $entity = $eventArgs->getEntity();

            $class = $this->em->getClassMetadata(get_class($entity));
            if (!$this->metadataFactory->isAudited($class->name)) {
                return;
            }

            $entityData = array_merge($this->getOriginalEntityData($entity), $this->uow->getEntityIdentifier($entity));
            $this->saveRevisionEntityData($class, $entityData, 'SDEL');
        }
    }
    
Entities that are soft deleted in the database will now be recorded in the revisions table with a SDEL flag so that we know 
who soft deleted the record and when.
    
## TODOS

* Currently only works with auto-increment databases
* Proper metadata mapping is necessary, allow to disable versioning for fields and associations.
* It does NOT work with Joined-Table-Inheritance (Single Table Inheritance should work, but not tested)
* Many-To-Many assocations are NOT versioned
