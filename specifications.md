Confpack Design Philosophy (WIP)
================================

Objectives/Motivations
----------------------

* Be able to replace Chef/Ansible/Puppet as the only configuration management system.
   * Why? Because Chef/Ansible/Puppet does not have a “down” operation. All operations are “up”. This means if a statement is removed from the cookbooks, the files/config/whatever will be left on the system until someone manually goes and remove it, or add a statement in the cookbook to remove. This is cumbersome and error-prone.
   * Package management has already partially solved this issue and comes preinstalled on most distributions.
* Make package building easy.
   * Automated builder from some sort of playbook specifying the files to include, templates to include.
* (STRETCH) Make package distribution easy.
   * Maybe upload to ubuntu ppa.
   * confpack-server for apt servers.
   * Probably requires separate program
* (STRETCH) Make binary distribution easy for deployment.
   * Probably requires separate program
   * Should allow for rollback as well as forward.

Typical Usage Patterns
----------------------

* Configuration management should be used for configuration management only. It should not be used for deployment, service discovery, continuous integration, or other “creative” use cases.
* This means most of the time:
   * Installing packages via package management.
   * You create a directory on the server.
   * You copy a file to the server.
   * You copy a file to the server and then apply templating on it.
   * You copy a file to the server and then apply templating on it with secrets.
   * You can execute certain commands such as restarting services via systemd.
* Sometimes, you would like to do somethings that are really just the stuff listed above:
   * “Enabled” a service to start on boot time.
   * Add a line to an existing configuration file such as /etc/hosts
   * Creating a user/group
   * Downloading certain files like an ubuntu iso (although maybe you should consider packing that)
      * And verify checksums
   * Manage iptables
   * Cloning a git repository (probably should be discouraged. Care to comment?)
* Other use cases should be evaluated to see if it fits within the envelop of configuration management. If they are deemed to not fit within the envelop of configuration management, we should discourage it from a documentation/code standpoint.
* The system should be able to manage one => hundreds => thousands => hundreds of thousands of nodes without being changed. Load balancing on the package distribution server might need to be a thing, but that’s just another concern of #scaling.

Design Philosophy
-----------------

1. Configuration management is for configurations. Creative usages such as service discovery, deployment, continuous integration, and so forth should be disallowed/discouraged.
2. All packages installed must cleanup itself after being removed, eliminating the concern of an eventually inconsistent system.
3. Use dependencies as much as possible rather than rewriting things. Modularity is important, but don’t over do it.
4. Use role packages (meta packages) to keep track of what each node have installed.
5. Removing explicitly installed package should remove unneeded implicitly installed packages to keep system clean. This means once a dependency is removed from a node package, upgrade should remove it.
6. A system should be as immutable as possible after configurations (packages) have been applied (evaluate Ubuntu snappy/Nix for this).

MVP Technical Requirements
--------------------------

* Ability to package files and templates.
* Ability to install and upgrade packages. When upgrading it will delete files that were in a previous version but not in the new version.
* Ability to remove packages, which will remove all files installed by that package.
* Ability to build meta packages so we can create “roles”. When upgrading roles it will cleanup packages that were in a previous version but not in the new version.
* (MVP v2) Ability to package secrets.
* (STRETCH) Ability to distribute packages via confpack-server.
* (STRETCH) Compile time broken dependency checks.

Technical Issues/Discussions
----------------------------
* All package probably needs to depend on a confpack-runtime package that allows us to do templating easily.
* Templating should be done in postinst with jinja2.
* All files probably should be declared as regular files rather than config files so we don’t have to use purge to remove everything.
* Nodes should have their own metapackages, either nodename.deb or companyname-webserver-meta.deb companyname-dbserver-meta.db

Examples (Maybe)
---------------

### Configuration repository structure ###

	packages/
		nginx-conf/
		        control/
		                control.yml
		                postinst
		                …
		        etc/
		                nginx/
		                        …
		dnsmasq-conf/
		        …
	roles/
		nodename.yml 
		webserver.yml


The `packages` directories contains all the typical playbooks for a program. Roles builds meta packages that depend on those on top.

### Command line usage ###

This is for the building process. Can be something like a git postreceive hook.

	$ confpack-build
	building all packages…
	…
	$ ls | grep debs
	debs 
	$ cd debs && ls
	nginx-conf-<version>-<sha>.deb 
	...
	...
	$ confpack-upload confpack-apt-server.company.com


On the server:

	$ sudo confpack set-environment master # or other labels you would like to apply. Use distributions or whatever. This work ties into the distribution server.
	$ sudo confpack update # does apt-get update
	$ sudo confpack upgrade # does apt-get upgrade and apt-get autoremove
	$ sudo confpack upgrade --version <version/sha/whatever>


To bootstrap a server:

	$ sudo dpkg -i confpack-runtime.deb # or setup apt sources and install it.
	$ sudo dpkg -i confpack-your-conf.deb # this is built by confpack, sets a target server, and other stuff. Need to hammer out confpack server.
	$ sudo confpack set-environment …
	$ sudo confpack install node-package # or if you will: install webserver appserver...
	$ sudo confpack update && sudo confpack upgrade
