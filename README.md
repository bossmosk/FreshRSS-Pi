
# FreshRSS

The setup

    Raspberry Pi running Raspian Lite (host & os)
    FreshRSS (aggregator)
    Lighttpd (web server)
    Kill the newsletter (service to transform email newsletters into RSS – awesome stuff)
    Reeder 5 (local client reader for iOS and MacOS), browser view works just as fine.

Assumptions

    You have SSH and sudo to the Pi
    You have git and know basic comands
    You have Lighttpd, Sqllite3, php running from a pi-hole install
    You know few basics around the CLI

Downloading

Let’s get started with updating the system first (you might want to do a reboot if it has been a while since you updated, just saying).

`sudo apt-get update && sudo apt-get dist-upgrade`

Get all the software we need from the repo (you should have most of the php packages from the pi-hole install anyways)

`sudo apt install php php-curl php-gmp php-intl php-mbstring php-sqlite3 php-xml php-zip`

Side note, if you want to check what is already installed for php before running the line above, run this

` dpkg --get-selections | grep -e php `

Ok, prep done mostly, once you launch FreshRSS for the first time it will check for dependencies, you might need to install something new.. it is in active development thankfully!

Time for FreshRSS, I’m going to download the git master branch in /srv/

` cd /srv/ && sudo git clone https://github.com/FreshRSS/FreshRSS.git `

Configure Lighttpd web-server

We need to tell lighttpd to publish our FreshRSS install as a website, but the lighttpd config file (/etc/lighttpd/lighttpd.conf) is automatically overwritten at install/update by pi-hole.. so any change we make there will be lost on the next update.

Thankfully we can write our config in another file that lighttpd will pick up, the external.conf file (/etc/lighttpd/external.conf): this file is untouched by pi-hole during install/update.

There is also something else to consider: if you have pi-hole install then you know visiting your pi local ip will show you the admin page for pi-hole, this means port 80 of lighttpd is publishing pi-hole, so to keep it simple we will use another port for FreshRSS, any will do, just stay out of the common ports (=<1023) or double check you are not taking up something on this list and you will be fine. I’m using port 2000 for this example.

To edit external.conf

`sudo nano /etc/lighttpd/external.conf`

Once inside, type this and then save.

# FreshRSS config

$SERVER["socket"] == ":2000" {
    server.document-root     = "/var/www/html/freshrss/p"
}

We just told lighttpd to take whatever is in the folder /p and publish it on port 2000, little problem: folder /p is not there yet, we need to link it. Also, until we restart the service/pi, lighttpd doesn’t know we changed the config, keep that in mind.

If you read the FreshRSS documentation, /p is the only folder you want to expose on the web server as other folders have personal data – this install is local only so not much of a problem if you mess up, but is always best to follow best practices.

Configure FreshRSS

Linking /p to the web server folder is easy, if you have followed along with the folder I used, you can copy this

`sudo ln -s /srv/FreshRSS/ /var/www/html/freshrss`

Linux is case sensitive, so make sure you get the right capitalisation – using the Tab key while in the terminal to autocomplete is the best approach.

The web server user account (www-data, in the www-data group) needs access to the FreshRSS folder in order to work with it: read permission for the whole folder and write for the /data folder.

If you want to check what user is assigned to the web server, check the /etc/lighttpd/lithtpd.conf file, specifically server.username and server.groupname: if you have the pi-hole conf it will be www-data for both.

`cd /srv/FreshRSS`
`sudo chown -R www-data:www-data .`
`sudo chmod -R g+r .`
`sudo chmod -R g+w ./data`

Configure cron job to update the feeds

All well and good we are getting FreshRSS installed, but if it is not pulling RSS updates automatically, then what is the point?

To do that we need to a create a cron job for sudo, first open the crontab for sudo

`sudo crontab -e`

Then copy in the below code to: run the php actualize_script every 15 minutes for the www-data user and write a log of the script running (or failing) at /tmp/FreshRSS.log – this is all on one line.

You can change the sintax if you want, plenty of resources online to learn it.

`*/15 * * * * 
sudo -u www-data php -f /srv/FreshRSS/app/actualize_script.php > /tmp/FreshRSS.log 2>&1`

First run

Ok, time to restart lighttpd and check we are running

`sudo /etc/init.d/lighttpd restart`

Now open a browser with access to the raspberry pi lan, and if all is well, typing http://raspberrypi_ipaddress:2000 will show FreshRSS

Next steps

From there you can configure your user and feeds.

I will run this locally for a while and see if I want to publish it online to gain access while away from home, another option is to open a VPN so that you can access this resource while away.. I’m still undecided.

To configure a client reader, like Reeder 5, you will need to enale the API access and probably setup a password too, I found some issues with the latest version fo FreshRSS and Reeder 5, probably because I’m not using a secure connection between the two, so I downgraded FreshRSS to a previous version and use the old auth method which works just fine.
