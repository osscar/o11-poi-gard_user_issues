Installation/migration notes for odoo11 running on an Ubuntu cloud instance:


    1. Connect to your server via SSH. Run apt install with sudo and not as root.

    2. Update the system packages.

    sudo apt update

    3. Install Git.

    sudo apt install git -y
    
    4. Install Python 3.7
    
    sudo apt install software-properties-common -y
    sudo add-apt-repository ppa:deadsnakes/ppa -y
    sudo apt update
    sudo apt install python3.7 python3.7-dev
    
    5. Install Pip.
    
    sudo apt install pip3
    
    6. Install virtualwrapper for easy virtualenv usage and maintenance
    
    cd ~/ # cd in to home folder logged in as user that will maintain the virtalenv's
    pip install virtualenvwrapper # no sudo
    
    Add three lines to your shell startup file (.bashrc, .profile, etc.) 
    to set the location where the virtual environments should live, 
    the location of your development project directories, and the location 
    of the script installed with this package:

    export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3 # establish this for first sourcing (will get replaced in next step)
    export WORKON_HOME=$HOME/.virtualenvs
    export PROJECT_HOME=$HOME/Devel
    source /usr/local/bin/virtualenvwrapper.sh
    
    source ~/.bashrc # after editing, reload the startup file 
    
    Edit again and replace VIRTUALENVWRAPPER_PYTHON var with:
    
    export VIRTUALENVWRAPPER_PYTHON="$(command \which python3)" # dynamic python command
    source ~/.bashrc # reload startup file
    
    Usage:
    
    1. Run: workon
    A list of environments, empty, is printed.
    
    2. Run: mkvirtualenv temp # temp for example
    A new environment, temp is created and activated.
    
    chown -R user:group /path_to_virtual_env # give ownership to the user/group that will use the virtualenv
    chmod -R 000 /path_to_virtual_env # with appropriate permission values
    
    3. Run: workon temp
    This time, the temp environment is included.
    
    4. Run: deactivate
    The virtualenv wil deactivate.
    
    * Run: lsvirtualenv 
    A list of available virtualenv's in $HOME folder, is printed.
    
    * Run: python -V
    The python version used by the virtualenv is printed.
    
    * Run: rmvirtualenv name_of_your_env
    This will delete the name_of_your_env virtualenv.

    7. Install other odoo build required dependencies.

    sudo apt update
    sudo apt install git build-essential python3.7-setuptools node-less wget \
    libjpeg-dev zlib1g-dev libpq-dev libxslt-dev libzip-dev libldap2-dev \
    libsasl2-dev libtiff5-dev libjpeg8-dev libopenjp2-7-dev liblcms2-dev \
    libwebp-dev libharfbuzz-dev libfribidi-dev libxcb1-dev

    8. Create odoo user to manage odoo.

    sudo useradd -m -d /opt/odoo -U -r -s /bin/bash odoo
    sudo visudo # edit sudoers file to allow local passwordless login to user
    
    Append the following after #includedir /etc/sudoers.d line:
    
    %odoo ALL=NOPASSWD: /bin/su - odoo # this allows passwordless login
                                       # to login to odoo user include the other users in odoo group
    sudo passwd -d odoo # delete current password (if any)
    
    9. Install wkhtmltox 0.12.5-1 for ubuntu (this will work fine with odoo 11 although 0.12.1x is the default version).

    wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb
    sudo apt install ./wkhtmltox_0.12.5-1.bionic_amd64.deb

    10. Install odoo python requirement (on virtualenv).

    su - odoo
    workon name_of_virtualenv
    cd /path_to_odoo_install # usually /opt/odoo
    pip install wheel
    pip install -r requirements.txt # do this for any additional odoo repositories that contain a requirement file

    11. Create and modify odoo.conf file
    sudo touch /etc/odoo11-server.conf
    sudo vi /etc/odoo11-server.conf

    The file should look something like this (adapt to running environment):
    
    
Tunning Up Odoo In Multiprocessing Server
Odoo October 8, 2021 0 Terry

By default, Odoo is working in multithreading mode. For production use, it is recommended to use the multiprocessing server as it increases stability, makes somewhat better use of computing resources and can be better monitored and resource-restricted.
Worker number calculation

    Rule of thumb : (#CPU * 2) + 1
    Cron workers need CPU
    1 worker ~= 6 concurrent users

Memory Size Calculation

    We consider 20% of the requests are heavy requests, while 80% are simpler ones
    A heavy worker, when all computed field are well designed, SQL requests are well designed, is estimated to consume around 1GB of RAM
    A lighter worker, in the same scenario, is estimated to consume around 150MB of RAM

Needed RAM = #worker * ( (light_worker_ratio * light_worker_ram_estimation) + (heavy_worker_ratio * heavy_worker_ram_estimation) )

If you do not know how many CPUs you have on your system, use the following grep command:

# grep -c ^processor /proc/cpuinfo

Let’s say you have a system with 4 CPU cores, 8 GB of RAM memory, and 30 concurrent Odoo users.

    30 users / 6 = 5 (5 is theoretical number of workers needed )
    (4 * 2) + 1 = 9 ( 9 is the theoretical maximum number of workers)

Based on the calculation above, you can use 5 workers + 1 worker for the cron worker that is a total of 6 workers.

Calculate the RAM memory consumption based on the number of workers:

    RAM = 6 * ((0.8150) + (0.21024)) ~= 2 GB of RAM

The calculation shows that the Odoo installation will need around 2GB of RAM.

To switch to multiprocessing mode, open the configuration file and append the calculated values:

Edit /etc/odoo-server.conf
# sudo nano /etc/odoo-server.conf
--start--
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200
max_cron_threads = 1
workers = 5
--end--

Restart the ODOO server to take effect.
# sudo service odoo-server restart
   
   
