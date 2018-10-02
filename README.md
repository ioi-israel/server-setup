# CMS + Gitolite server setup

This repository contains documentation and files for setting up a server with [CMS](http://cms-dev.github.io/) for training or testing.

The documentation is for Ubuntu 16.04.3 Server 64 bit ([release page](http://releases.ubuntu.com/16.04/); [ISO](http://releases.ubuntu.com/16.04/ubuntu-16.04.3-server-amd64.iso)). The main software to be installed is [CMS 1.3](https://github.com/cms-dev/cms/tree/v1.3) with [custom modifications](https://github.com/ioi-israel/cms/tree/v1.3-israel), and optionally [gitolite](http://gitolite.com) for task development.

## Typical objective
* One public server will be available for contestants. It will always have a running instance of CMS for training, containing only finalized tasks.
* One private server will be available for task developers. It will always have a running instance of CMS for testing, containing tasks in progress. It will host a gitolite system for developers (one repository per task).

## Experimenting in local virtual machines
When getting acquainted with the system for the first time, try everything in virtual machines first, and install things manually rather than using the automation script.

You will need two virtual machines: one for the public server ("training") and one for the private development server ("testing"). It is recommended to name them appropriately, e.g. `local-ioi-testing` and `local-ioi-training`, to distinguish them from the actual servers. You should be able to access them with SSH easily (consider setting up static IPs), and they should have a shared directory (you can simply define a shared directory with the host for each of them, and choose the same path).

Note that some of the installation is the same for both servers. It may be convenient to start with one virtual machine, and duplicate it when they need to diverge.

## Install Ubuntu
* Install a fresh copy of [Ubuntu 16.04.3 Server 64 bit](http://releases.ubuntu.com/16.04/). The default settings should work, except:
    * Choose a hostname (typically `ioi-training` for the public training server and `ioi-testing` for the private testing server).
    * Choose a username such as `ioi`.
    * Mark OpenSSH server for installation.
* If using a local virtual machine, set up the network such that the host can administrate the guests with SSH:
    * Map some host port to guest port 22, if needed.
    * Make the IP static: you can find relevant instructions [here](http://coding4streetcred.com/blog/post/VirtualBox-Configuring-Static-IPs-for-VMs) and [here](https://askubuntu.com/questions/264768/how-to-configure-static-ip-in-ubuntu-running-on-virtual-box). Choose between bridge or host adapter according to your preference and local setup. The important thing is to be able to access the machines via SSH reliably and easily.
* Bring the system up to date:
    ```
    $ sudo apt update && sudo apt full-upgrade
    ```
    Reboot if necessary:
    ```
    $ sudo reboot
    ```
* Enable key-based login with `ssh-copy-id`. Disable password login if needed.
* Customize the environment as needed, for example:
    * Install `zsh` and [`oh-my-zsh`](https://github.com/robbyrussell/oh-my-zsh/):
        ```
        $ sudo apt-get install zsh
        ```
        ```
        $ sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
        ```
        See `custom/zsh/.zshrc` in this repository for a suggested configuration. Note that the plugin [`zsh-syntax-highlighting`](https://github.com/zsh-users/zsh-syntax-highlighting) has to be installed for shell colors, and the Ubuntu `source-highlight` package has to be installed for `less` colors.
    * Use `tmux` or `screen` for managing terminal sessions. See `custom/screen/.screenrc` in this repository for a suggested configuration.
    * See `custom/nano/.nanorc` in this repository for a suggested nano configuration.
* The following packages are needed for our custom repositories.
    ```
    $ sudo apt-get install python-networkx python-pyinotify python-flufl.lock
    ```

    This customization and most of the following steps can be automated with `auto/AutoSetup.py` in this repository. However, when experimenting with these instructions for the very first time, it is instructive to go through them manually.

## Install CMS
* Use the [CMS 1.3 documentation](https://cms.readthedocs.io/en/v1.3/) to install the [customized CMS version](https://github.com/ioi-israel/cms/tree/v1.3-israel). All needed commands are described here.
* Install the required packages:
    ```
    $ sudo apt-get install build-essential openjdk-8-jre openjdk-8-jdk fpc postgresql postgresql-client gettext python2.7 iso-codes shared-mime-info stl-manual cgroup-lite libcap-dev python-dev libpq-dev libcups2-dev libyaml-dev libffi-dev python-pip
    ```
    For C# support, install `mono-mcs` as well. For complete CMS testing, install all other languages:
    ```
    $ sudo apt-get install fp-compiler fp-units-base fp-units-fcl fp-units-misc fp-units-math fp-units-rtl gcj-jdk haskell-platform rustc php7.0-cli php7.0-fpm
    ```
* If relevant, install `nginx`:
    ```
    $ sudo apt-get install nginx-full
    ```
* Clone the CMS repository (we will use `~/Github` as a container directory), and go to the currently used branch of `cms` and of `isolate`:
    ```
    $ mkdir -p ~/Github/ioi-israel
    $ cd ~/Github/ioi-israel
    $ git clone --recursive https://github.com/ioi-israel/cms.git
    $ cd cms
    $ git checkout v1.3-israel
    $ cd isolate
    $ git checkout c8b0eef
    ```
* Run CMS prerequisites:
    ```
    $ cd ~/Github/ioi-israel/cms
    $ sudo ./prerequisites.py install
    ```
    The script will ask whether to add `ioi` to the `cmsuser` group. Answer `Y` to confirm. Log out and back in for the changes to take effect.
* Install CMS Python dependencies. Include the developer requirements for CMS testing.
    ```
    $ sudo pip2 install -r requirements.txt
    $ sudo pip2 install -r dev-requirements.txt
    $ sudo python2 setup.py install
    ```
    **Note:** in the output of the last command, a Python syntax error will be shown for the file `compile-fail.py`. This is normal.
* Create the database, make note of the chosen database username and password. Note there might be issues if the database is not unicode (this is taken care of in the `createdb` line, modify it as needed).
    ```
    $ sudo su - postgres
    $ createuser --username=postgres --pwprompt cmsuser
    $ createdb --username=postgres --owner=cmsuser cmsdb --encoding='UTF8' --locale='en_US.UTF-8' --template=template0
    $ psql --username=postgres --dbname=cmsdb --command='ALTER SCHEMA public OWNER TO cmsuser'
    $ psql --username=postgres --dbname=cmsdb --command='GRANT SELECT ON pg_largeobject TO cmsuser'
    ```
    For CMS testing, create a database `cmsdbfortesting`:
    ```
    $ createdb --username=postgres --owner=cmsuser cmsdbfortesting --encoding='UTF8' --locale='en_US.UTF-8' --template=template0
    $ psql --username=postgres --dbname=cmsdbfortesting --command='ALTER SCHEMA public OWNER TO cmsuser'
    $ psql --username=postgres --dbname=cmsdbfortesting --command='GRANT SELECT ON pg_largeobject TO cmsuser'
    ```
    Don't forget to go back to the normal user after executing these database commands as `postgres`.
* Configure CMS:
    * Use the configuration files in this (`server_setup`) repository under the `cms` directory. Put `cms.conf` and `cms.ranking.conf` in `~/Github/ioi-israel/cms/config`, and `nginx.conf` in `/etc/nginx`. These files are different from the original CMS files (i.e. the `*.conf.sample` files in `cms/config`). The changes are described here:
        * In `nginx.conf`:
            * We have one worker process per CPU core.
            * HTTPS is handled separately, so we commented out the lines that begin with `ssl` and `auth_basic`, and changed port 443 to 80.
            * **Important:** AWS should only be available by accessing the server directly. To administrate remotely, use a tunnel, for example:
                ```
                $ ssh myuser@myserver -L 5000:127.0.0.1:8889 -N
                ```
                where 5000 is the local port, and 8889 is the AWS port on the server. Thus everything about `/aws` and `/rws` in `nginx.conf` is commented out. In any case, make sure the administration is not accessible publicly.
        * In `cms.conf`:
            * We changed the amount of workers to 1. This is normal for a single server with 2 cores.
            *  We changed `max_submission_length` to a more suitable value of 10000000 (approximately 10MB; such files are needed for output-only tasks).
            * We put `127.0.0.1` in `admin_listen_address`.
    * Reload the nginx settings after any modification to its configuration:
        ```
        $ sudo nginx -s reload
        ```
    * Generate a random hex key:
        ```
        $ python -c 'import cmscommon.crypto; print cmscommon.crypto.get_hex_random_key()'
        ```
        Put the key in the `secret_key` field of `cms.conf`.
    * Change the database login information in `cms.conf` to match the ones chosen earlier.
    * In `cms.ranking.conf`, change the login information, then change the `rankings` string in `cms.conf` to match that login. If RWS will not be used, `rankings` should be an empty list (otherwise CMS will try to send the score to a non-existing server, resulting in connection errors). However, do not leave it empty for testing, the proxy service relies on it.
    * Edit the two custom fields at the end of `cms.conf`, called `custom_instructors_path` and `custom_contestants_path`. Each is an absolute path to a directory that will be available for instructors and contestants, respectively. Normally these are `/home/ioi/for-instructors` and `/home/ioi/for-contestants`, respectively.
    * Run the prerequisites again:
        ```
        $ cd ~/Github/ioi-israel/cms
        $ sudo ./prerequisites.py install
        ```
        Confirm overwriting the configuration files for the changes to take effect.
* Initialize the database:
    ```
    $ cmsInitDB
    ```
* Turn off swap. This is necessary because the `isolate` sandbox does not include swap when enforcing the memory limit.
    ```
    $ sudo swapoff -a
    ```
* Run the automatic tests:
    ```
    $ cd ~/Github/ioi-israel/cms
    $ cmsRunTests
    ```
    * This may take a while.
    * If some tests fail, inspect the logs and consider running `cmsRunTests -r` to retry only the failed ones. Consider running `cmsLogService` in another window, to inspect events more closely. Note it should be executed before `cmsRunTests`.
    * If swap is enabled, some out of memory ("oom") tests may fail.
    * The number of workers defined under `cmstestsuite` should match the actual number of workers defined in `cms.conf`. In our custom branch the number of workers is 1.
    * When testing several times, the database may become polluted (which is undesirable, and by itself can cause some tests to fail). Consider dropping the database and initializing it after every testing session.
* If there is nothing important in the database, clean it up:
     ```
    $ cmsDropDB
    $ cmsInitDB
    ```
* Add an AWS administrator:
    ```
    $ cmsAddAdmin <username>
    ```

## Test CMS manually
* Run `cmsLogService` and `cmsAdminWebServer` (use `screen` or `tmux` to keep control of the terminal). Login from a local browser, using an SSH tunnel (described above).
* Use the website to create a user, a contest, and a task with some testcases. Add the user and task to the contest.
* Shut down `cmsAdminWebServer` and run `cmsResourceService -a 1`, where 1 is the contest ID. Now all services are up (including AWS). Login as a contestant in a local browser. Submit a program and make sure everything works correctly (correct output, incorrect output, failed compilation, and so on).
* While testing, always check the `cmsLogService` output for errors, as well as the AWS overview page. There will be internal errors if, for example, the task is missing some parameters, or the scorer program crashed, or some Python package is missing, etc.

## Clone and configure custom repositories
Clone our custom repositories on the server:
```
$ cd ~/Github/ioi-israel
$ git clone https://github.com/ioi-israel/server_utils.git
$ git clone https://github.com/ioi-israel/task_utils.git
$ git clone https://github.com/ioi-israel/task_algorithms.git
$ git clone https://github.com/ioi-israel/contestants_docs.git
```
These scripts should be accessible from the shell, (e.g. `import server_utils` should work). The simplest way to do this is to add `ioi-israel` to the python path in `.zshrc`:
```
export PYTHONPATH=/home/ioi/Github/ioi-israel:$PYTHONPATH
```
Note this already exists in the suggested file `custom/zsh/.zshrc`.

Set up a shared directory for the two servers. This may be a network disk, or (if using VirtualBox with guest additions) a shared directory with the host. Suppose it is is mounted on `/data`.
In `server_utils/config`, make a copy of `config.sample.yaml` called `config.yaml` in the same directory. Make the following changes:
* `general/name` should be a short name for this server, distinguishing it from the other server. Use `training` or `testing` on the appropriate machines.
* `paths/clone_dir`: this should be a path to a directory inside `/data`, where repositories will be cloned for server use. Typically this may be `/data/Clone` (create this directory).
* `paths/requests_dir`: this is a path for git automation (see later). The default should work.
* `paths/locks_file`: this is a path for a lock file, used to prevent collisions in the clone directory. In our example, this should be `/data/Clone/.lock`.
* `locks/lifetime`: number of seconds a directory is allowed to be locked. The default should work.
* `locks/timeout`: number of seconds before a script gives up on trying to obtain the lock. The default should work.
* `requests/cooling_period`: number of seconds the server waits between acting on requests. The default should work.
* `requests/active_contests`: a list of contest paths which are considered active. An active contest is one whose tasks are processed whenever a relevant repository is updated (see later). Assuming you will create a contest called `testing`, the default should work.

## Split the machines
If installing on a local virtual machines, this is the point where training and testing diverge. The training one will contain just CMS, and the testing one will additionally contain gitolite and the surrounding scripts.
Remember to distinguish the machines, because we need to run them simultaneously:
* Change the hostname.
* Change the static IP.
* Change `general/name` in `server_utils/config/config.yaml`.

## Install gitolite
* Gitolite works with SSH keys. Create one locally if needed, and make sure the **private** key file (e.g. `id_rsa`) is secure.
* Install the `gitolite3` package on the testing server:
    ```
    $ sudo apt-get install gitolite3
    ```
    During the installation, give gitolite the **public** key (e.g. `id_rsa.pub`) to be used for administration.
* The `gitolite3` home directory is `/var/lib/gitolite3`, in which the `repositories` directory will contain the data. The tasks input/output data is sometimes large (even when compressed). If there is an external disk intended for large files, replace `repositories` with a symlink.
* Clone the `gitolite-admin` repository to your computer (change the IP address to the testing server).
    ```
    $ git clone ssh://gitolite3@192.168.56.210/gitolite-admin
    ```
    
    This is where you control the configuration of gitolite users and permissions. Pushing this repository activates any changes you make. Read through before pushing.
    * Copy `gitolite/gitolite.conf` from this repository to `gitolite-admin/conf/gitolite.conf`. This file gives admins (you) full permissions, while task developers can only create repositories of the form `tasks/dev-name/task_name`.
    * The server needs to be able to access repositories locally. Therefore we add it as a gitolite admin. Create an SSH key on the testing server, as the normal user (the defaults should work):
        ```
        $ ssh-keygen
        ```
        Add the new public key to `gitolite-admin` in the file `keys/ioi-testing.pub`, and push.
        
        The SSH identity should be added to the shell. The `ssh-agent` plugin can help with `zsh` (see suggested file `.zshrc`).
    * For testing purposes on a local virtual machine, we don't need to add task developers (admin is enough). On an actual server, a new developer's public SSH key should be added to `gitolite-admin/keys` as a file `dev-name.pub`; and their name should be added in `gitolite.conf` where indicated.

Now we will set up the communication between gitolite and CMS.

* Add the regular user to the `gitolite3` group:
    ```
    $ sudo usermod -a -G gitolite3 ioi
    ```
    Log out and back in for the changes to take effect.
* Create the directory `requests` in the `gitolite3` directory. It should be owned by `gitolite3`, and have read and write permissions for the `gitolite3` group.
    ```
    $ sudo su - gitolite3
    $ mkdir requests
    ```
* Configure gitolite to enable custom hooks. Edit `/var/lib/gitolite3/.gitolite.rc`, within the `%RC` block, and uncomment the following line:
    ```
    LOCAL_CODE => "$ENV{HOME}/local",
    ```
    See further details [here](http://gitolite.com/gitolite/cookbook/).
* Put our custom `GitoliteRequest.py` hook in. With the hook, each time a gitolite repository is updated by a task developer, a corresponding request file is created in the `requests` directory.
    ```
    $ sudo su - gitolite3
    $ mkdir -p local/hooks/common
    $ cd local/hooks/common
    $ cp /home/ioi/Github/ioi-israel/server_utils/auto/GitoliteRequest.py post-receive
    $ chmod ug+x post-receive
    ```
* Run the request handler:
    ```
    $ python ~/Github/ioi-israel/server_utils/auto/RequestHandler.py
    ```
    It should say that it is watching the requests directory for changes. When a request arrives signifying that a task repository has been updated, the request handler will process the task (generate testcases etc.) and update the contest in CMS. Leave the script running forever, except when you specifically need to prevent this from happening.

## Testing gitolite
* Notes on permissions and VirtualBox:
    * Part of task processing involves creating a directory writable by anyone (`0777`) in the task's directory, inside the clone directory. When using VirtualBox shared directories, it might not allow using `chmod`. This can be worked around tentatively (not to be used on production machines) by mounting the directory with full permissions. For example:
        ```
        $ sudo mount -t vboxsf -o uid=1000,gid=100,dmode=777,fmode=777 data /data
        ```
    * The locking mechanism uses `python-flufl.lock`, which internally relies on creating hard links. This may not work in VirtualBox shared directories. One possible workaround (tentative, not to be used on production machines) is to change the lock path in `config.yaml` to something local. This of course disables the locking, because one server will not see the other's lock. You will have to be careful not to create collisions (a collision may happen when simultaneously `ioi-testing` is processing an updated repository and `ioi-training` is fetching contest/task/user data).
* Make sure the request handler is running, see above.
* Clone a testing contest repository from gitolite to your computer.
    ```
    $ git clone ssh://gitolite3@192.168.56.210/contests/testing
    ```
    Put a `module.yaml` file in it, in a format similar to `server_utils/templates/contest_module.yaml`. The users file should be a YAML file inside `users`, e.g. `users/testing-users.yaml`. Edit the tasks list to include just one task, called `task1`, with path `tasks/joe/task1`.
    
    When you push, the request handler might ask (just once per installation) to confirm the authenticity of its own fingerprint. This is because it uses git with SSH to clone the repository to the clone directory.

    Then, the request handler will complain that the specified users file doesn't exist yet. This is normal.
* Clone the `users` repository from gitolite. Put a new YAML file in it, in a format similar to `server_utils/templates/users_file.yaml`. Give it the same name that the contest is expecting.

    These will be the users who can access CMS as contestants and make submissions. Add some dummy users, and also an unrestricted user called "`autotester`". It will be used for automated submissions.

    When you push, the request handler will complain that `task1` does not contain a module yet. This is normal.
* Create `task1` on your computer:
    ```
    $ git clone ssh://gitolite3@192.168.56.210/tasks/joe/task1
    ```
    Put a valid task `module.py` in it, and include any necessary additional files. See `task_utils/templates/documented_template.py`.

    When you push, the task will be processed, because it belongs to a contest which was defined as active in `config.yaml`. All the test data will be generated into `task1/auto.gen` in the clone directory. If there are no errors, CMS will be updated, and automated submissions (if any) will be inserted. Run `cmsResourceService`, log in as a contestant, and test.

    Future pushes to the active contest, to any of its tasks, or to the users repository will trigger another update. Note that this can be done while `cmsResourceService` is running.
* Note about users: due to a technical limitation, removing and modifying users in the CMS database cannot be done via the automation process (only adding users is supported). If users need to be modified or removed (which is rare), perform these actions first through AWS, and then in the `users` repository too (for consistency).

Now we have a working environment for development and testing. Task developers can create their own repositories independently. Once they want to test on CMS, they let the admin know, and the admin adds the new task to `contests/testing/module.yaml`. Test that this process works.

## From testing to training
[Todo]


# Usage and maintenance

## Periodic maintenance checklist
* Free up space:
    * Check the free space with `df -h`.
    * Check the contents of `/tmp` for sandboxes or other temporary files that may be deletable. Consider removing all directories of the form `/tmp/tmp*` when the server is not in use.
    * Check the contents of `/var/local/cache/cms` and `/var/local/log/cms`. Consider clearing them when the server is not in use.
* Upgrade the operating system. Upgrade `ioi-testing` before `ioi-training`, to handle any breakages more easily.

## Pre-season checklist
* Make sure the previous season is fully backed up, then clean the database.
* Create a new users file, and update `server_utils/config/config.yaml` accordingly.

## Post-season checklist
* Make sure the season is fully backed up, then clean the database.
* If a newer stable version of CMS is out, test it and consider upgrading.


# Todo

## General todo
* Complete documentation on working with CMS + gitolite.
* Collect auxiliary scripts into the `server_utils` repository, refer to it here and include it in the automatic script.
* Complete documentation on interacting between the public and private servers.
* Complete the `task_utils` repository to make life easy for task developers.
* Add script to generate users and passwords from a text file.
* Add more checklists, including troubleshooting tips.
* Automatic attachments for output only tasks.
* Force TwoSteps tasks to have:
    * `manager.cpp`
    * `manager.h`
    * `encoder.h`
    * `decoder.h`
* In OutputOnly tasks, automatically zip the input files and include it as an attachment. Consider running the files through `unix2dos`, for contestants who use Windows.
* Documentation about requests. Note the `GitoliteRequest.py` script needs to know the path to the requests directory separately, because it is run by a different user. Add the main user to the `gitolite3` group.
* Documentation about NFS locks and safety, both in the requests directory and in the repositories directory.
* Documentation about maintenance mode, including `public/images/stop-sign.svg`.
* Lock down read permissions for `/var/lib/gitolite3`.
* Documentation about passive analysis mode (year >= 3000).
* Documentation about ranking, including task names that start with a dot (home training), and crontab.

## Automation todo
* Replace home directory inside `.zshrc` with real home directory.
* Add a `SafeImport` alias in `.zshrc` for importing a contest on the training server.
* Clone `task_utils` and `server_utils` into `ioi-israel`.
* Create `config.yaml` from the sample in `server_utils`.
* Start `ssh-agent` and use `ssh-keygen` in `ioi-testing`, for cloning the repositories locally.
* Add the main user to the `gitolite3` group.
* Clone `contestants_docs` into `~/public/docs`.
* Clone `task_algorithms`.
