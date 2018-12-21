Projects
________

This section covers documentation for specific projects supported by Deployer.

JupyterHub
==========

TACC's integrated JupyterHub project extends the community supported
`JupyterHub <https://jupyterhub.readthedocs.io/en/stable/>`_  with customizations that enable deeper integration and
ease of use of TACC resources from directly within running notebook servers.

The main features provided by the TACC JupyterHub are:

* Individual Jupyterhub instance for each project with customized notebook server image based on the research interests of the community.
* Dedicated notebook server running in a Docker container for each user.
* Persistent storage volumes backed by large-scale storage at TACC and mounted directly to the "permanent" directory within each notebook container.
* Additional volume mounts, customizable through configuration, to provide POSIX interfaces to additional file collections stored on TACC systems.
* Integrated authentication with the TACC OAuth server or another tenant OAuth server.
* TACC CLI, agavepy, and other libraries pre-installed in the user's notebook server for integrating with other TACC Cloud services.


The CIC group at TACC maintains the core TACC JupyterHub code and automated deployment management in Deployer. The notebook
server images are developed by a number of individuals across multiple groups at TACC as well as through contributions
by external partners.


Primary Server Requirements and Options
+++++++++++++++++++++++++++++++++++++++

The TACC JupyterHub uses Docker Swarm 1.11 to deploy notebook servers as Docker containers on a cluster. It requires
a minimum of two servers meeting the following requirements:

* One server with the  ``hub`` role. This server will run the main JupyterHub application and therefore will require
  ports 80 and 443 open to the public internet.

* One or more servers with the ``worker`` role. These servers will run the user notebook containers.

* Ports must be opened between manager and workers to enable communication. For each server, the optional ``private_ip``
  attribute may be specified. If is it, communication will happen on this IP. Otherwise, it will happen on ``ssh_host``.
  If firewalld and/or iptables are running on the hosts, make sure that both the local IP and the 172.17.0.0 subnets
  (the Docker0 bridge) are opened.

In addition to a Docker 1.11 Swarm cluster, an NFS management cluster will be constructed from the ``hub`` node (nfs server)
to all ``worker`` nodes (nfs clients). This NFS management cluster is required as it is used to persist login data from
the hub to the workers.


Primary Configuration Requirements and Options
++++++++++++++++++++++++++++++++++++++++++++++

The following configuration properties are required.

* ``tenant`` - The tenant id this JupyterHub belongs to. Determines the OAuth server to use.

* ``jupyterhub_tenant_base_url`` - The base URL for the OAuth server. Note: this property is only required if not
  using a standard TACC tenant; for standard TACC tenants, Deployer will automatically derive the base URL.

* ``jupyterhub_host`` - The domain to serve JupyterHub on (without the protocol); e.g. "jupyter.designsafe-ci.org".

* ``jupyterhub_oauth_client_id`` - The ID for the OAuth client that JupyterHub will use to authenticate users.
  The JupyterHub requires a valid OAuth client for the tenant being used, and this
  OAuth client *must* be registered with the required ``callbackUrl`` for the hub, which has the form
  ``https://<jupyterhub_host>/hub/oauth_callback``. See `Additional Considerations`_ below for more details.

* ``jupyterhub_oauth_client_secret`` - The corresponding client secret for the OAuth client. Note: it is recommended
  that the ``jupyterhub_oauth_client_secret`` be stored in a ``.passwords`` file.


The following configuration properties are optional, though some are strongly encouraged; see below.

* ``jupyterhub_cert`` and ``jupyterhub_key`` - As strings, the contents of an SSL certificate and key for the
  domain specified in ``jupyterhub_host``. If these attributes are not specified, *self-signed certificates will be
  supplied*. This will result in insecure warnings in the browser for users.

.. * ``use_stock_nfs`` (true/false) - Whether to use the management NFS cluster for user's persistent data instead of a
  custom NFS share for user data. The default is ``false``. This option is not recommended since user data will be
  stored on the ``hub`` node which will typically limit overall storage capacity.

* ``volume_mounts`` - A list of directories to mount into every user notebook container. Each mount should be a string
  in the format ``<host_path>:<container_path>:<mode>`` where ``host_path`` and ``container_path`` are absolute paths
  and ``mode`` is one of ``ro`` (read only), or ``rw`` (read-write). Also, the paths variables recognize the following
  template variables:
    * ``{username}`` - the username of the logged in user.

    * ``{tenant_id}`` - the tenant id for the JupyterHub.

    * ``{tas_homeDirectory}`` - the home directory for the logged in user in TAS (TACC Accounting System). This
      template variable can only be used in tenants using the TACC identity provider (see `Integrating with StockYard`_
      below).

* ``jupyterhub_image`` - Docker image to use for central JupyterHub program. Will default to deploying the latest stable
  version. Image must be available on the public Docker Hub.

* ``jupyter_user_image`` - Docker image to use for user notebook servers. If not specified, the latest stable version of
  ``taccsciapps/jupyteruser-base`` will be used.

