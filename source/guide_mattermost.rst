.. author:: Nico Graf <hallo@uberspace.de>

.. tag:: web
.. tag:: lang-go
.. tag:: chat

.. highlight:: console

.. sidebar:: Logo

  .. image:: _static/images/mattermost.svg
      :align: center

##########
Mattermost
##########

.. tag_list::

`Mattermost`_ is an open-source, self-hosted online chat service written in Go and JavaScript.

----

.. note:: For this guide you should be familiar with the basic concepts of

  * :manual:`PostgreSQL <guide_postgresql>`
  * :manual:`supervisord <daemons-supervisord>`
  * :manual:`domains <web-domains>`

Prerequisites
=============

You need to have :manual:`PostgreSQL <guide_postgresql>` initialized and configured. Tested with PostgreSQL version 15.6.

Your URL needs to be setup:

.. include:: includes/web-domain-list.rst

Installation
============

Download the most recent Linux TAR archive from the `Mattermost website`_:

.. code-block:: console

  [isabell@stardust ~]$ wget https://releases.mattermost.com/9.7.1/mattermost-9.7.1-linux-amd64.tar.gz
  --2024-04-22 17:06:14--  https://releases.mattermost.com/9.7.1/mattermost-9.7.1-linux-amd64.tar.gz
  Resolving releases.mattermost.com (releases.mattermost.com)... 18.244.18.26, 18.244.18.94, 18.244.18.91, ...
  Connecting to releases.mattermost.com (releases.mattermost.com)|18.244.18.26|:443... connected.
  HTTP request sent, awaiting response... 200 OK
  Length: 289186599 (276M) [application/x-tar]
  Saving to: ‘mattermost-9.7.1-linux-amd64.tar.gz’

  100%[============================================================================================================================================================================================>] 289'186'599 68.8MB/s   in 4.1s

  2024-04-22 17:06:19 (66.8 MB/s) - ‘mattermost-9.7.1-linux-amd64.tar.gz’ saved [289186599/289186599

  [isabell@stardust ~]$

Extract the archive:

.. code-block:: console

  [isabell@stardust ~]$ tar -xvzf mattermost-*.tar.gz
  […]
  mattermost/prepackaged_plugins/mattermost-plugin-custom-attributes-v1.0.2.tar.gz
  mattermost/prepackaged_plugins/mattermost-plugin-zoom-v1.1.2.tar.gz
  [isabell@stardust ~]$


Configuration
=============

Set up a Database
-----------------

Run the following code to create the database and user:

.. code-block:: console

  [isabell@stardust ~]$ createdb --encoding=UTF8 mattermost
  [isabell@stardust ~]$ psql -d mattermost -c "CREATE USER mmuser WITH PASSWORD 'MySuperSecretPassword';"
  CREATE ROLE
  [isabell@stardust ~]$ psql -d mattermost -c "GRANT ALL PRIVILEGES ON DATABASE mattermost to mmuser;"
  GRANT
  [isabell@stardust ~]$ psql -d mattermost -c "ALTER DATABASE mattermost OWNER TO mmuser;"
  ALTER DATABASE
  [isabell@stardust ~]$ psql -d mattermost -c "GRANT USAGE, CREATE ON SCHEMA PUBLIC TO mmuser;"
  GRANT
  [isabell@stardust ~]$

Change the configuration
------------------------

You need to set up your URL, and database settings in ``~/mattermost/config/config.json``.

First, set your site URL:

.. code-block:: none

    "SiteURL": "https://isabell.uber.space"

Then find the ``SqlSettings`` block and in ``DataSource`` replace ``mostest`` with your database password and ``mattermost_test`` with your database name:

.. code-block:: javascript
 :emphasize-lines: 2,3

  "SqlSettings": {
    "DriverName": "postgres",
    "DataSource": "postgres://mmuser:MySuperSecretPassword@localhost/mattermost?sslmode=disable\u0026connect_timeout=10\u0026binary_parameters=yes",
    […]
  },



Configure web server
--------------------

.. note::

    Mattermost is running on port 8065.

.. include:: includes/web-backend.rst

Setup daemon
------------

Create ``~/etc/services.d/mattermost.ini`` with the following content:

.. code-block:: ini

 [program:mattermost]
 command=%(ENV_HOME)s/mattermost/bin/mattermost
 autorestart=true
 directory=%(ENV_HOME)s/mattermost


.. include:: includes/supervisord.rst

If it's not in state RUNNING, check your configuration or logfile.

.. code-block:: console

  [isabell@stardust ~]$ less ~/mattermost/logs/mattermost.log


Setup a user
------------

You can now point your browser to your URL and setup a user. Do this immediately after starting the server, as the very first user will be an administrator user.

Further customisation
---------------------

To further customise your configuration, you can open the ``system console`` in your browser and adapt any settings to your wishes. Setting the SMTP server is a good idea.

Update
======

Manual
------

1. Stop your service,
2. Backup your directories
	- ``/home/isabell/mattermost/client/plugins``,
	- ``/home/isabell/mattermost/config``,
	- ``/home/isabell/mattermost/data``,
	- ``/home/isabell/mattermost/logs``
	- ``/home/isabell/mattermost/plugins``
3. Rename/delete your ``/home/isabell/mattermost`` directory.
4. Proceed with the installation steps and restore the ``client/plugins``, ``config``, ``data``, ``logs`` and ``plugins`` directories.
5. Then you can start your service again.

When upgrading to Mattermost 6.4 or newer you need to change the collation of the database:

.. code-block:: console

  [isabell@stardust ~]$ mysql -e "ALTER DATABASE isabell_mattermost COLLATE = utf8mb4_general_ci;"
  [isabell@stardust ~]$

Automatic Update
-----------------

Use the script attached to update Mattermost to the current version. Run the script above the mattermost-root directory.

.. code-block:: shell

	#!/bin/sh
	printf "\n===\nWelcome to updating Mattermost... let us begin!\n===\n"

	printf "\n== Preflight Checks ==\n"
	if [ ! -d "./mattermost" ]; then
	printf "Error: ./mattermost directory does not exist. Are you running in the correct place?. Exit\n"
	exit 1
	else
		printf "Directory ./mattermost found.\n"
	fi

	database=$(cat ./mattermost/config/config.json | jq '.SqlSettings.DataSource' | grep -o '/.*?' | tail -c +2 | head -c -2)
	if [ -z "$database" ]
	then
		printf "\nError: Could not extract the database name from ./mattermost/config/config.json. Maybe the script runs in the wrong directory? Exit 1.\n"
		exit 2
	else
		printf "Using database: $database\n"
	fi

	version=$1
	if [ -z "$version" ]
	then
		printf "No manual version given. Check from official website ...\n"

		tag_name=$(curl --silent --location https://api.github.com/repos/mattermost/mattermost/releases/latest |
		  jq --raw-output '.tag_name')
		version=${tag_name:1} # This will remove the first character `v` from the version tag `v1.2.3`

		printf "Newest Version detected: v$version. For the changes introduced with this update see: https://docs.mattermost.com/install/self-managed-changelog.html\n"
	fi

	usedVersion=$(./mattermost/bin/mattermost version | grep -Po "(?<=Version: )([0-9]|\.)*(?=\s|$)")
	if [ "$usedVersion" = "$version" ]
	then
		printf "\n=== You are up-to-date. No update needed. ===\n"
		exit 0
	fi

	printf "\n== Prepare for takeoff ==\n"

	printf "Downloading Version v$version...\n"
	mattermostfile="mattermost-$version-linux-amd64.tar.gz"
	wget https://releases.mattermost.com/${version}/$mattermostfile

	printf "\nExtracting Version ..."
	tar -xf $mattermostfile --transform='s,^[^/]\+,\0-upgrade,'

	printf "\nCreate Backup of filesystem..."
	backupdir="mattermost-back-$(date +'%F-%H-%M')/"
	cp -ra mattermost/ $backupdir

	printf "\nCreate Backup of database: '$database' ..."
	mysqldump --databases $database > $backupdir/$database.dump.sql

	printf "\n== Take off ==\n"
	printf "\nStop Service - ..."
	supervisorctl stop mattermost

	printf "\nDelete all old files ..."
	find mattermost/ mattermost/client/ -mindepth 1 -maxdepth 1 \! \( -type d \( -path mattermost/client -o -path mattermost/client/plugins -o -path mattermost/config -o -path mattermost/logs -o -path mattermost/plugins -o -path mattermost/data \) -prune \) | sort | xargs rm -r

	printf "\nApply new files ..."
	cp -an mattermost-upgrade/. mattermost/

	printf "\nStart Service again ..."
	supervisorctl start mattermost

	printf "\n== Take off complete ==\n"

	printf "\nClean up files ..."
	rm -r mattermost-upgrade/
	rm -i mattermost*.gz

	printf "\n===\nUpdate completed successfully.\n==="


.. _`Mattermost website`: https://docs.mattermost.com/install/install-tar.html#download
.. _`Mattermost`: https://mattermost.com/


----

Tested with Mattermost 7.7.1 and Uberspace 7.13.0

.. author_list::
