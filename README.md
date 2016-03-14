# OSP Deployments

This repository contains a set of Ansible playbooks that automate the process of deploying the code for the [Open Syllabus Project] (https://github.com/davidmcclure/open-syllabus-project) - both the metadata extraction rig and the public-facing web application. These playbooks can be used to configure a local development environment managed by Vagrant, or to deploy public-facing instances on EC2.

#### Set up a local development environment

1. Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/downloads.html).

1. Install the [Vai](https://github.com/MatthewMi11er/vai) plugin for Vagrant with:

  `vagrant plugin install vai`

1. Symlink `config/ansible.vagrant.cfg` -> `ansible.cfg`.

1. Copy `/vars/local.changeme.yml` -> `vars/local.yml`.

1. In `vars/local.yml`, set a value for the `osp_db_pass` variable, which will be used as the password for the main Postgres database.

1. For `osp_sync_code` and `osp_sync_data`, enter paths to directories on the local filesystem that will by synced with the Vagrant VM. Eg, on a Mac, something like:

  ```yaml
  osp_sync_code: /Users/davidmcclure/Projects/osp-vagrant-code
  osp_sync_data: /Users/davidmcclure/Projects/osp-vagrant-data
  ```

  These directories don't need to exist - they'll be created automatically when the VM is started.

1. Start the Vagrant box with:

  `vagrant up`

1. Then, provision the box with:

  `ansible-playbook deploy-workers.yml`

  On the first run, this will take 20-30 minutes on most systems, since the pip install has to compile a number of very large packages (`scipy`, `pgmagick`).

1. Once the playbooks run, log into VM with:

  `vagrant ssh`

1. And start the testing Elasticsearch process server:

  `sudo supervisorctl start es-test`

1. Wait ~5s for Elasticsearch to start, and then change into `/home/vagrant/osp` and run the test suite:

  `py.test`

1. If this passes, the environment is fully configured and ready for work. Any changes to the code made in the synced directory (set by `osp_sync_code`) will be automatically propagated to the VM, and vice versa.

1. At this point, if you're working with one of the public data dumps, just move the dump into the synced `osp_sync_data` directory, and then, from the Vagrant VM, use `pg_restore` to source in the data:

  `pg_restore /osp/osp-public.sql -d osp -U osp -v`

  This will take 30-40 minutes. (Ignore any errors here - these are fixed when the foreign key constraints are rebuilt at the end.) Once it's complete, hop into `psql`, and you should be able to interact with the data:

  ```sql
  > psql osp -U osp

  psql (9.5.1)
  Type "help" for help.

  osp=> select count(*) from document;
    count
  ---------
   1415005
  (1 row)
  ```

#### Make a wheelhouse to speed up deployments

The slowness of the pip install is a real drag, especially when it comes time to put up a big set of workers on EC2. To get around this, we can build a "wheelhouse" from the local Vagrant VM, which contains pre-built binaries for Ubuntu. Then, when deploying new servers (either on EC2 or locally), we can tell the provisioning scripts to use these binaries, instead of recompiling everything from scratch.

1. Log into the Vagrant box with:

  `vagrant ssh`

1. Change down into `/home/vagrant/osp`, and run:

  `pip wheel -r requirements.txt -w wheelhouse`

1. Tar up the wheelhouse:

  `tar -cvzf wheelhouse.tar.gz wheelhouse`

1. Move the tarball into the synced directory:

  `mv wheelhouse.tar.gz /vagrant`

Now, any future deployments will automatically detect the wheelhouse and deploy it to the remote server.
