# ELK Stack Installation Steps
The first step is to install Elasticsearch & Kibana. To get started, add the Elastic GPG key to your server with the following command: 

    curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

Next, add the Elastic source list to the `sources.list.d` directory, where `apt` will search for new sources:

    echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

Now update your server’s package index and install Elasticsearch and Kibana:

    sudo apt update && sudo apt install net-tools
    sudo apt upgrade
    sudo apt install elasticsearch kibana
------------
Elasticsearch is configured to only accept local connections by default. Additionally, it does not have any authentication enabled, so tools like Filebeat will not be able to send logs to it. In this section we will configure the network settings for Elasticsearch and then enable Elasticsearch’s built-in `xpack` security module.
Firstly, we need to check our IP-Address which is assigned to our machine. Use the following command to check the IP Address. 

    ifconfig

Now open the `/etc/elasticsearch/elasticsearch.yml` file using `nano` or your preferred editor:

    sudo nano /etc/elasticsearch/elasticsearch.yml
Find the commented out `#network.host: 192.168.0.1` line between lines 50–60 and add a new line after it that configures the `network.bind_host` setting, as highlighted below:

    network.bind_host: ["127.0.0.1", "your_private_ip"]
    
![Replace "your_private_ip" with your Machine IP!](https://i.imgur.com/X8BdWkK.png)
Basically, we have to replace `"your_private_ip"` with the machine's IP address (from this IP the elasticsearch is accessible). This line will ensure that Elasticsearch is still available on its local address so that Kibana can reach it, as well as on the private IP address for your server.

Next, go to the end of the file using arrow keys. Add the following highlighted lines to the end of the file:
```
discovery.type: single-node
xpack.security.enabled: true
```
The  `discovery.type`  setting allows Elasticsearch to run as a single node, as opposed to in a cluster of other Elasticsearch servers. The  `xpack.security.enabled`  setting turns on some of the security features that are included with Elasticsearch.

Save and close the file when you are done editing it. If you are using  `nano`, you can do so with  `CTRL+X`, then  `Y`  and  `ENTER`  to confirm.

Finally, add firewall rules to ensure your Elasticsearch server is reachable on its private network interface. If you followed the prerequisite tutorials and are using the Uncomplicated Firewall (`ufw`), run the following commands:

    sudo ufw allow in on eth0
    sudo ufw allow out on eth0

Now that you have configured networking and the  `xpack`  security settings for Elasticsearch, you need to start it for the changes to take effect.

Run the following  `systemctl`  command to start Elasticsearch:

    sudo systemctl start elasticsearch.service
Once Elasticsearch finishes starting, you can continue to the next section of this tutorial where you will generate passwords for the default users that are built-in to Elasticsearch.

### Configuring Elasticsearch Passwords

Now that you have enabled the  `xpack.security.enabled`  setting, you need to generate passwords for the default Elasticsearch users. Elasticsearch includes a utility in the  `/usr/share/elasticsearch/bin`  directory that can automatically generate random passwords for these users.

Run the following command to  `cd`  to the directory and then generate random passwords for all the default users:

    cd /usr/share/elasticsearch/bin
    sudo ./elasticsearch-setup-passwords auto
You will receive output like the following. When prompted to continue, press `y` and then `RETURN` or `ENTER`:
Below is a sample output! The password will vary from system to system! Make sure you save all of these passwords somewhere as we may need it later! 
```
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
The passwords will be randomly generated and printed to the console.
Please confirm that you would like to continue [y/N]y


Changed password for user apm_system
PASSWORD apm_system = eWqzd0asAmxZ0gcJpOvn

Changed password for user kibana_system
PASSWORD kibana_system = 1HLVxfqZMd7aFQS6Uabl

Changed password for user kibana
PASSWORD kibana = 1HLVxfqZMd7aFQS6Uabl

Changed password for user logstash_system
PASSWORD logstash_system = wUjY59H91WGvGaN8uFLc

Changed password for user beats_system
PASSWORD beats_system = 2p81hIdAzWKknhzA992m

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = 85HF85Fl6cPslJlA8wPG

Changed password for user elastic
PASSWORD elastic = 6kNbsxQGYZ2EQJiqJpgl
```
You will not be able to run the utility again, so make sure to record these passwords somewhere secure.
## Step 3 — Configuring Kibana

First you’ll enable Kibana’s  `xpack`  security functionality by generating some secrets that Kibana will use to store data in Elasticsearch. Then you’ll configure Kibana’s network setting and authentication details to connect to Elasticsearch.
### Enabling  `xpack.security`  in Kibana

To get started with  `xpack`  security settings in Kibana, you need to generate some encryption keys. Kibana uses these keys to store session data (like cookies), as well as various saved dashboards and views of data in Elasticsearch.

You can generate the required encryption keys using the  `kibana-encryption-keys`  utility that is included in the  `/usr/share/kibana/bin`  directory. Run the following to  `cd`  to the directory and then generate the keys:

    cd /usr/share/kibana/bin/
    sudo ./kibana-encryption-keys generate -q
   Given below is a sample output example: 

```
SAMPLE OUTPUT!
xpack.encryptedSavedObjects.encryptionKey: 66fbd85ceb3cba51c0e939fb2526f585
xpack.reporting.encryptionKey: 9358f4bc7189ae0ade1b8deeec7f38ef
xpack.security.encryptionKey: 8f847a594e4a813c4187fa93c884e92b
```
Copy your output somewhere secure. You will now add them to Kibana’s  `/etc/kibana/kibana.yml`  configuration file.

Open the file using  `nano`  or your preferred editor:

    sudo nano /etc/kibana/kibana.yml
Go to the end of the file using the `nano` shortcut `CTRL+v` until you reach the end. Paste the three `xpack` lines that you copied to the end of the file. 
To configure Kibana’s networking so that it is available on your Elasticsearch server’s private IP address, find the commented out `#server.host: "localhost"` line in `/etc/kibana/kibana.yml`. The line is near the beginning of the file. Add a new line after it with your server’s private IP address, as highlighted in the sample image below: 
![enter image description here](https://i.imgur.com/fZCgdTc.png)
Substitute your private IP in place of the  `your_private_ip`  address.

Save and close the file when you are done editing it. If you are using  `nano`, you can do so with  `CTRL+X`, then  `Y`  and  `ENTER`  to confirm.

Next, you’ll need to configure the username and password that Kibana uses to connect to Elasticsearch.

### Configuring Kibana Credentials
To add a secret to the keystore using the `kibana-keystore` utility, first `cd` to the `/usr/share/kibana/bin` directory. Next, run the following command to set the username for Kibana:

    sudo ./kibana-keystore add elasticsearch.username

You will receive a prompt like the following:
```
Enter value for elasticsearch.username: **************
```
Enter  `kibana_system`  when prompted, either by copying and pasting, or typing the username carefully. Each character that you type will be masked with an  `*`  asterisk character. Press  `ENTER`  or  `RETURN`  when you are done entering the username.

Now repeat the same command for the password. Be sure to copy the password for the  `kibana_system`  user that you generated in the previous section. For reference, in previous step the example password is  `1HLVxfqZMd7aFQS6Uabl`.

Run the following command to set the password:

    sudo ./kibana-keystore add elasticsearch.password
When prompted, paste the password to avoid any transcription errors:
```
Enter value for elasticsearch.password: ********************
```
### Starting Kibana

Now that you have configured networking and the  `xpack`  security settings for Kibana, as well as added credentials to the keystore, you need to start it for the changes to take effect.

Run the following  `systemctl`  command to restart Kibana:

```
sudo systemctl start kibana.service
```
Once all the above steps are executed properly, you can nagivate to Kibana console. 
Kibana runs on port 5601 by default. 
To access kibana navigate: 

> http://< your-ip >: 5601

Once the login screen is prompted, enter the username as 'elastic' and password will be the default password of the 'elastic_user', which was generated in the previous steps. 
