Overview
--------

This section of the documentation is intended for individuals planning to use Deployer to mange their own deployments of
CIC projects. Before starting this section, make sure to go through the :doc:`getting-started/index` section and
install the Deployer CLI.

Terminology
-----------
Deployer makes use of the following concepts to manage remote deployments:

:Project: A CIC software project that will be managed. Support is planned for JupyterHub, Abaco, and Agave.
:Instance: A physical manifestation of a project, often times for a specific environment such as “dev”, “staging”, “production”, etc.
:Tenant: A logical isolation of a (typically, subset) of resources within an instance.
:Action: The devops action to take against the project such as “deploy”, “update”, etc. Represents a single, atomic action to take on a set of servers; typically implemented as one or more playbooks. Actions may invoke other actions as part of execution.

Note that a given instance is only defined within the context of a project; for example, the ``prod`` instance of the
``JupyterHub`` project and the ``prod`` instance of the ``Abaco`` project are different. In the same way, a tenant
is only defined within a given instance.


Servers
-------
Servers are the remote hosts where projects will be deployed. Deployer will ultimately execute commands over SSH on these
servers to manage the deployments on behalf of the operator. The servers an operator wishes to have Deployer operate on
are described in one or more server files. Here is a small example of a potential servers file:

.. code-block:: bash

    ---
    ssh_user: root
    ssh_key: my_ssh.key

    jupyterhub.prod.sd2e:
        cloud: roundup
        hosts:
            - host_id: manager1
              ssh_host: 129.114.97.247
              private_ip: 129.114.97.247
              roles:
                - hub
            - host_id: worker1
              ssh_host: 129.114.97.248
              private_ip: 129.114.97.248
              roles:
                - worker
            - host_id: worker2
              ssh_host: 129.114.97.249
              private_ip: 129.114.97.249
              roles:
                - worker

    jupyterhub.staging:
        cloud: jetstream
        hosts:
            - host_id: worker2
              ssh_host: 129.114.17.47
              private_ip: 10.10.10.6
              roles:
                - worker

In the above example, we define two global properties, ``ssh_user`` and ``ssh_key``, and two groups of servers defined
in the ``jupyterhub.prod.sd2e`` and ``jupyterhub.staging`` stanzas.

In general, server files should conform to the following requirements:

* Written in the YML format ending in the ``.hosts`` extension.
* Each entry in the YML file should be either a global property to apply to all servers in the file (e.g., ``ssh_key``
for the SSH key Deployer should use to access to the servers, as in the above example) or a stanza containing
descriptions of one or more servers.
* Each host must be assigned to a project and can be optionally assigned to an instance or tenant. The project, instance
and tenant are specified through the YML key in dot notation, ``<project>.<instance>.<tenant>``
* Each host must define ``ssh_host``, the IP address or domain of the host and ``host_id``, a unique name for the host.
* Each host can optionally provide a list of roles. The roles attribute must be a list.
* Each host can have a additional key:value pairs used for further filtering (e.g. “cloud: roundup”)

The servers file to be used can be specified at the command line (-s) or in the deployment.yml file. Otherwise,
the first file with a .hosts extension in the current working directory will be used.
ssh_key is a relative path from the directory where your hosts/configs are



Configs
-------
Files in the YML format ending in the ``.configs`` extension.
The config file to be used can be specified at the command line (-c) or in the deployment.yml file. Otherwise, the first file with a .configs extension in the current working directory will be used.
Each config must specify a project and can be optionally assigned to an instance or tenant.
Configs are arbitrary key:value pairs, and the values can be complex objects.

Passwords
---------
All files in the current working directory with a ``.passwords`` extension will be read in.
The passwords file to be used can be specified at the command line (-z) or in the deployment.yml file. Otherwise, the first file with a .passwords extension in the current working directory will be used.
Each password can be optionally assigned to a project, instance or tenant.
Configs are arbitrary key:value pairs, and the values can be complex objects.

ensure that the ``oauth_client_secret`` is in the ``.passwords`` file
to generate:

.. code-block:: bash

    $ curl -u <service_account>:<service_account_password> -d "clientName=jupyterhub_staging&callbackUrl=https://<jupyter_staging_domain>/hub/oauth_callback" https://api.3dem.org/clients/v2

The consumerKey returned maps to ``<project>_oauth_client_id``
consumerSecret returned maps to ``<project>_oauth_client_secret``

extras_dir
----------
Path relative to the current working directory to a directory containing extra files needed for configuring the actions. These files could include SSL certificates, the Jupyter volume_mounts config, the Abaco service.conf, etc. When possible, actions are strongly encouraged to generate these files using configuration strings and templates instead of requiring physical files in the extras_dir.