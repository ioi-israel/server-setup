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
* Install convenience packages for task development:
    ```
    $ sudo apt-get install python-networkx
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
* Run `cmsLogService` and `cmsAdminWebServer` (use `screen` or `tmux` to keep control of the terminal). Login from a local browser.
* Use the website to create a user, a contest, and a task with some testcases. Add the user and task to the contest.
* Shut down `cmsAdminWebServer` and run `cmsResourceService -a 1`, where 1 is the contest ID. Now all services are up (including AWS). Login as a contestant in a local browser. Submit a program and make sure everything works correctly (correct output, incorrect output, failed compilation, and so on).
* While testing, always check the `cmsLogService` output for errors, as well as the AWS overview page. There will be internal errors if, for example, the task is missing some parameters, or the scorer program crashed, or some Python package is missing, etc.

## Install gitolite
* Gitolite works with SSH keys. Create one locally if needed, and make sure the **private** key file (e.g. `id_rsa`) is secure.
* Install the `gitolite3` package:
    ```
    $ sudo apt-get install gitolite3
    ```
    During the installation, give gitolite the **public** key (e.g. `id_rsa.pub`) to be used for administration.
* The `gitolite3` home directory is `/var/lib/gitolite3`, in which the `repositories` directory will contain the data. The tasks input/output data is sometimes large (even when compressed). If there is an external disk intended for large files, replace `repositories` with a symlink.


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
* Add `PYTHONPATH` as a requirement.
* Add script to generate users and passwords from a text file.
* Add more checklists, including troubleshooting tips.
* Documentation of `config.yaml` in `server_utils`.
* Automatic attachments for output only tasks.
* Force TwoSteps tasks to have:
    * `manager.cpp`
    * `manager.h`
    * `encoder.h`
    * `decoder.h`
* In OutputOnly tasks, automatically zip the input files and include it as an attachment. Consider running the files through `unix2dos`, for contestants who use Windows.
* Documentation about requests. Note the `GitoliteRequest.py` script needs to know the path to the requests directory separately, because it is run by a different user. Add the main user to the `gitolite3` group.
* Documentation about NFS locks and safety, both in the requests directory and in the repositories directory.
* Documentation about adding an SSH key, and starting ssh-agent, for internally cloning repositories.
* Documentation about `gitolite/gitolite.conf`.
* Documentation about maintenance mode, including `public/images/stop-sign.svg`.
* Lock down read permissions for `/var/lib/gitolite3`.
* Documentation about passive analysis mode (year >= 3000).
* Documentation about ranking, including task names that start with a dot (home training), and crontab.

## Automation todo
* Replace home directory inside `.zshrc` with real home directory.
* Add `ioi-israel` directory to `PYTHONPATH` in `.zshrc`.
* Add a `SafeImport` alias in `.zshrc` for importing a contest on the training server.
* Clone `task_utils` and `server_utils` into `ioi-israel`.
* Create `config.yaml` from the sample in `server_utils`.
* Start `ssh-agent` and use `ssh-keygen` in `ioi-testing`, for cloning the repositories locally.
* Add the main user to the `gitolite3` group.
* Clone `contestants_docs` into `~/public/docs`.
* Clone `task_algorithms`.
