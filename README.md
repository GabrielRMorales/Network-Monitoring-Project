## For Hiring Managers and Recruiters

This README contains detailed setup steps, configurations, and troubleshooting notes.

If you're looking for a quick, high-level overview of this project (architecture, goals, and outcomes), please see:

**[Project Overview](./PROJECT_OVERVIEW.md)**

Network Monitoring Homelab

Host System 1
Operating System: Windows 10 Pro
CPU: Intel i7-10700KF (8 Cores, 16 Logical Processors)
RAM: 32 GB DDR4
Storage: 1TB NVMe SSD
Hypervisor: Oracle VirtualBox 7.1.4

Host System 2
Operating System: Windows 11 Pro
CPU: AMD Ryzen 7 5800H (8 Cores, 16 Logical Processors)
RAM: 32 GB DDR4
Storage: 1TB NVMe SSD
Hypervisor: Oracle VirtualBox 7.1.6

Ubuntu Server
Type: Ubuntu (64-bit)
RAM: 4096 MB
CPU: 2 cores
Disk: 25 GB (dynamic)

Ubuntu-VM (added after initial setup):
Type: Ubuntu (64-bit)
RAM: 8192 MB
CPU: 4 cores
Disk: 100 GB (dynamic)
Hosted on Host System 2

Topology
[Host Devices / VM1 and 2 / PC]

  [Node Exporter]
        ↓
  [  Prometheus   ] ← metrics (CPU, RAM, etc.)

       ↓
 
  [   Grafana     ]  ← dashboards
 
        + 

  [   Nagios      ]  ← alerts (UP/DOWN, failures)

Key Commands (for manual running)
Run Prometheus:
prometheus --config.file=/etc/prometheus/prometheus.yml
or
sudo systemctl start prometheus

Run Node Exporter:
./node_exporter 
or 
sudo systemctl start node_exporter

Run Grafana:
sudo systemctl start grafana-server

Run Nagios Core:
sudo systemctl start nagios
 

Step 1 — Install Prometheus (metrics engine)

Install Prometheus
# Create user
sudo useradd --no-create-home --shell /bin/false prometheus

# Create directories
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus

# Download
cd /tmp
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-3.11.1.linux-amd64.tar.gz
tar xvf prometheus-3.11.1.tar.gz
cd prometheus-3.11.1

Copy binaries:

sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

Copy config:

sudo cp -r consoles console_libraries /etc/prometheus
sudo cp prometheus.yml /etc/prometheus/
Minimal config (edit this)
sudo nano /etc/prometheus/prometheus.yml

Replace with:

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
Run Prometheus
prometheus --config.file=/etc/prometheus/prometheus.yml

Open in browser:

http://VM_IP:9090
Step 2 — Install Node Exporter (get system metrics)

This gives CPU, RAM, disk, etc.

Install
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.11.1.linux-amd64.tar.gz
tar xvf node_exporter-1.11.1.tar.gz
cd node_exporter-1.11.1
sudo cp node_exporter /usr/local/bin/
Run it:
node_exporter

Test:

http://VM_IP:9100/metrics
Add to Prometheus

Edit config again:

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]

Restart Prometheus.

Step 3 — Install Grafana (visualization)

Install Grafana:

sudo apt-get install -y software-properties-common
sudo apt-get install -y grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

Open:

http://VM_IP:3000 

Login:

user: admin
password: admin (change later)
Connect Prometheus to Grafana
Go to Settings → Data Sources
Add:
Type: Prometheus
URL: http://localhost:9090

Add dashboard
Import dashboard ID: 1860 (Node Exporter dashboard)

Shows:

CPU usage
RAM
Disk
Load
Step 4 — Install Nagios (alerting layer)

Now add Nagios Core

Install dependencies
sudo apt update
sudo apt install -y apache2 php gcc make wget unzip
Download Nagios
cd /tmp
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.5.12.tar.gz
tar xvf nagios-4.5.12.tar.gz
cd nagios-4.5.12
Compile & install
./configure
make all
sudo make install
sudo make install-init
sudo make install-commandmode
sudo make install-config
sudo make install-webconf

Set password:

sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

Start services:

sudo systemctl restart apache2
sudo systemctl start nagios

Open:

http://VM_IP/nagios

Troubleshooting:
-running the command: sudo apt-get install grafana
brought up the output of
Reading package lists...
Done Building dependency tree...
Done Reading state information...
Done E: Unable to locate package grafana

I had already run sudo apt-get update, so this required further research. This was beause Grafana was not included in the default Ubuntu/Debian repositories. Running the follow commands helped to resolve this:
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

I was then able to install Grafana via:
sudo apt-get install grafana

-Ran into an issue with getting data through the Node Exporter dashboard-1860. Node exporter and Prometheus were running, but the CPU basic, memory basic, network traffic basic,
and Disk Space Used Basic graphs showed no data. 
To troubleshoot, I verified a graph existed on Prometheus after running: node_cpu_seconds_total

In Explore, run:

up

Look at the instance value

Example:

instance="127.0.0.1:9100"

Fixes:
On Node Exporter Full dash, setting these allowed for data to show:

Job = node
Instance = localhost:9100

