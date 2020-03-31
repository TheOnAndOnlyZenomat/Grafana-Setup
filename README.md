# Grafana-Setup# Systemmonitoring with Grafana, InfluxDB and Telegraf

![dashboard.png](/dashboard.png)
This is the endresult. We can see the total cpu usage, the usage per core, ram, processes (runnin and idle), total diskspace used, network usage and uptime. I also monitor our diskIO, but this isn't on the screenshot

## Prerequisits
I use a Raspberry Pi with Debian as my server, so my Instructions are for Debian and ARM packages, but should translate to other cpu-architectures. You may just have to add another repository, but this should be doable with a little bit of google. Just look at the documentation for the services. It is covered there

## 1. Requirenments
- InfluxDB
- Telegraf
- Grafana

## 2. Setup
### 2.1 InfluxDB
1. Install the InfluxDB package for your system
1.1 Add the repository
    `wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -`
    `source /etc/os-release`
    `echo "deb https://repos.influxdata.com/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/influxdb.list`
1.2 Install the Package and start the service
     `sudo apt-get update && sudo apt-get install influxdb`
     `sudo service influxdb start`
     Use systemd instead of `service` to restart the service after reboot (not sure if you use the `service` comand if it restarts after reboot)
     `sudo systemctl unmask influxdb.service`
     `sudo systemctl start influxdb`
2. Configure InfluxDB
2.1 The config file is located at `/etc/influxdb/influxdb.conf`
2.2 Open the file and search for the `[http]`part of the file
2.3 Uncomment the `enable = true` and the `bind-address = :8086` in the config file.
By uncommenting `enable = true` this you enable the HTTP-Endpoint. This is where Telegraf will     connect to, to input the measurment data in the database.
By uncommenting the `bind-address = :8086` you tell Influx on which port it should run. You can     change this to whatever port you want, you just have to change this at some later points. 
3. Configure the Database
3.1 Change into the database shell with by typing `influx`
3.2 Create a database for telegraf called `telegraf`
You do this by typing `CREATE DATABASE telegraf`
If you want advanced security, this maybe is a good idea if you want to make you dashboard and server accessible from the internet, you can create a user with the minimal required privileges and grant him all privileges on the database, this I wont cover here, but shouldn't be to hard to implement if you know how to google.
</br>
This should suffice for the database setup, now we will take care of the collection of data with Telegraf

### 2.2 Telegraf
1. Install Telegraf for your system
1.1 Add the repository
`sudo apt-get update && sudo apt-get install apt-transport-https` The Telegraf documentation recommends this, so that apt will be able to read the repository
`wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -`
`source /etc/os-release`
`test $VERSION_ID = "7" && echo "deb https://repos.influxdata.com/debian wheezy stable" | sudo tee /etc/apt/sources.list.d/influxdb.list`
`test $VERSION_ID = "8" && echo "deb https://repos.influxdata.com/debian jessie stable" | sudo tee /etc/apt/sources.list.d/influxdb.list`
`test $VERSION_ID = "9" && echo "deb https://repos.influxdata.com/debian stretch stable" | sudo tee /etc/apt/sources.list.d/influxdb.list`
`test $VERSION_ID = "10" && echo "deb https://repos.influxdata.com/debian buster stable" | sudo tee /etc/apt/sources.list.d/influxdb.list`
1.2 Install the Package and start the service
`sudo apt-get update && sudo apt-get install telegraf`
`sudo service telegraf start`
Use systemd instead of `service` to restart the service after reboot (not sure if you use the `service` comand if it restarts after reboot)
`sudo systemctl start telegraf`
2. Configure Telegraf
The config file we will use is located at `/etc/telegraf/telegraf.conf`, but there is the posibility to use a configfile ate another location. You just have to start telegraf with the argument `-config /path/to/file`
In the config file basically everything is commented out, because then the defaults are used.
First we have to configure our output, so that Telegraf outputs its data to our InfluxDB instance. For this we have to search the `[[outputs.influxdb]]` section in the config file. Here we uncomment the 7th line from `[[outputs.influxdb]]` `urls = ["http://127.0.0.1:8086"`. If you have defined a different port for the InfluxDB Instance, you have to change this in the URL.
Another three lines down from there, we have to uncomment the `database = ""` part and fill in our database name between quotationmarks. We named our database `Telegraf` so we will add this between the quotation marks.
Next we can choose our data collection rate. We can do this under the `[agent]` section in the file. There you can change the `interval` to whatever interval you want. I choose 5s, but this is up to you.
After we have taken care of our output and our measurment interval, we can face the stuff we want to collect. If you scroll through the file you can see what is possible. Every plugin is enclosed in `[[plugin]]`.
</br>
For the beginning, say we want to see our cpu utilization. For this we have to uncomment the line `[[inputs.cpu]]`.
Under the point `[[inputs.cpu]]`we also can see some configuration options, but we can leave them at their default options.
</br> I also uncommented the the `[[inputs.disk, [[inputs.diskio]], [[inputs.kernel]], [[inputs.mem]], [[inputs.processes]], [[inputs.swap]], [[inputs.apache]], [[inputs.net]]`
The configuration of those modules is not explicitly necessary, so I wont cover this. The only one I will mention very brief is the `[[inputs.net]]` module.
Here you have to change the `interfaces = ["put you network interface here"]`, I use wlan, so I use `["wlan0"]`. But you have to change this to whatever you use. You can see this by typing `ifconfig` in your terminal and look for the one, where you have proper ip address. In most cases it is something with `198.178.x.x`.
</br>
Now we have taken care of our datacollection, we just have to import it to Grafana

