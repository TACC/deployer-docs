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

In addition to a Docker 1.11 Swarm cluster, an NFS management cluster will be constructed from the ``hub`` node (nfs server)
to all ``worker`` nodes (nfs clients). This NFS management cluster is required as it is used to persist login data from
the hub to the workers.


Primary Configuration Requirements and Options
++++++++++++++++++++++++++++++++++++++++++++++

The following configuration properties are required.

* ``tenant`` - The tenant id this JupyterHub belongs to. Determines the OAuth server to use.

* ``jupyterhub_host`` - The domain to serve JupyterHub on (without the protocol); e.g. "jupyter.designsafe-ci.org".

* ``jupyterhub_oauth_client_id`` - The ID for the OAuth client that JupyterHub will use to authenticate users.

* ``jupyterhub_oauth_client_secret`` - The secret for the OAuth client that JupyterHub will use to authenticate users.
  It is recommended that the ``jupyterhub_oauth_client_secret`` be stored in a ``.passwords`` file.


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


Abaco
=====

`Coming soon...`

Agave
=====

`Coming soon...`
