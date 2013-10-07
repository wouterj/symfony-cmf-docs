.. index::
    single: Database Layer
    single: PHPCR-ODM

The Database Layer: PHPCR-ODM
=============================

The Symfony CMF is storage layer agnostic, meaning that it can work with many
storage layers. By default, the Symfony CMF works with the `Doctrine PHPCR-ODM`_.
In this chapter, you'll learn how to work with the Doctrine PHPCR-ODM.

.. tip::

    Read more about choosing the correct storage layer in
    :doc:`../cookbook/database/choosing_storage_layer`

PHPCR: A Tree Structure
-----------------------

The Doctrine PHPCR-ODM is a doctrine object-mapper on top of the
`PHP Content Repository`_ (PHPCR), which is a PHP adaption of the
`JSR-283 specification`_. The most important feature of PHPCR is the tree
structure to store the data. All data is stored in items of a tree, called
nodes. You can think of this like a file system, that makes it perfect to use
in a CMS.

On top of the tree structure, PHPCR also adds features like searching,
versioning and access control.

Doctrine PHPCR-ODM has the same API as the other Doctrine libraries, like the
`Doctrine ORM`_. The Doctrine PHPCR-ODM adds another great feature to PHPCR:
Multilanguage support.

.. sidebar:: PHPCR Implementations

    In order to let the Doctrine PHPCR-ODM communicate with the PHPCR, a PHPCR
    implentation is needed. `Jackalope`_ is a famous PHPCR implementation,
    which can work with `Apache Jackrabbit`_ (with the `jackalope-jackrabbit`_
    package) and with Doctrine DBAL (providing support for postgres, sqlite
    and mysql) with the `jackalope-doctrine-dbal`_ package.

A Simple Example: A Task
------------------------

The easiest way to get started with the PHPCR-ODM is to see it in action. In
this section, you are going to create a ``Task`` object and learn how to
persist it.

Creating a Document Class
~~~~~~~~~~~~~~~~~~~~~~~~~

Without tinking about Doctrine or PHPCR-ODM, you can create a ``Task`` object
in PHP::

    // src/Acme/TaskBundle/Document/Task.php
    namespace Acme\TaskBundle\Document;

    class Task
    {
        protected $description;

        protected $done = false;
    }

The class - often called a "document" in the ODM - is a simple PHP class which
can't be persisted yet.

Add Mapping Information
~~~~~~~~~~~~~~~~~~~~~~~

Doctrine allows you to work with PHPCR in a much more interesting way than
just fetching data back and forth as an array. Instead, Doctrine allows you to
persist entire objects to PHPCR and fetch entire *objects* out of PHPCR.
This works by mapping a PHP class and its properties to the PHPCR tree.

For Doctrine to be able to do this, you just have to create "metadata", or
configuration that tells Doctrine exactly how the ``Task`` document and its
properties should be *mapped* to PHPCR. This metadata can be specified in a
number of different formats including YAML, XML or directly inside the ``Task``
class via annotations:

