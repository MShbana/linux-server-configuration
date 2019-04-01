# Linux Server Configuration
Third required project for the [Full Stack Web Developer Nanodegree][link_1].

## Requirements to SSH to the Ubuntu Server:
1. **Ubuntu Instance's Static IP Address:** 52.28.246.104
2. **SSH Port:** 2200


## URI to the Hosted Web Application

#### http://mymovies.cf


## Summary of the software installed
1. `python3-pip`
2. `python3-venv`
3. `apache2`
4. `libapache2-mod-wsgi-py`
5. `postgresql`
6. `psycopg2`, `psycopg2-binary` inside the virtual envrironement of the item-catalog project. The requirements.txt file is included in the repo.


## Summary of the configuration steps and changes

- Download the default, automatically created, **Private/Public Key Pair** from **AWS LightSail Instance** page and change its permission to be **read/write** by the local machine's owner only: `chmod 600 /path/to/LightSail_Default_key.pem`.

- SSH to the Instance using the downloaded **default key pair** and the default **ubuntu** username: `ssh -p 22 -i /path/to/LightSail_Default_key.pem ubuntu@<instance's_Static_or_Public_IP_address>`.

- Create two users with **sudo** privileges:
    - Create user: `sudo adduser <user_name>`.
    - Add user to the sudoers group: `sudo adduser <user_name> sudo`.

- Create an **SSH Key pair for each user** from within the local machine &mdash; For each user:
    - **In the remote server:**
        - Inside `/home/<user_name>/` &mdash; create a directory **.ssh** and a file **.ssh/authorized_keys**.
        - Give **read/write/execute** privileges for the owner for the **.ssh** directory: `chmod 700 /home/<user_name>/.ssh`.
        - Give **read/write** privileges for the owner for the **.ssh directory descendants**: `chmod 600 /home/<user_name>/.ssh/*`.
    - **In the local machine**:
        -  Create a key pair: `ssh-keygen`.
        - Choose the directory and name in which you want to store the **Private/Public key pair**. The one I have chosen to use is the default: `/home/<user-name>/.ssh/<key_file_name>`
        - Securely copy the user's generated **Public Key** to the remote machine's **authorized_keys** file: `scp -P 22 /home/<user_name>/.ssh/<key_file_name>.pub <user_name>@<instance's_Static_or_Public_IP_address>:/home/<user_name>/.ssh/authorized_keys`.

- SSH to the remote server, using **Public Key Authentication**:
    `ssh -p 22 -i /home/<user_name>/.ssh/<key_file_name> <user_name>@<instance's_Static_or_Public_IP_address>`

- Update the remote server to have the latest packages:
    - `sudo apt-get update`  
    - `sudo apt-get upgrade`

- Inside the **/etc/ssh/sshd_config** file &mdash; update the following lines as follows:
    - **Change the SSH Port:** Change `#Port 22` to `Port 2200`.
    - **Permit Log in by the Root User:** `PermitRootLogin no`.
    - **Stop SSHing to the remote server using Password Authentication:** `PasswordAuthentication no`.
    - **Restart the SSH service:** `sudo service restart ssh` or `sudo systemctl restart ssh`.

- Configure the **Uncomplicated FireWall (UFW)** to only accept requests on the ports **123 (NTP)**, **80 (HTTP)** and **2200 (SSH)**:
    - `sudo ufw default deny incoming`
    - `sudo ufw default allow outgoing`
    - `sudo ufw allow ntp`
    - `sudo ufw allow http`
    - `sudo ufw allow 2200/tcp`
    - `sudo ufw enable`

- Install the required packages:
    - `sudo apt-get install python3-pip`
    - `sudo apt-get install python3-venv`
    - `sudo apt-get install apache2`
    - `sudo apt-get install libapache2-mod-wsgi-py`
    - `sudo apt-get install postgresql postgresql-contrib`

- Inside the **/var/www/html/** directory:
    - Change the directory permissions:
        1. Allow Apache access:
            ```
            sudo chgrp -R www-data /var/www/html
            sudo find /var/www/html -type d -exec chmod g+rx {} +
            sudo find /var/www/html -type f -exec chmod g+r {} +
            ```
        2. Give the owner **read/write** privileges and permit directory access to other users:
            ```
            sudo chown -R <user_name> /var/www/html/
            sudo find /var/www/html -type d -exec chmod u+rwx {} +
            sudo find /var/www/html -type f -exec chmod u+rw {} +
            ```

    - `git clone` the **item-catalog** project.
    - `nano setup.wsgi` from within the **cloned item-catalog** directory:
        ```
        import sys
        sys.path.insert(0, '/var/www/html/movie_catalog')
        from setup import app as application
        ```

    - `mkdir venvs` inside the **item-catalog** directory.
    - `cd venvs`
    - `python3 -m venv venv`
    - `source venv/bin/activate`
    - `pip install ../requirements.txt`

- Create a **movie_catalog.conf** in the **/etc/apache2/sites-available/** directory, with the following content:

    ```
    <VirtualHost *:80>
        ServerName mymovies.cf
        WSGIDaemonProcess setup user=www-data group=www-data threads=5 python-home=/var/www/html/movie_catalog/venvs/venv
        WSGIScriptAlias / /var/www/html/movie_catalog/setup.wsgi
        <Directory /var/www/html/movie_catalog>
            WSGIProcessGroup setup
            WSGIApplicationGroup %{GLOBAL}
            Require all granted
        </Directory>
    </VirtualHost>

    ```

- Enable the new **Apache Configuration**: `sudo a2ensite /etc/apache2/sites-available/movie_catalog.conf`.

- Restart the **Apache Server**: `sudo systemctl restart apache2` or `sudo apache2ctl restart`.

- Configure the postgres database:    
    - `sudo su - postgres`
    - `psql`
    - `CREATE DATABASE movie_catalog;`
    - `CREATE USER catalog WITH PASSWORD '<password>';`
    - `ALTER ROLE catalog SET client_encoding TO 'utf8';`
    - `ALTER ROLE catalog SET default_transaction_isolation TO 'read committed';`
    - `ALTER ROLE catalog SET timezone TO 'UTC';`
    - `GRANT ALL PRIVILEGES ON DATABASE movie_catalog TO catalog;`
    - `\q`
    - `exit`
    - To login to the created database again: `psql postgresql://<database_user>:<user_password>@localhost/<database_name>`


- Create a **config.json** file in the **/etc/** directory &mdash; whose content will be used inside the **setup.py** file &mdash; with the following variables and their values:
    - SECRET_KEY
    - DATABASE_USER
    - DATABASE_USER_PASS
    - DATABASE_HOSTNAME
    - DATABASE_NAME


- For problems with the syntax of the **/etc/apache2/sites-available/movie_catalog.conf** file:
    - `cd /etc/apache2`
    - `sudo apache2ctl configtest`
    - Fix the syntax errors, then restart the server: `sudo apache2ctl restart`

- For restarting the **Apache server**, choose one of the following methods:
    1. `sudo apache2ctl stop`, then `sudo apache2ctl start`
    2. `sudo apache2ctl restart`
    3. `sudo systemctl restart apache2`



[//]:  # (Links and images relative paths)

[link_1]: <https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004>
