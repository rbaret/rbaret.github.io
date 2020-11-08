---
layout: post
title: "Configure Grafana with InfluxDB and MQTT"
permalink: /configure-grafana-influxdb-mqtt-debian/
categories: Debian Grafana InfluxDB MQTT Domotic
---

Following [DiyIOt](https://diyi0t.com/visualize-mqtt-data-with-influxdb-and-grafana/) instructions.

- [Install InfluxDB](#install-influxdb)
- [Configure MQTT bridge between InfluxDB and MQTT broker](#configure-mqtt-bridge-between-influxdb-and-mqtt-broker)
- [Install Grafana](#install-grafana)

## Install InfluxDB

```bash
sudo apt install influxdb
sudo apt install influxdb-client

# Start InfluxDb service
sudo service influxdb start

# Show InfluxDB status
sudo service influxdb status

# Edit configuration
sudo nano /etc/influxdb/influxdb.conf
```

Enable http endpoint by uncommenting the *enabled = true* line in the \[http\] section :

```bash
[http]
  # Determines whether HTTP endpoint is enabled.
   enabled = true
```

Restart the service :

```bash
# Retart InfluxDb service
sudo service influxdb start
```

Now the configuration is finished and we create a database where all MQTT information are stored and a user that write the MQTT data to the database.
First start InfluxDB with the following command :

```bash
# Start InfluxDb
influx
```

Create a new database :

```sql
-- Connected to http://localhost:8086 version 1.6.4
-- InfluxDB shell version: 1.6.4
CREATE DATABASE smarty
CREATE USER mqtt WITH PASSWORD 'xxxxxx'
GRANT ALL ON smarty TO mqtt
-- Exit Influx
exit
```

## Configure MQTT bridge between InfluxDB and MQTT broker

The bridge is a python script that requires python to be installed :

```bash
# Install Python 3 and PIP if not already present
sudo apt install python3 python3-pip

# Install prerequisites
sudo pip3 install paho-mqtt # Install Python3 MQTT language bindings
sudo pip3 install influxdb # Install Python3 InfluxDB package


sudo nano MQTTInfluxDBBridge.py # Create the python file for the MQTT subscriber
```

Content of *MQTTInfluxDBBridge.py*

```python
import re
from typing import NamedTuple

import paho.mqtt.client as mqtt
from influxdb import InfluxDBClient

INFLUXDB_ADDRESS = 'localhost' # Replace with IP adress if on remote host
INFLUXDB_USER = 'mqtt'
INFLUXDB_PASSWORD = 'xxxxxxxx'
INFLUXDB_DATABASE = 'smarty'

MQTT_ADDRESS = 'localhost' # Replace with IP adress if on remote host
MQTT_USER = 'smarty'
MQTT_PASSWORD = 'xxxxxxxxxx'
MQTT_TOPIC = [("lamsmarty/pwr_delivered/value",0),("lamsmarty/gas_index/value",0),("lamsmarty/energy_delivered_tariff1/value",0)]
MQTT_REGEX = 'lamsmarty/([^/]+)/([^/]+)'
MQTT_CLIENT_ID = 'MQTTInfluxDBBridge'

influxdb_client = InfluxDBClient(INFLUXDB_ADDRESS, 8086, INFLUXDB_USER, INFLUXDB_PASSWORD, None)

class SensorData(NamedTuple):
    location: str
    measurement: str
    value: str


def on_connect(client, userdata, flags, rc):
    """ The callback for when the client receives a CONNACK response from the server."""
    print('Connected with result code ' + str(rc))
    client.subscribe(MQTT_TOPIC)


def on_message(client, userdata, msg):
    """The callback for when a PUBLISH message is received from the server."""
    print(msg.topic + ' ' + str(msg.payload))
    sensor_data = _parse_mqtt_message(msg.topic, msg.payload.decode('utf-8'))
    if sensor_data is not None:
        _send_sensor_data_to_influxdb(sensor_data)

def _parse_mqtt_message(topic, payload):
    match = re.match(MQTT_REGEX, topic)
    if match:
        location = match.group(1)
        measurement = match.group(2)
        if measurement == 'status':
            return None
        payloadstr = str(payload)
        payloadstr.lstrip('0')
        return SensorData(location, measurement, float(payloadstr))
    else:
        return None


def _send_sensor_data_to_influxdb(sensor_data):
    json_body = [
        {
            'measurement': sensor_data.measurement,
            'tags': {
                'location': sensor_data.location
            },
            'fields': {
                'value': sensor_data.value
            }
        }
    ]
    influxdb_client.write_points(json_body)


def _init_influxdb_database():
    databases = influxdb_client.get_list_database()
    if len(list(filter(lambda x: x['name'] == INFLUXDB_DATABASE, databases))) == 0:
        influxdb_client.create_database(INFLUXDB_DATABASE)
    influxdb_client.switch_database(INFLUXDB_DATABASE)



def main():
    _init_influxdb_database()

    mqtt_client = mqtt.Client(MQTT_CLIENT_ID)
    mqtt_client.username_pw_set(MQTT_USER, MQTT_PASSWORD)
    mqtt_client.on_connect = on_connect
    mqtt_client.on_message = on_message

    mqtt_client.connect(MQTT_ADDRESS, 1883)
    mqtt_client.loop_forever()


if __name__ == '__main__':
    print('MQTT to InfluxDB bridge')
    main()

```

Try the python script to see if subscription is ok

```shell
python3 MQTTInfluxDBBridge.py
```

Check if data is inserted into the smarty database

```shell
influx
```

```sql
USE smarty
SHOW MEASUREMENTS
SELECT * from value
exit
```

Run MQTT bridge at startup

```shell
# Create launch script
nano launcher.sh
```

Edit the content :

```bash
#!/bin/sh
# launcher.sh

sleep 10

cd /
cd home/richard
python3 MQTTInfluxDBBridge.py
cd /
```

Quit editor (CTRL+X, Y)
Launch crontab

```shell
crontab -e
```

Add the following line :

```shell
@reboot sh /home/richard/launcher.sh >/home/richard/logs/cronlog 2>&1
```

and quit the editor.

Reboot or launch the script manually.

Check the logs to check everything is working

```shell
cat logs/cronlog
```

## Install Grafana

Check Grafana's latest version on [Grafana repo](https://github.com/grafana/grafana/releases). At the time of this documentation the latest versios is 7.3.1

Install Grafana :

```shell
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_7.3.1_amd64.deb
sudo dpkg -i grafana_7.3.1_amd64.deb
sudo apt update
sudo apt install grafana

# Start Grafana service
sudo /bin/systemctl start grafana-server

# Enable Grafana to start automatically
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server
```

Access Grafana in the browser using host's <http://x.x.x.x:3000>
Login using default credentials admin/admin and change default ad,in password

Add a new data source using the dedicated card and choose InfluxDB as type

Under HTTP, set <http://localhost:8086> in URL (or your InfluxDB host IP)

Under **InfluxDB Details** type **mqtt** user credentials we created in InfluxDB and **smarty** as database name, and set HTTP method to **GET**

Click *Save & Test* and check that you receive a message of successful connection.
