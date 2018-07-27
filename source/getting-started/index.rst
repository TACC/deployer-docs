Getting Started
---------------

This Getting Started guide will conver installing the Deployer CLI as well as some initial basic concepts needed before
moving to the `User Guide <../users/index.html>`_.


Installation
============

To install the Deployer CLI, the only dependency is Docker. If you do not have Docker installed, see the
`Official Docs <https://docs.docker.com/install/>`_ for installation instructions for your platform.

Once Docker is installed, change into a temporary directory and run:

.. code-block:: bash

    $ docker run -v $(pwd):/conf tacc/deployer --setup && mv deployer /usr/local/bin/deployer

The first step pulls down the latest stable version of the Deployer Docker image and installs a small alias script,
``deployer``, in the current working directory. The second stop is optional but recommended so that the ``deployer``
script is on ``$PATH`` and not left in the temporary directory.

.. note:: Different versions of Deployer can be installed by specifying a TAG on the ``tacc/deployer`` image.


With the alias installed, simply issue commands directly to the script. For example, validate your installation by
executing:

.. code-block:: bash

    $ deployer --help

which should display the help.


Basic Usage
===========

The general format for executing commands via the the Deployer CLI is:

.. code-block:: bash

    $ deployer <command> -p <project> -i <instance> -t <tenant> -a <action> -d <deployment_file>

For any kind of deployment activity, one will use the ``execute`` command to the Deployer CLI, which is its default
value and can be omitted. However, the Deployer CLI recognizes a few other informational commands. For example, we can
get the version of the installed Deployer using the ``version`` command which requires no other arguments:

.. code-block:: bash

  $ deployer version
  TACC Deployer
  Version: 0.1-rc1

The arguments ``project``, ``instance``, etc, are defined in the following table
and are covered in more detail in the `Basic Concepts`_ section below.

CLI Arguments:

+----------------------------------------+--------------------------------------------------------------------------+
| -h, --help                             | show help message and exit                                               |
+----------------------------------------+--------------------------------------------------------------------------+
| -p PROJECT, --project PROJECT          | Software project to deploy such as JupyterHub or Abaco.                  |
|                                        |                                                                          |
|                                        | Overrides that specified in the deployment file.                         |
+----------------------------------------+--------------------------------------------------------------------------+
| -i INSTANCE, --instance INSTANCE       | Instance to deploy, such as 'prod' or 'dev'.                             |
|                                        |                                                                          |
|                                        | Overrides that specified in the deployment file.                         |
+----------------------------------------+--------------------------------------------------------------------------+
| -t TENANT, --tenant TENANT             | Tenant to deploy, such as 'SD2E' or 'designsafe'.                        |
|                                        |                                                                          |
|                                        | Overrides that specified in the deployment file.                         |
+----------------------------------------+--------------------------------------------------------------------------+
| -a ACTION, --action ACTION             | Action to take, such as 'deploy' or 'update'                             |
|                                        | Must be a valid action for the project specified.                        |
|                                        |                                                                          |
+----------------------------------------+--------------------------------------------------------------------------+
| -s SERVERS, --servers SERVERS          | Relative path to servers file, in the YAML format,                       |
|                                        | of servers to target. This file should be in the                         |
|                                        | current working directory or a sub directory therein.                    |
+----------------------------------------+--------------------------------------------------------------------------+
| -c CONFIGS, --configs CONFIGS          | Relative path to config file, in the YAML format,                        |
|                                        | of config to use. This file should be in the                             |
|                                        | current working directory or a sub directory therein.                    |
+----------------------------------------+--------------------------------------------------------------------------+
| -z PASSWORDS, --passwords PASSWORDS    | Relative path to passwords file, in the YAML format,                     |
|                                        | of passwords to use. This file should be in the                          |
|                                        | current working directory or a sub directory therein.                    |
+----------------------------------------+--------------------------------------------------------------------------+
| -e EXTRAS, --extras EXTRAS             | Relative path to a directory of extra files needed for                   |
|                                        | the action. This path should be in the current working                   |
|                                        | directory or a sub directory therein.                                    |
+----------------------------------------+--------------------------------------------------------------------------+
| -o OVERRIDES, --overrides OVERRIDES    | String of config overrides; should have the form                         |
|                                        | key1=value1&key2=value2.. These values will override                     |
|                                        | values supplied in the config file at the                                |
|                                        | project/instance/tenant level specified through those                    |
|                                        | corresponding command line arguments or the deployment file              |
+----------------------------------------+--------------------------------------------------------------------------+
| -v, --verbose                          | The value of the actor's state at the start of the execution.            |
+----------------------------------------+--------------------------------------------------------------------------+
| -k, --keep_tempfiles                   | Whether to keep the temporary Ansible files generated to                 |
|                                        | execute playbooks                                                        |
+----------------------------------------+--------------------------------------------------------------------------+
| -vv, --very_verbose                    | Display very verbose output.                                             |
+----------------------------------------+--------------------------------------------------------------------------+


