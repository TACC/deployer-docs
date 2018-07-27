Overview
--------

This section of the documentation is intended for individuals planning to use Deployer to mange their own deployments of
CIC projects. Before starting this section, make sure to go through the
`Getting Started <../getting-started/index.html>`_ section and install the Deployer CLI.

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


Configs
-------
Operations teams put desired project configuration in YML files ending in the ``.configs`` extension. The config file
to be used can be specified at the command line (``-c``) or in the deployment.yml file. Otherwise, the first file with a
``.configs`` extension in the current working directory will be used.

Each config must specify a project and can be optionally assigned to an instance or tenant. These are specified by
defining a YML key in the format ``<project>.<instance>.<tenant>`` and then defining the properties using valid
YML within that stanza. Configs can be complex YML objects.

Consider the following example config file:

.. code-block:: bash

    ---
    jupyterhub:
        jupyterhub_image: taccsciapps/jupyterhub:0.8.0
        jupyter_user_image: taccsciapps/jupyteruser-ds:1.1.0
        use_stock_nfs: false
        jupyterhub_config_py_template: jupyterhub_config_0.8.py.j2
        worker_tasks_file: designsafe.yml

    jupyterhub.prod.designsafe:
        use_stock_nfs: true
        # prod didn't define images, etc., so it should inherit from the above.

        # here's an example of a config leveraging a complex YML object:
        volume_mounts:
          - /corral-repl/tacc/NHERI/shared/{username}:/home/jupyter/MyData:rw
          - /corral-repl/tacc/NHERI/published:/home/jupyter/NHERI-Published:ro

    jupyterhub.staging.designsafe:
        jupyterhub_image: taccsciapps/jupyterhub:0.8.1
        jupyter_user_image: taccsciapps/jupyteruser-ds:1.2.0rc9
        # these image properties ^^ override the global onces defined above, but just for Designsafe staging
        cull_idle_notebook_timeout: 259200
        notebook_memory_limit: 3G
        notebook_offer_lab: True

In the first stanza, with key ``jupyterhub``, a set of configuration properties and values are specified. These
values will provide a default for all actions taken on the "jupyterhub" project if the property isn't otherwise
specified. In the next two stanzas, properties values for the "designsafe" tenant within the "prod" and "staging"
instances, respectively, are defined. These values override the values defined in the "jupyterhub" stanza when
deployment actions are taken against the particular instances.

So, when running Deployer actions against the
prod instance of the DesignSafe tenant, the ``jupyterhub_image`` would have value ``taccsciapps/jupyterhub:0.8.0``
(inherited from the ``jupyterhub`` stanza) and ``use_stock_nfs`` would have value ``true``. For the staging instance,
``jupyterhub_image`` would have value ``taccsciapps/jupyterhub:0.8.1`` and ``use_stock_nfs`` would have value ``false``.

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
in the ``jupyterhub.prod.sd2e`` and ``jupyterhub.staging`` stanzas. Property values are organized hierarchically just like
configuration values, and more "local" values override more "global" values.

In general, server files should conform to the following requirements:

* Written in the YML format ending in the ``.hosts`` extension.
* Each entry in the YML file should be either a global property to apply to all servers in the file (e.g., ``ssh_key``
for the SSH key Deployer should use to access to the servers, as in the above example) or a stanza containing
descriptions of one or more servers.
* Each host must be assigned to a project and can be optionally assigned to an instance or tenant. The project, instance
and tenant are specified through the YML key in dot notation, ``<project>.<instance>.<tenant>``
* Each host must define ``ssh_host``, the IP address or domain of the host and ``host_id``, a unique name for the host.
* The ``ssh_key`` and ``ssh_user`` properties are also required for all hosts, though they can be inherited. The
``ssh_key`` parameter must be a relative path from the directory where Deployer is run to a key file that can be used to
access the server via SSH, and the ``ssh_user`` parameters must be a string representing the user account to SSH as.
* Each host can optionally provide a list of ``roles``. Project scripts use ``roles`` to distinguish which services
should run on which servers. The roles attribute must be a list.
* Each host can have a additional key:value pairs defined by the operators used for further filtering (e.g.
“cloud: jetstream”)

The servers file to be used can be specified at the command line (-s) or in the deployment.yml file. Otherwise,
the first file with a .hosts extension in the current working directory will be used.



Passwords
---------

In Deployer, password files are used to specify sensitive properties such as database passwords or the contents of an
SSL certificate. The passwords file to be used can be specified at the command line (-z) or in the deployment.yml file.
Otherwise, the first file with a .passwords extension in the current working directory will be used. Password files
are the same as config files in all regards; for example, each password is either a global proerty or (optionally)
is assigned to a project, instance or tenant and password properties can be arbitrary YML objects.


extras_dir
----------
Path relative to the current working directory to a directory containing extra files needed for configuring the actions. These files could include SSL certificates, the Jupyter volume_mounts config, the Abaco service.conf, etc. When possible, actions are strongly encouraged to generate these files using configuration strings and templates instead of requiring physical files in the extras_dir.