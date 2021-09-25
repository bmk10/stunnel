# stunnel overview


### Short description

     The stunnel program is designed to work as an SSL encryption
     wrapper between remote client and local (inetd-startable) or
     remote servers. The goal is to facilitate SSL encryption and
     authentication for non-SSL-aware programs.

     stunnel can be used to add  SSL  functionality  to  commonly
     used  inetd  daemons  like  POP-2,  POP-3  and  IMAP servers
     without any changes in the programs' code.

### License

     See <COPYING.md> file.

### Other files you should read

     <NEWS.md> What I did
     <TODO.md> What I'm going to do

### Reporting problems and other contacts

     See <https://www.stunnel.org/faq.html>.
 ### INSTALL
 
 Install using Yum:

yum install stunnel

(Now I cant remember exactly but I think stunnel is not on the default CentOS repos… so you can add the RPMforge repos like I tend to do: http://www.rpmrepo.org/RPMforge/Using)

Install from source:

wget http://mirror.hudecof.net/stunnel/stunnel-4.22.tar.gz
tar zxf stunnel-4.22.tar.gz
cd stunnel-4.22
./configure
make
make install

Configuration:

On CentOS the default location for the stunnel.conf is /etc/stunnel/ so open this file in your editor of choice:

vi /etc/stunnel/stunnel.conf

Lets set the following options:

chroot = /var/run/stunnel
setuid = nobody
setgid = nobody
pid = /stunnel.pid
client = yes

And create a service:

[munin]
accept = 127.0.0.1:4948
connect = myserver.hostname.com:4948

Now as you can see here I’ve set the listener to 127.0.0.1 but you could set this to all interfaces or a specific one by omitting or replacing the 127.0.0.1: The ‘connect’ setting is the servers hostname or IP address and the port that the stunnel ‘server’ is listening on. Another example of using stunnel could be to direct all web requests onto another server using a secure layer:

[www]
accept = 80
connect = myserver.hotname.com:8080

You would then setup the ‘server’ stunnel to listen on 8080 and connect to the local (or even a remote!) web server.

Ok thats it for the client side for now. Lets look at the server:

Install stunnel as per the installation instructions above.

On CentOS the default location for the stunnel.conf is /etc/stunnel/ so open this file in your editor of choice:

vi /etc/stunnel/stunnel.conf

Set the following options:

cert = /etc/stunnel/stunnel.pem
chroot = /var/run/stunnel
setuid = nobody
setgid = nobody
pid = /stunnel.pid

And create a service:

[munin]
accept = 4948
connect = 127.0.0.1:4949

Now you can see here I have set the connect to 127.0.0.1… this is because I can now set the vulnerable service to only bind to a local port rather than a external accessible IP of my server.

Or to continue from our secondary www example:

[www]
accept = 8080
connect = myserver.hostname.com:80

With a bit of messing around with DNS or hosts files we could use stunnel to bounce our connection to any server for example www.bbc.co.uk! But thatâ€™s not really what we’re covering in this article and I’m guessing if you have your reasons for doing something like that you can figure it out on your own.

Ok we’re nearly ready to start stunnel. But the observant amongst you will have noticed that stunnel.pem certificate file that we set in the server options doesnâ€™t exist! So lets create one now!

openssl req -new -x509 -days 3650 -nodes -out /etc/stunnel/stunnel.pem -keyout /etc/stunnel/stunnel.pem

Right now lets start stunnel, on both machines simply run the following command:

/usr/sbin/stunnel /etc/stunnel/stunnel.conf

Now lets test it! If the service we’ve setup can be talked to with telnet (eg. Munin or SMTP) then we can test this very simply from the client machine:

telnet localhost:4948

You should get the following back:

Trying 127.0.0.1…
Connected to localhost (127.0.0.1).
Escape character is ‘^]’.
# munin node at mysever.hostname.com

(Now its a little confusing because ‘Connected to localhost’ is actually the response from the munin-node on the remote server!)

As you can see your telnet session has gone into stunnel locally, been transmitted from stunnel on the local machine to stunnel on the server and then from stunnel into the Munin node on the server! Magic!

Stunnel startup script:

Now you can either start stunnel every time your machine starts up manually, add it to the crontab (if you try and start stunnel again and its already running the second instance will just close, but it leaves a mess in your /var/log/secure so donâ€™t do it to often) or use a simple startup script like this one I ( used ) to use:

#!/bin/bash
    if [ -f /var/run/stunnel/stunnel.pid ]; then
        ps aux |grep -v grep |grep $pid |grep stunnel > /dev/null
        if [ $? = 0 ]; then
            echo “Server is already running !!”
        else
            echo “Pid file exists but process not found … trying to start stunnel”
            /usr/sbin/stunnel /etc/stunnel/stunnel.conf
        fi
        rm -f /tmp/stunnelrun > /dev/null
    else
        echo “Pid file not found. Starting stunnel.”
        /usr/sbin/stunnel /etc/stunnel/stunnel.conf
    fi

This is a very simple script I knocked up in a few mins. I later replaced it with an init script that I wrote when I had a little more time. But I’m going to post that as another article as I have a bit more to say about that (including some adaptationto make general init scripts for various programs)

To use the above script to start stunnel do the following:

cd /usr/local/sbin
vi stunnel-run

Paste in the above code.
Save and exit.

chmod +x stunnel-run

Test by doing the following:

./stunnel-run

Now add it to the crontab:

crontab -e

Insert the following line:

*/15 * * * * /usr/local/sbin/stunnel-run 2>&1 > /dev/null

Now your system will run the stunnel-run script every 15mins, checking if stunnel is running and starting stunnel if it is not running. You could simply start stunnel every 15mins as it will exit if it finds it can not use the ports its been assigned, but that leaves a mess in your secure log.