.. configuration-block::

    .. code-block:: php-annotations

 TODO FIX ANNOTATION COMMENTS

        // src/Acme/TaskBundle/Document/Task.php
        namespace Acme\TaskBundle\Document;

        use Doctrine\ODM\PHPCR\Mapping\Annotations as PHPCR;

        /**
         * @PHPCR\Document()
         *
        class Task
        {
            /**
             * @PHPCR\Id()
             *
            protected $id;

            /**
             * @PHPCR\String()
             *
            protected $description;

            /**
             * @PHPCR\Boolean()
             *
            protected $done = false;

            /**
             * @PHPCR\ParentDocument()
             *
            protected $parent;
        }

    .. code-block:: yaml

        # src/Acme/TaskBundle/Resources/config/doctrine/Task.odm.yml
        Acme\TaskBundle\Document\Task:
            id: id

            fields:
                description: string
                done: boolean

            parent_document: parent

    .. code-block:: xml

        <!-- src/Acme/TaskBundle/Resources/config/doctrine/Task.odm.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <doctrine-mapping
            xmlns="http://doctrine-project.org/schemas/phpcr-odm/phpcr-mapping"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://doctrine-project.org/schemas/phpcr-odm/phpcr-mapping
            https://github.com/doctrine/phpcr-odm/raw/master/doctrine-phpcr-odm-mapping.xsd"
            >

            <document name="Acme\TaskBundle\Document\Task">

                <id name="id" />

                <field name="description" type="string" />
                <field name="done" type="boolean" />

                <parent-document name="parent" />
            </document>

        </doctrine-mapping>

After this, you have to create getters and setters for the properties.

.. note::

    This Document uses the parent document and a node name to determine its
    position in the tree. Because there isn't any name set, it is generated
    automatically. If you want to use a specific node name, such as a
    sluggified version of the title, you need to add a property mapped as
    ``Nodename``.

    A Document must have an id property. This represents the full path (parent
    + name) of the Document. This will be set by Doctrine by default and it is
    not recommend to use the id to determine the location of a Document.

    For more information about identifier generation strategies, refer to the
    `doctrine documentation`_

.. seealso::

    You can also check out Doctrine's `Basic Mapping Documentation`_ for all
    details about mapping information. If you use annotations, you'll need to
    prepend all annotations with ``PHPCR\`` (e.g. ``PHPCR\Document(..)``), which is not
    shown in Doctrine's documentation. You'll also need to include the use
    ``use Doctrine\ODM\PHPCR\Mapping\Annotations as PHPCR;`` statement, which
    imports the PHPCR annotations prefix.

Persisting Documents to PHPCR
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that you have a mapped ``Task`` document, complete with getter and setter
methods, you're ready to persist data to PHPCR. From inside a controller,
this is pretty easy, add the following method to the ``DefaultController`` of the
AcmeTaskBundle::

    // src/Acme/TaskBundle/Controller/DefaultController.php

    // ...
    use Acme\TaskBundle\Document\Task;
    use Symfony\Component\HttpFoundation\Response;

    // ...
    public function createAction()
    {
        $documentManager = $this->get('doctrine_phpcr')->getManager();

        // get the root node for tasks
        $rootTask = $documentManager->find(null, '/tasks');

        // create a new task (just like an object)
        $task = new Task();
        $task->setDescription('Finish CMF project');
        $task->setParent($rootTask);

        // persist the task into the db
        $documentManager->persist($task);

        // save the actions
        $documentManager->flush();

        return new Response('Created task "'.$task->getDescription().'"');
    }

.. note::

    The example uses the ``find`` method of the document manager to get the
    documents. You can also use Repositories, but this requires to know the
    class of a node in the tree. Unless you want to do specific things, it's
    recommend to use the document manager's ``find`` and ``findAll`` methods
    to find documents.

.. sidebar:: Creating the Root Node

    You'll wonder where the root node ``/tasks`` is coming from. This is the
    root node of the tasks and it doesn't exists yet. To create this root
    node, it is recommend to use :ref:`Repository Initializers
    <phpcr-odm-repository-initializers>`.  These will be executed when
    executing ``doctrine:phpcr:repository:init`` and should create all
    required root nodes.

.. _`Doctrine PHPCR-ODM`: http://docs.doctrine-project.org/projects/doctrine-phpcr-odm/en/latest/index.html
.. _`PHP Content Repository`: http://phpcr.github.io/
.. _`JSR-283 specifation`: http://jcp.org/en/jsr/detail?id=283
.. _`Doctrine ORM`: http://symfony.com/doc/current/book/doctrine.html
.. _`Jackalope`: http://jackalope.github.io/
.. _`Apache Jackrabbit`: http://jackrabbit.apache.org/
.. _`jackalope-jackrabbit`: https://github.com/jackalope/jackalope-jackrabbit
.. _`jackalope-doctrine-dbal`: https://github.com/jackalope/jackalope-doctrine-dbal
.. _`doctrine documentation`: http://docs.doctrine-project.org/projects/doctrine-phpcr-odm/en/latest/reference/basic-mapping.html#basicmapping-identifier-generation-strategies
.. _`Basic Mapping Documentation`: http://docs.doctrine-project.org/projects/doctrine-phpcr-odm/en/latest/reference/annotations-reference.html
