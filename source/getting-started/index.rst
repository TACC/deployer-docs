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

    $ export version=dev
    $ docker run  -v $(pwd):/conf tacc/deployer:$version --setup

This step pulls down the image and installs a small alias script, ``deployer.sh``, in the current working directory.
With the alias installed, simply issue commands directly to the script; for e.g.

.. code-block:: bash

    $ ./deployer --help

will display help.


Basic Usage
===========

The general format for executing commands via the the Deployer CLI is:

.. code-block:: bash

    $ ./deployer -p <project> -i <instance> -t <tenant> -a <action> -d <deployment_file>

The arguments ``project``, ``instance``, etc, are defined in the following table
and are covered in more detail in the User Guide.

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
| -d DEPLOYMENT, --deployment DEPLOYMENT | Relative path to deployment file, in the YAML format,                    |
|                                        | describing the activity. This file should be in the                      |
|                                        | current working directory or a sub directory therein.                    |
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

Ansible
+++++++

The Deployer contains scripts that can be launched from the command line to manage deployments on remote servers.
It does so by first reading configuration files, server files, (optionally) extra files and command line arguments
provided by the operator to generate temporary `Ansible <https://www.ansible.com/>`_ playbooks and then execute
these playbooks on the remote servers specified. In general, the operator should not need to know anything about the
generated Ansible scripts, and by default Deployer removes these files after each command. For debugging purposes,
Deployer can be instructed to keep these files using the ``-k`` flag.

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

The goal of the Deployer design is to minimize the time needed to write deployment files by eliminating duplicate
the need to ever duplicate a property definition for a server or a project configuration. To achieve this goal,
Deployer uses a hierarchical organization of properties for both servers and configuration.