Additionally, what occured is that there was no data for CPU Basic and Network Traffic Basic when the time range was set to <24 hours. When set to 24 hours, it did show data for the different graphs.

-when running ./configure for a few of the programs, this error occured:
checking for SSL headers... configure: error: Cannot find ssl headers
Resolved with sudo apt install -y libssl-dev
Then reran ./configure 
Followed by: make all

-upon visiting http://192.168.1.214/nagios initially and checking Services, it downloaded as a .cgi file rather than showing the services. This was resolved by
enabling the CGI module in apache:
sudo a2enmod cgi
restarting apache:
sudo systemctl restart apache2
Checking the Nagios Apache config via sudo nano /etc/apache2/sites-enabled/nagios.conf revealed that the Options was set to None. Options were set to ExecCGI

-Checking localhost and Services on http://192.168.1.214/nagios showed all as Down/Red/Critical. This required multiple steps of troubleshooting, broken down as follows:
1. Plugins not working initially

Problem:
Nagios couldn’t find or execute the plugins correctly

Root cause:
Mismatch between:

/usr/local/nagios/libexec (expected by source installs)
/usr/lib/nagios/plugins (actual apt-installed plugins)

2. Wrong plugin path in Nagios config

Problem:
Nagios was looking in:

/usr/local/nagios/libexec

Fix:
Updated:

$USER1$=/usr/lib/nagios/plugins

in:

/usr/local/nagios/etc/resource.cfg

3. Check command definition cleaned up

Problem:
Host/service checks needed correct argument format

Fix:
Correct check_ping command:

command_line $USER1$/check_ping -H $HOSTADDRESS$ -w $ARG1$ -c $ARG2$ -p 5

4. Apache command pipe permission issue

Problem:
Couldn’t re-schedule checks:

nagios.cmd permission denied

Fix:

Added www-data to nagios group
Fixed permissions on /usr/local/nagios/var/rw

5. Host/service stuck RED (core issue)

Even though config was correct:

Real root cause:
Nagios was still trying to execute:

/usr/local/nagios/libexec/check_ping  # this didn't exist

The final fix included:
-identifying missing plugin path
Running:
/usr/local/nagios/libexec/check_ping
led to: command not found
-Verified real plugin location
/usr/lib/nagios/plugins/check_ping
-ensured Nagios used correct plugin directory by fixing resource.cfg
sudo nano /usr/local/nagios/etc/resource.cfg
then edited $USER1$=/usr/local/nagios/plugins to be:
$USER1$=/usr/lib/nagios/plugins
-Fixed all config references to use $USER1$ such that there was nothing hardcoded to /usr/local/nagios/libexec
Lastly, Nagios was restarted:
sudo systemctl restart nagios

Enhancements

Add a Second Machine
VM: Linux Ubuntu 22.04.3

On the 2nd VM

1. Set up Node Exporter
Commands:
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.11.1.linux-amd64.tar.gz
tar xvf node_exporter-1.11.1.linux-amd64.tar.gz
cd node_exporter-1.11.1.linux-amd64
./node_exporter

2. On the monitoring VM, verified node exporter worked on second machine, via:
curl http://192.168.1.216:9100/metrics | head

3. Add the second VM for Prometheus to collect metrics from.

On the monitoring VM, ran:
sudo nano /etc/prometheus/prometheus.yml
Added the second VM by editing configuration:

- job_name: "node"
  static_configs:
    - targets: ["localhost:9100", "192.168.1.216:9100"]

Restarted Prometheus with:
sudo systemctl restart prometheus

4. Add the second VM for Nagios to check
-Edit the hosts config file via: 
sudo nano /usr/local/nagios/etc/objects/hosts.cfg
-add:
define host {
    use             linux-server
    host_name       second-machine
    alias           second-machine
    address         192.168.1.50
}

define service {
    use                     generic-service
    host_name               second-machine
    service_description     PING
    check_command           check_ping!100.0,20%!500.0,60%
}

-restart nagios via:
sudo systemctl restart nagios

Set up Services to Run on Start

-for Prometheus, run (see Troubleshooting):
sudo systemctl start prometheus
sudo systemctl enable prometheus

-for Node Exporter, run:
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

-for Grafana, run:
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
 
-for Nagios Core, run:
sudo systemctl start nagios
sudo systemctl enable nagios

Then reboot:
sudo reboot

Enhancements Troubleshooting:

For Prometheus, running sudo systemctl start prometheus led to error:
Failed to start prometheus.service: Unit prometheus.service not found.

The resolution was to run:
sudo nano /etc/systemd/system/prometheus.service

Pasting the following in:

[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

Restart=always

[Install]
WantedBy=multi-user.target

Running these commands to configure permissions:
sudo mkdir -p /var/lib/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus

Reloading the systemd:
sudo systemctl daemon-reload

Then running:
sudo systemctl start prometheus
sudo systemctl enable prometheus

Stress Test example:

While the In the 2nd VM, run:
stress --vm 1 --vm-bytes 2G --timeout 30

The Grafana dashboard should show a spike on the Memory Basic panel.

To stress test the CPU, run:
stress --cpu 2 --timeout 30

The Grafana dashboard should show a spike on the CPU Basic panel.

If the CPU Basic panel is empty, click the three dots>Edit and enter this in the query field:
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