* ``jupyteruser_uid`` and ``jupyteruser_gid`` - The uid and gid to run the user notebook servers as; if not specified,
  the uid and gid created in the Docker image specified by ``jupyterhub_image`` will be used unless integration with
  TAS is enabled (see `Integrating with TAS`_ below).


Additional Considerations
+++++++++++++++++++++++++

We highlight some additional considerations regarding JupyterHub configuration.

* While ``volume_mounts`` is technically optional, at least one mount is needed to provide persistent user data; we
  generally recommend mounting a user directory on the host (e.g. ``/path/to/nfs/share/{username}`` to a path such as
  ``/home/juupyter/my_data`` inside the container.

* Every ``host_path`` in the ``volume_mounts`` parameter must exist on *all* worker nodes (for example, via an NFS share)
  or container execution will fail. Unless using the NFS management cluster, Deployer assumes these directories have
  been created already.

* For security purposes, ensure that the ``oauth_client_secret`` is in the ``.passwords`` file

* To generate an OAuth client key and secret for your JupyterHub instance, use a command like the following:

.. code-block:: bash

    $ curl -u <service_account>:<service_account_password> \
    -d "clientName=<you_pick>&callbackUrl=https://<jupyterhub_host>/hub/oauth_callback" \
    https://<tenant_base_url>/clients/v2

Integrating with TAS
++++++++++++++++++++

If the tenant being used for JupyterHub leverages the TACC identity provider (i.e., ldap.tacc.utexas.edu) then
JupyterHub can integrate with TAS (the TACC Accounting System) to enable individual notebook servers to run as the uid
and gid of the logged in user. This feature provides the advantages of "vertical single sign-on". i.e., files created
and/or updated by the user in the notebook server will have the correct ownership properties. Integrating with TAS is
required for integrating with Stockyard, TACC's gloabl file system. (see `Integrating with Stockyard`_ below).

Integrating with TAS also requires the following configurations:

* ``use_tas_uid`` and ``use_tas_gid`` - Setting to ``true`` instructs JupyterHub to launch the user's notebook with the
  uid and gid for the user in TAS.

* ``tas_role_account`` and ``tas_role_pass`` - an account and password in TAS for Jupyterhub to use to make TAS API
  calls.


Integrating with StockYard
++++++++++++++++++++++++++

JupyterHub instances can integrate with TACC's global file system, Stockyard, if TAS integration has been enabled. This
option is only available to approved JupyterHub instances deployed within the secured TACC network.

In order to provide file system mounts from Stockyard into user notebook servers, a Lustre mount to Stockyard must be
made on all worker nodes.

Once the Lustre mounts have been created on the worker nodes, the only configuration required is to add notebook
container mounts to Stockyard using the ``volume_mounts`` parameter. For example, if Stockyard is mounted at ``/work``
on all worker nodes, creating a mount with the following config

.. code-block:: bash

  /work/{tas_homeDirectory}:/home/jupyter/tacc-work:rw

would mount the user's $WORK directory at ``/home/jupyter/tacc-work`` in read-write mode within the container.


Support for Project Mounts (DesignSafe)
+++++++++++++++++++++++++++++++++++++++

As an example of custom functionality that be can be added for specific JupyterHub instances, we describe the support
for project mounts in the DesignSafe tenant.

`Coming soon...`


Abaco API
=========

`Coming soon...`

Agave API
=========

TACC's Agave API provides services for enabling computational experiments on HPC and HTC resources. For more information
on the API, see TACC's official `documentation <https://agave.readthedocs.io>`_.

The Agave API is currently organized into frontend services and worker components. These agents interact through a
persistence layer comprised of a Mongo and MySQL database and a Beanstalk and RabbitMQ queue. All Agave components are
packaged into Docker images and configured through environment variables.

In addition to the persistence layer, the Agave services are secured with `JWT <https://tools.ietf.org/html/rfc7519>`_
authentication and some authorization aspects through claims. The services are built to integrate with the
TACC `OAuth Gateway`_, though in theory, any API Gateway capable of generating a conformant JWT could work.


Minimal Quickstart
++++++++++++++++++

Agave can be deployed in a "sandbox" configuration for evaluation and/or prototyping purposes with minimal configuration.
When using this setup, Deployer installs and runs all required Agave databases and automatically wires up the Agave
services to those databases.

.. note::
  The minimal quickstart does not include settings for changing the default database passwords, so it is insecure and
  should not be used in production!

The minimal quickstart required three servers (or VMs) - one for the OAuth Gateway, one for the Agave components, and
a third for the persistence layer. A hosts file and a config file will be required. The following samples can be used
as a starting point for deploying a complete sandbox, including all AGave components and databases and the OAuth Gateway:

Hosts File Sample
~~~~~~~~~~~~~~~~~

.. code-block:: bash

    ssh_user: root
    ssh_key: my_ssh.key

    oauth.sandbox.dev:
        hosts:
            - host_id: oauth-sbox
              ssh_host: <IP_1>
              roles:
                - oauth
            - host_id: oauth-dbs-sbox
              ssh_host: <IP_3>
              roles:
                - all_oauth_dbs

    agave.sandbox:
        hosts:
            - host_id: ag-sbox-all
              ssh_host: <IP_2>
              roles:
                - agave_frontends
                - agave_workers
            - host_id: ag-sbox-dbs
              ssh_host: <IP_3>
              roles:
                - all_agave_dbs

Notes:

* Replace ``<IP_1>, <IP_2>, <IP_3>`` with actual IPs or hostnames for your servers. Deployer must be able to SSH to these IPs to install the software.
* We have used ``sanbox`` as the instance identifier; this can be changed, as desired.
* We have used ``dev`` as the tenant identifier; just as with the instance identifier, this can be changed.


Configs File Sample
~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

    oauth.sandbox.dev:
        base_url: dev.api.example.com
        agave_frontend_host: <IP_2>
        use_cic_ldap: True


Notes:

* We have not specified any Agave configs as technically they are not required for a minimal setup. However, without setting at a minimum database password condfigs, the deployment will not be secure.
* The value of ``base_url`` (``dev.api.example.com`` in the example above) will be the primary URL for all APIs. This should be changed to a domain owned by the organization.
* Using the above config, the OAuth Gateway will be deployed with self-signed certificates. See the `OAuth Gateway`_ section for additional configuration options, including deploying with valid certiciates.

A Word on Ports
~~~~~~~~~~~~~~~

The OAuth Gateway and Agave projects communicate with the various databases on specific ports. Therefore, the database
ports on ``<IP_3>`` must be reachable from ``<IP_1>`` and ``<IP_2>``. If that is not possible using the ``ssh_host``
value configured in the hosts file, separate configs can be provided to specify the IP to use for each database, e.g.,
``agave_mysql_host`` -- see `Service and Host Configs`_ for a complete list.

For example, when using a cloud provider such as OpenStack, it is often possible to assign servers an IP on a private
network and for the OAuth Gateway and Agave services to use that IP for communication to the databases.


Deploying the Sandbox
~~~~~~~~~~~~~~~~~~~~~

Use the Deployer CLI to deploy the sandbox with two commands. First, deploy the Agave project as follows:

.. code-block:: bash

    $ deployer -p agave -i sandbox -a deploy

Next, deploy the OAuth Gateway project, which requires specifying the tenant in addition to the instance:

.. code-block:: bash

    $ deployer -p oauth -i sandbox -t dev -a deploy

Note that these commands do not explicitly specify the hosts and configs file to use. Deployer will use the first file
it finds with extension `.hosts` (respectively, `.configs`) in the current working directory. If you have multiple
hosts or configs files in the current working directory, specify the correct one using the ``-s`` (respectively, ``-c``)
flags. See the `User Guide <../users/index.html>`_ for more details.


Service and Host Configs
++++++++++++++++++++++++

Like other Deployer projects, Agave deployments leverage settings specified on hosts, either through special roles or
other attributes, and global settings specified through the configs file. The configs object should contain an instance
identifier and any Agave attributes to apply to all services in the instance.

At a minimum, the following global configs must be specified:

* ``agave_mysql_host`` - Hostname or IP address for the MySQL database.
* ``agave_mysql_user`` - MySQL user to use.
* ``agave_mysql_password`` - Password associated with MySQL user.
* ``mysql_root_user`` - MySQL root user; used to create schemas, etc.
* ``mysql_root_password`` - Password associated with MySQL root user.

* ``agave_mongo_host`` - Hostname or IP address for the MongoDB database.
* ``agave_mongo_user`` - MongoDB user to use.
* ``agave_mongo_password`` - Password associated with MongoDB user.
* ``mongo_admin_user`` - Admin mongo user; used to create collections, indexes, etc.
* ``mongo_admin_password`` - Admin mongo password.


* ``agave_beanstalk_host`` - Hostname or IP address for the beanstalk instance.

* ``agave_rabbitmq_host`` - Hostname or IP address for the RabbitMQ instance.

Note: the database host attribute will be derived automatically if a host in the servers file has the corresponding role,
e.g., ``agave_mysql`` or ``all_agave_dbs``.


Roles
~~~~~

The following roles can be set on a per-host basis to deploy specific components of Agave on a given server.

* ``agave_frontends`` - Run a set of Agave Frontend services. Looks for the ``agave_frontends`` attribute to determine which services to run. If that attribute is not defined, all frontend services will be run.
* ``agave_workers`` - Run a set of Agave Workers. Looks for the ``agave_workers`` attribute to determine which workers to run. If that attribute is not defined, all workers will be run.

Attributes
~~~~~~~~~~



OAuth Gateway
=============

The TACC OAuth Gateway provides two primary functions: 1) an OAuth2 provider server and 2) an API Gateway and reverse
proxy to APIs registered with the server. When a request is made to a registered API using an OAuth access token, the
API Gateway function will generate a JWT corresponding to the identity information contained within the token before
forwarding the request to the actual service.


