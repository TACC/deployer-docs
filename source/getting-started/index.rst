Getting Started
---------------

This Getting Started guide will conver installing the Deployer CLI as well as some initial basic concepts needed before
moving to the `User Guide <../users/index.html>`_.


Installation
============

To install the Deployer CLI, the only dependency is Docker. If you do not have Docker installed, see the
`Official Docs <https://docs.docker.com/install/>`_ for installation instructions for your platform.

Once Docker is installed, change into a directory that will contain your configuration
files and run:

.. code-block:: bash

    $ docker run  -v $(pwd):/conf tacc/deployer --setup

This step pulls down the latest stable version of the Deployer Docker image and installs a small alias script,
``deployer.sh``, in the current working directory.

.. note:: Different versions of Deployer can be installed by specifying a TAG on the ``tacc/deployer`` image.


With the alias installed, simply issue commands directly to the script. For example. validate your installation by
executing:

.. code-block:: bash

    $ ./deployer --help

which should display the help.



Basic Usage
===========

The general format for executing commands via the the Deployer CLI is:

.. code-block:: bash

    $ ./deployer -p <project> -i <instance> -t <tenant> -a <action> -d <deployment_file>

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
A ``project`` is one of a set of systems Deployer knows how to manage, and will eventually include values "JupyterHub",
"Abaco" and "Agave". The values for ``instance`` and ``tenant`` can be chosen by the operations team to best organize
their infrastructure and configuration. One approach is to use ``instance`` values to distinguish physically isolated
systems such as "development" and "production" and to use ``tenant`` values to distinguish logically separated aspects of
systems (such as the DesignSafe tenant for JupyterHub or Abaco).

Actions
+++++++



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

