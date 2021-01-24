# Overview
These instructions explain how to create a three-node cluster running the following services, suited for use as a data ingestion pipeline:
  - Python 3.8.5,
  - Airflow 2.0.0,
  - Hadoop 3.3.0,
  - HDFS, 
  - Hive,
  - Trino. 

The setup process is currently manual (using Vagrant to create the base images).

The nodes are:

| Node name | Purpose | Username |
| --- | --- | --- |
| `airflow` | Airflow and Trino | `airflow` |
| `hdfs-1` | HDFS | `hdfs` |
| `hdfs-2` | HDFS | `hdfs` |


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
    # airflow
    ssh -i .vagrant/machines/airflow/virtualbox/private_key vagrant@10.0.0.10
    #hdfs-1
    ssh -i .vagrant/machines/hdfs-1/virtualbox/private_key vagrant@10.0.0.11
    #hdfs-2
    ssh -i .vagrant/machines/hdfs-2/virtualbox/private_key vagrant@10.0.0.12
    ```


# Install Airflow
On the `airflow` node:
1. Create a `virtualenv` to manage Python:
    ```bash
    export PATH=$PATH:~/.local/bin
    sudo apt-get install -y python3-pip python3-virtualenv
    virtualenv venv
    source venv/bin/activate
    ```

2. Install Airflow (from https://airflow.apache.org/docs/apache-airflow/stable/installation.html):
    ```bash
    export AIRFLOW_HOME=~/airflow
    AIRFLOW_VERSION=2.0.0
    PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
    CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
    pip install "apache-airflow[postgres]==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
    ```

3. Configure Airflow (from http://airflow.apache.org/docs/apache-airflow/stable/start.html):
    ```bash
    airflow db init
    airflow users create --username admin --firstname Ben --lastname Watson --role Admin --email benwatson528@gmail.com
    # Enter password "admin"
    ```


# Manage Airflow's services via Supervisor
1. Run Airflow services persistently by installing Supervisor to manage the webserver and scheduler:
    ```bash
    sudo apt-get install -y supervisor
    ```

2. Create Supervisor jobs by placing the following files from `/scripts` into the following destinations on the `airflow` node:
   
   | `/scripts` file | Destination |
   | --- | --- |
   | `airflow_webserver.sh` | `~/airflow/scripts/` |
   | `airflow_scheduler.sh` | `~/airflow/scripts/` |
   | `airflow_webserver.conf` | `/etc/supervisor/conf.d/` |
   | `airflow_server.conf` | `/etc/supervisor/conf.d/` |

3. Make the two scripts executable:
    ```bash
    chmod +x ~/airflow/scripts/airflow_scheduler.sh
    chmod +x ~/airflow/scripts/airflow_webserver.sh
    ```

4. Start the Supervisor processes:
    ```bash
    sudo supervisorctl reread
    sudo supervisorctl update
    ```

5. Check that the Airflow webserver is running by visiting `10.0.0.10:8080` in a web browser.


# Install Hadoop
Taken from https://medium.com/@jootorres_11979/how-to-set-up-a-hadoop-3-2-1-multi-node-cluster-on-ubuntu-18-04-2-nodes-567ca44a3b12.

Perform all of these steps on all three nodes:
1. Install Java 8:
    ```bash
    sudo apt-get install -y openjdk-8-jdk
    ```

2. Download Hadoop 3.3.0:
    ```bash
    cd ~
    sudo wget -P ~ https://mirrors.sonic.net/apache/hadoop/common/hadoop-3.3.0/hadoop-3.3.0.tar.gz
    tar xvf hadoop-3.3.0.tar.gz
    mv hadoop-3.3.0 hadoop
    rm -f hadoop-3.3.0.tar.gz
    ```

3. Configure Hadoop to point to Java by adding to `~/hadoop/etc/hadoop/hadoop-env.sh`:
    ```bash
    export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
    ```

4. Move Hadoop so that it lives in its "normal" installation area:
    ```bash
    sudo mv hadoop /usr/local/hadoop
    ```

5. Replace the contents of `/etc/environment` with:
    ```bash
    PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/local/hadoop/bin:/usr/local/hadoop/sbin"

    JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64/jre"
    ```

6. Setup a Hadoop user:
    ```bash
    sudo adduser hadoopuser
    # Enter password "admin" and leave the rest blank
    sudo usermod -aG hadoopuser hadoopuser
    sudo chown hadoopuser:root -R /usr/local/hadoop/
    sudo chmod g+rwx -R /usr/local/hadoop/
    sudo adduser hadoopuser sudo
    ```

7. Add networking into `/etc/hosts`:
    ```bash
    10.0.0.10 airflow
    10.0.0.11 hdfs-1
    10.0.0.12 hdfs-2
    ```

8. Remove the last entry in `/etc/hosts` (it will refer to `127.0.0.1`).


# Setup the Hadoop Master
Perform these steps on `airflow`:
1. Setup an SSH key:
    ```bash
    sudo apt install -y pdsh
    echo "export PDSH_RCMD_TYPE=ssh" >> ~/.bashrc
    ssh-keygen -t rsa -P ""
    # Press enter
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    ```

2. Add the following property to `/usr/local/hadoop/etc/hadoop/core-site.xml`:
    ```xml
    <configuration>
       <property>
          <name>fs.defaultFS</name>
          <value>hdfs://airflow:9000</value>
       </property>
    </configuration>
    ```

3. Add the following properties to `/usr/local/hadoop/etc/hadoop/hdfs-site.xml`:
    ```xml
    <configuration>
      <property>
         <name>dfs.namenode.name.dir</name><value>/usr/local/hadoop/data/nameNode</value>
      </property>
      <property>
         <name>dfs.datanode.data.dir</name><value>/usr/local/hadoop/data/dataNode</value>
      </property>
      <property>
         <name>dfs.replication</name>
         <value>2</value>
      </property>
   </configuration>
    ```

4. Identify worker notes by replacing the contents of `/usr/local/hadoop/etc/hadoop/workers` with:
    ```bash
    hdfs-1
    hdfs-2
    ```

5. Move all of this configuration to the worker nodes:
    ```bash
    scp /usr/local/hadoop/etc/hadoop/* hdfs-1:/usr/local/hadoop/etc/hadoop/
    scp /usr/local/hadoop/etc/hadoop/* hdfs-2:/usr/local/hadoop/etc/hadoop/
    ```

6. Setup SSH for `hadoopuser`:
    ```bash
    su - hadoopuser
    ssh-keygen -t rsa
    # Press enter
    ```

7. Copy the key to all nodes:
    ```bash
    ssh-copy-id hadoopuser@airflow
    ssh-copy-id hadoopuser@hdfs-1
    ssh-copy-id hadoopuser@hdfs-2
    ```

8. Restart **all nodes** with `sudo reboot`.

9. Log back in to `airflow` HDFS and start HDFS:
    ```bash
    su - hadoopuser
    source /etc/environment
    hdfs namenode -format
    start-dfs.sh
    ```
