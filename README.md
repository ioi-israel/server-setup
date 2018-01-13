# CMS + Gitolite server setup

This repository contains documentation and files for setting up a server with [CMS](http://cms-dev.github.io/) for training or testing.

The documentation is for Ubuntu 16.04.3 Server 64 bit ([release page](http://releases.ubuntu.com/16.04/); [ISO](http://releases.ubuntu.com/16.04/ubuntu-16.04.3-server-amd64.iso)). The main software to be installed is [CMS 1.3](https://github.com/cms-dev/cms/tree/v1.3) with [custom modifications](https://github.com/ioi-israel/cms/tree/v1.3-israel), and optionally [gitolite](http://gitolite.com) for task development.

## Typical objective
* One public server will be available for contestants. It will always have a running instance of CMS for training, containing only finalized tasks.
* One private server will be available for task developers. It will always have a running instance of CMS for testing, containing tasks in progress. It will host a gitolite system for developers (one repository per task).

## Install Ubuntu
* Install a fresh copy of [Ubuntu 16.04.3 Server 64 bit](http://releases.ubuntu.com/16.04/). The default settings should work, except:
    * Choose a hostname (typically `ioi-training` for the public training server and `ioi-testing` for the private testing server).
    * Choose a username such as `ioi`.
    * Mark OpenSSH server for installation.
* If using a local virtual machine, set up the network such that the host can administrate the guest with SSH (for example, map some host port to guest port 22).
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
* Clone the CMS repository (we will use `~/Github` as a container directory):
    ```
    $ mkdir -p ~/Github/ioi-israel
    $ cd ~/Github/ioi-israel
    $ git clone --recursive https://github.com/ioi-israel/cms.git
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
    $ psql --username=postgres --dbname=cmsdbfortesting --command='GRANT SELECT ON pg_largeobject TO cmsuser'"
    ```
* Configure CMS:
    * Use the sample configuration files in this repository under `cms`. Put `cms.conf` and `cms.ranking.conf` in `~/Github/ioi-israel/cms/config`, and `nginx.conf` in `/etc/nginx`. The changes from the original CMS files are described here.
    * Change `nginx.conf` according to the desired settings.
        * If HTTPS is handled externally, comment out the lines that begin with `ssl` and `auth_basic`, and change port 443 to 80.
        * When running CMS (later), if AWS isn't accessible from within the server, try adding `allow <explicit ip>` below `allow 127.0.0.1`. Reload the nginx settings after any such modification.
            
            **Important:** AWS should only be available by accessing the server directly. To administrate remotely, use a tunnel, for example:
            ```
            $ ssh myuser@myserver -L 5000:127.0.0.1:8889 -N
            ```
            where 5000 is the local port, and 8889 is the AWS port on the server.
    * In `cms.ranking.conf`, change the login information.
    * In `cms.conf`:
        * Change the `rankings` string to match the login from `cms.ranking.conf`.
        * Change the database login information to match the ones chosen earlier.
        * Change the amount of workers if needed. If there is only one server, the number of workers should probably be 1.
        *  Change `max_submission_length` to a more suitable value, like 10000000 (approximately 10MB; such files are needed for output-only tasks).
        * Generate a random hex key:
            ```
            $ python -c 'import cmscommon.crypto; print cmscommon.crypto.get_hex_random_key()'
            ```
            Put the key in the `secret_key` field.
        * Put `127.0.0.1` in `admin_listen_address`.
        * Add two custom fields, `custom_instructors_path` and `custom_contestants_path`. Each is an absolute path to a directory that will be available for instructors and contestants, respectively.
    * Run the prerequisites again:
        ```
        $ cd ~/Github/ioi-israel/cms
        $ sudo ./prerequisites.py install
        ```
* Initialize the database:
    ```
    $ cmsInitDB
    ```
* Turn off swap. This is necessary because the `isolate` sandbox does not include swap when enforcing the memory limit.
    ```
    $ sudo swapoff -a
    ```
* Test CMS:
    ```
    $ cd ~/Github/ioi-israel/cms
    $ cmsRunTests
    ```
    * This may take a while.
    * If some tests fail, inspect the logs and consider running `cmsRunTests -r` to retry only the failed ones.
    * If swap is enabled, some out of memory ("oom") tests may fail.
    * The number of workers defined under `cmstestsuite` should match the actual number of workers defined in `cms.conf`. In our custom branch the number of workers is 1.
    * When testing several times, the database may become polluted (which is undesirable, and by itself can cause some tests to fail). Consider dropping the database and initializing it after every testing session.
* Add an AWS administrator:
    ```
    $ cmsAddAdmin <username>
    ```

## Test CMS
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

## Automation todo
* Replace home directory inside `.zshrc` with real home directory.
* Add `ioi-israel` directory to `PYTHONPATH` in `.zshrc`.
* Clone `task_utils` and `server_utils` into `ioi-israel`.
* Create `config.yaml` from the sample in `server_utils`.