Basic Concepts
==============

The following concepts are important to understand before using Deployer.

Projects, Instances and Tenants
+++++++++++++++++++++++++++++++

The notions of ``project``, ``instance`` and ``tenant`` are fundamental to Deployer's approach to managing deployments.
A ``project`` is one of a set of systems Deployer knows how to manage, and will eventually include the TACC JupyterHub,
Abaco and Agave projects. When working with Deployer CLI, projects are referenced by a project id. To see information
about what projects are supported in an existing Deployer installation, including their id's, use the
``list_projects`` command:

.. code-block:: bash

  $ deployer list_projects
    Available Projects:

    TACC Integrated JupyterHub
    **************************
    id: jupyterhub
    Description: Customized JupyterHub enabling deeper integration and
    ease of use of TACC resources from within running notebooks.
    Docs: http://cic-deployer.readthedocs.io/en/latest/users/projects.html#jupyterhub


The values for ``instance`` and ``tenant`` can be chosen by the operations team to best organize
their infrastructure and configuration. One approach is to use ``instance`` values to distinguish physically isolated
systems such as "development" and "production" and to use ``tenant`` values to distinguish logically separated aspects of
systems (such as the DesignSafe tenant for JupyterHub or Abaco).

Actions
+++++++

Actions define what procedure should be taken on the deployment. Actions are defined on a project by project basis,
though some standard actions such as ``deploy`` are available for all projects.

To see which actions are available for a given project, use the ``list_actions`` command, specifying a project; e.g.:

.. code-block:: bash

  $ deployer -p jupyterhub list_actions
    Available actions: ['deploy']

Note that the value to the project argument must be a project id, as output by the ``list_projects`` command.


Deployment Files
++++++++++++++++

In order to use Deployer, the operator will need to supply some deployment files describing the infrastructure to
deploy onto and the configuration for the projects to deploy. At a minimum, this includes configuration file(s) and
server file(s). Details on how to write these files are provided in the `User Guide <../users/index.html>`_. We
encourage teams to keep these files in a version control system and check them out on each machine that will run
Deployer. For example, the CIC team stores
its own deployment files in a `bitbucket repository <https://bitbucket.org/tacc-cic/cic-deployments>`_.

Hierarchical Organization of Properties
+++++++++++++++++++++++++++++++++++++++

The goal of the Deployer design is to minimize the time needed to write deployment files by eliminating
the need to ever duplicate a property definition for a server or a project configuration. To achieve this goal,
Deployer uses a hierarchical organization of properties for both servers and configuration, organized by ``project``,
``instance`` and ``tenant``.

In general, each ``instance`` belongs to exactly one ``project``, and
each ``tenant`` belongs to exactly one ``instance``. Properties can be defined at a
``project``, ``instance`` or ``tenant`` level, and property values defined at a more "local" level override those
defined at a more "global" level. For example. if the ``jupyter_user_image`` property is defined for the "prod"
instance but also for the "DesignSafe" tenant within the "prod" instance, then the value defined for DesignSafe would
be used for all deployment actions taken against that tenant.

More details are given in the `User Guide <../users/index.html>`_.

Ansible
+++++++

The Deployer contains scripts that can be launched from the command line to manage deployments on remote servers.
It does so by first reading configuration files, server files, (optionally) extra files and command line arguments
provided by the operator to generate temporary `Ansible <https://www.ansible.com/>`_ playbooks and then execute
these playbooks on the remote servers specified. In general, the operator should not need to know anything about the
generated Ansible scripts, and by default, Deployer removes these files after each command. For debugging purposes,
Deployer can be instructed to keep these files using the ``-k`` flag.

