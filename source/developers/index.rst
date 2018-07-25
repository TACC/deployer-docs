High Level Algorithm Design of Core CLI
---------------------------------------

1. Read Servers, Configs and Passwords files, and denormalize each of these into “flat” python objects; properties specified at a more “local” level override properties specified at a more “global” level.

2. Filter Servers, Configs and Passwords using the specified project, instance and tenant (if specified).

3. Combine Configs and Passwords into a single Configuration object.

4. Look up action class by project; each project/action can do the following things:
    - Define implied roles to apply to the servers list; e.g., the “hub” role implies the “swarm_manager” role.
    - Audit the filtered Servers and Configuration object for required fields.
    - Supply default values for variables within the Configuration object for optional fields. This will be a formal mechanism so that the central script can report when optional field values are supplied.
    - Execute one or more playbooks. A central “execute_playbook()” callable will be provided that can accept the Servers and Configuration objects (the action code could thus further filter either object before calling the central execute_playbook).
    - Execute one or more additional actions. A central “execute_action()” callable will be provided that can accept the Servers and Configuration objects (the action code could thus further filter either object before calling the central execute_action).
5. execute_playbook
    - Create a temporary directory for the playbook to run in.
    - Copy playbook source directory (roles, playbook files, etc.) to the temp dir.
    - Copy the extras_dir to a directory called _deployer_extras within the temp dir.
    - Write the Servers object to a file called _deployer_servers.yml in the temp dir.
    - Determine the unique roles among the servers.
        - For each role, make a group in the servers file (i.e., [<role_name>]) and write out all servers that are in that role with all variables attached.
        - In general, a given server could appear in multiple groups but that is OK.
    - Write the Configuration object to a file called _deployer_configuration.yml in the temp dir.
    - Execute the playbook through a subprocess (ansible-playbook) and reference the servers file (-i <servers>).
    - The playbook/roles should, by convention, issue a “- include_vars: _deployer_configuration.yml” at the top of the tasks list to pick up the configuration.


