# Overview
These instructions explain how to create a three-node cluster:

| Node name | Purpose | Username (and password) |
| --- | --- | --- |
| `airflow` | Host Airflow and Trino | `airflow` |
| `hdfs-1` | Host HDFS | `hdfs` |
| `hdfs-2` | Host HDFS | `hdfs` |

# Setup VMs
1. Install Vagrant.
2.  Create a file named `Vagrantfile` with the following content, to create the three Ubuntu VMs. Adjust RAM and CPUs depending on the host machine power:
    ```ruby
    BOX_IMAGE = "generic/ubuntu2004"
    NODE_COUNT = 2

    Vagrant.configure("2") do |config|
      config.vm.define "airflow" do |subconfig|
        subconfig.vm.box = BOX_IMAGE
        subconfig.vm.hostname = "airflow"
        subconfig.vm.network :private_network, ip: "10.0.0.10"
        subconfig.vm.boot_timeout = 600
      end

      (1..NODE_COUNT).each do |i|
        config.vm.define "hdfs-#{i}" do |subconfig|
          subconfig.vm.box = BOX_IMAGE
          subconfig.vm.hostname = "hdfs-#{i}"
          subconfig.vm.network :private_network, ip: "10.0.0.#{i+10}"
          subconfig.vm.boot_timeout = 600
        end
      end

      config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", 8096]
        vb.customize ["modifyvm", :id, "--cpus", 2]
      end

      config.vm.provision "shell", inline: <<-SHELL
      apt-get install -y avahi-daemon libnss-mdns build-essential
      SHELL
    end
    ```
3. Run `vagrant up`
4. To SSH into the nodes (in separate terminals):
    ```bash
    ssh -i .vagrant/machines/airflow/virtualbox/private_key vagrant@10.0.0.10 #airflow
    ssh -i .vagrant/machines/hdfs-1/virtualbox/private_key vagrant@10.0.0.11 #hdfs-1
    ssh -i .vagrant/machines/hdfs-2/virtualbox/private_key vagrant@10.0.0.12 #hdfs-2
    ```


# Install Airflow
On the `airflow` node:
1. Create a `virtualenv` to run Python from:
    ```bash
    export PATH=$PATH:~/.local/bin
    sudo apt-get install python3-pip
    pip3 install virtualenv
    virtualenv venv
    source venv/bin/activate
    ```

2. Install Airflow 1.10.14 (from https://airflow.apache.org/docs/apache-airflow/stable/installation.html):
    ```bash
    export AIRFLOW_HOME=~/airflow
    AIRFLOW_VERSION=1.10.14
    PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
    CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
    pip install "apache-airflow[postgres]==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}" --use-deprecated legacy-resolver
    ```

3. Configure Airflow (from http://airflow.apache.org/docs/apache-airflow/stable/start.html):
    ```bash
    airflow initdb
    airflow users create --username admin --firstname Ben --lastname Watson --role Admin --email benwatson528@gmail.com
    # Enter password "admin"
    ```


# Manage Airflow's services via Supervisor
1. Run Airflow services persistently by installing Supervisor to manage the webserver and scheduler:
    ```bash
    sudo apt-get install supervisor
    ```

2. Create Supervisor jobs by placing the following files from `/scripts` into the following destinations on the `airflow` node:
   
   | `/scripts` file | Destination |
   | --- | --- |
   | `airflow_webserver.sh` | `~/airflow/scripts/` |
   | `airflow_scheduler.sh` | `~/airflow/scripts/` |
   | `airflow_webserver.conf` | `/etc/supervisor/conf.d/` |
   | `airflow_server.conf` | `/etc/supervisor/conf.d/` |


4. Start the Supervisor processes:
    ```bash
    sudo supervisorctl reread
    sudo supervisorctl update
    ```