### 2.3 Grafana
1. Install Grafana on you system
1.1 Add the repository
The following is for ARM, but here https://grafana.com/docs/grafana/latest/installation/debian/ you can see how to do it with other architectures.
`echo "deb https://packages.grafana.com/oss/deb beta main" | sudo tee -a /etc/apt/sources.list.d/grafana.list`
`sudo apt-get update`
`sudo apt-get install grafana`
2. Start the server with systemd
`sudo systemctl daemon-reload`
`sudo systemctl start grafana-server`
This should do it. With the command `sudo systemctl status grafana-server` you can see the status of the process.
If you want it to start at system reboot, execute `sudo systemctl enable grafana-server.service`
3. Configure Grafana
The default port is `3000` but you can change this at the config file, but I wont cover this because this is not explicitly necessary and you can configure everything we need in Grafana.
Head over to `http://localhost:3000` and log in with `admin` `admin`. From there you can change the password in the settings, but I bet you can find this on your own.
4. Create the dashboard
Go to the sidebar and chose Data Sources under the Configuration tab. Click on `Add data source` and choose InfluxDB.
Give it a name and enter the URL which is, in our case, `http://localhost:8086` (if you have choosen a different port for the InfluxDB instance, put that there instead of `:8086`). We can leave the rest, if you did choose to create a user for the database fill those credentials in at the point `InfluxDB Details`.
Then go to the bottom of the page and click on `Save & Test`. It should give you a green light.

From there head over to sidepannel and choose Create Dashboard. New dashboard automatically start with a new Panel. There click on `Add Query`, choose you `InfluxDB` in the Dropdown next to `Query`.
Say we want to proceed with our CPU monitoring:
Do the configuration just like this. Of yourse you can play around with the values a bit.
![cpudash.png](/cpudash.png)
For the network I use our configured `net` plugin from telegraf. Here is my queryconfiguration:![networkquery.png](/networkquery.png)
In the Visualization tab you have to choose bytes/sec on the y-axis, otherwise the proportions won't fit.
One problem I had when doing this dashboard was figuring out how to display my diskio. Here is my solution:
![diskioquery.png](/diskioquery.png)
Just do the same for writes, just change (bytes_read) to (bytes_write). Again, in the Visualization tab choose bytes/sec for the y-axis.
Now one last thing, my disk space gauge. For the query just choose disk and and the right disk name in the `from` section and choose `field(used)`. Keep in mind this is also measured in bytes, so head over to the Visualization tab and configure it like this:
![diskdash.png](/diskdash.png)
</br>
The rest of my graphs are pretty easy to configure, so I wont cover this.
</br>
And there we are, a finished dashboard.
