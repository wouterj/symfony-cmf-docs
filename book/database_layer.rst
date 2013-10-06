.. index::
    single: Database Layer
    single: PHPCR-ODM

The Database Layer: PHPCR-ODM
=============================

The Symfony CMF is storage layer agnostic, meaning that it can work with all
sorts of storage layers. By default, the Symfony CMF works with the
`Doctrine PHPCR-ODM`_. In this chapter, you'll learn how to work with the
Doctrine PHPCR-ODM.

.. tip::

    Read more about choosing the correct storage layer in
    :doc:`../cookbook/database/choosing_storage_layer`

PHPCR: A Tree Structure
-----------------------

The Doctrine PHPCR-ODM is an implementation of the `PHP Content Repository`_
(PHPCR), which is a PHP adaption of the `JSR-283 specification`_. The most
important feature of PHPCR is the tree structure to store the data. All data
is stored in items of a tree, called nodes. You can think of this like a file
system, that makes it perfect to use in a CMS.

On top of the tree structure, it also adds features like searching,
versioning, access control and multi-language support.

Doctrine PHPCR-ODM has almost the same API as the `Doctrine ORM`_ and
`Doctrine ODM`_.

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

For Doctrine to be able to persist objects, instead of arrays, you have to
create *metadata*, configuration that tells Doctrine exactly how the ``Task``
class and its properties should be *mapped* to the PHPCR. This metadata can be
specified in a number of different formats including YAML, XML or directly
inside the ``Task`` class via annotations.

Because the PHPCR uses a tree structure, instead of using IDs, a class has a
*parent*.

.. configuration-block::

    .. code-block:: php-annotations

        // src/Acme/TaskBundle/Document/Task.php
        namespace Acme\TaskBundle\Document;

        use Doctrine\ODM\PHPCR\Mapping\Annotations as PHPCR;

        /**
         * @PHPCR\Document()
         */
        class Task
        {
            /**
             * @PHPCR\String()
             */
            protected $description;

            /**
             * @PHPCR\Boolean()
             */
            protected $done = false;

            /**
             * @PHPCR\ParentDocument()
             */
            protected $parent;
        }

    .. code-block:: yaml

        # src/Acme/TaskBundle/Resources/config/doctrine/Task.odm.yml
        Acme\TaskBundle\Document\Task:
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

                <field name="description" type="string" />
                <field name="done" type="boolean" />

                <parent-document name="parent" />
            </document>

        </doctrine-mapping>

After this, you have to create getters and setters for the properties.

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
        $rootTask = ...; // TODO

        $task = new Task();
        $task->setDescription('Finish CMF project');
        $task->setParent($rootTask);

        $documentManager->persist($task);
        $documentManager->flush();

        return new Response('Created task "'.$task->getDescription().'"');
    }

.. seealso::

    You can also check out Doctrine's `Basic Mapping Documentation`_ for all
    details about mapping information. If you use annotations, you'll need to
    prepend all annotations with ``PHPCR\`` (e.g. ``PHPCR\Document(..)``), which is not
    shown in Doctrine's documentation. You'll also need to include the use
    ``use Doctrine\ODM\PHPCR\Mapping\Annotations as PHPCR;`` statement, which
    imports the PHPCR annotations prefix.

.. _`Doctrine PHPCR-ODM`: http://docs.doctrine-project.org/projects/doctrine-phpcr-odm/en/latest/index.html
.. _`PHP Content Repository`: http://phpcr.github.io/
.. _`JSR-283 specifation`: http://jcp.org/en/jsr/detail?id=283
.. _`Doctrine ORM`: http://symfony.com/doc/current/book/doctrine.html
.. _`Doctrine ODM`: http://symfony.com/doc/current/bundles/DoctrineMongoDBBundle/index.html
.. _`Basic Mapping Documentation`:http://docs.doctrine-project.org/projects/doctrine-phpcr-odm/en/latest/reference/annotations-reference.html
