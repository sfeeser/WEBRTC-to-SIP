# WEBRTC to SIP client and server
How to setup Kamailio + RTPEngine + TURN server to enable calling between WEBRTC client and legacy SIP clients. This setup will bridge SRTP --> RTP and ICE --> nonICE to make a WEBRTC client (SIPJs) be able to call legacy SIP clients.

This setup is for Debian 9 Stretch for all servers.

This setup is configured to run with the following servers:

1. Server - Kamailio + RTPEngine + Nginx (WEBRTC client)
2. Server - TURN

The configuration is setup to always bridge via RTPEngine. To change the behavior, take a look in the `NATMANAGE` route.

## Architecture
![WebRTC - SIP architecture](https://raw.githubusercontent.com/havfo/WEBRTC-to-SIP/master/images/webrtc-sip.png "WebRTC to SIP architecture")

## Get certificates
For the certificates you need a simple solution is Let's Encrypt certificates. They will work for both Kamailio TLS and Nginx TLS. On the servers you need certificates, run the following (you must stop services running on port 443 during certificate request/renewal):
```bash
apt-get install certbot
certbot certonly --standalone -d YOUR-DOMAIN
```
You will then find the certificates under:
```bash
/etc/letsencrypt/live/YOUR-DOMAIN/privkey.pem
/etc/letsencrypt/live/YOUR-DOMAIN/fullchain.pem
```

## Get configuration files
All files needed to setup all components on Debian 9 Stretch.
```bash
git clone https://github.com/havfo/WEBRTC-to-SIP.git
cd WEBRTC-to-SIP
find . -type f -print0 | xargs -0 sed -i 's/XXXXX-XXXXX/PUT-IP-OF-YOUR-SIP-SERVER-HERE/g'
find . -type f -print0 | xargs -0 sed -i 's/XXXX-XXXX/PUT-DOMAIN-OF-YOUR-SIP-SERVER-HERE/g'
find . -type f -print0 | xargs -0 sed -i 's/XXX-XXX/PUT-DOMAIN-OF-YOUR-TURN-SERVER-HERE/g'
```

## Install RTPEngine
This will do the SRTP-RTP bridging needed to make WEBRTC clients talk to legacy SIP server/clients. You can find the latest build instructions in their [readme](https://github.com/sipwise/rtpengine#on-a-debian-system).

After you have successfully installed RTPEngine, copy the configuration from this repository.
```bash
cd WEBRTC-to-SIP
cp etc/default/ngcp-rtpengine-daemon /etc/default/
cp etc/rtpengine/rtpengine.conf /etc/rtpengine/
/etc/init.d/ngcp-rtpengine-daemon restart
```

## Install IPTables firewall (optional)
RTPEngine handles the chain for itself, but make sure to not block the RTP-ports it is using. Take a look in iptables.sh for details, and apply it by doing the following. This will persist after reboot. You can run the iptables.sh script at any time after it is set up.
```bash
cd WEBRTC-to-SIP
chmod +x iptables.sh
cp etc/network/if-up.d/iptables /etc/network/if-up.d/
chmod +x /etc/network/if-up.d/iptables
touch /etc/iptables/firewall.conf
touch /etc/iptables/firewall6.conf
./iptables.sh
```

## Install Kamailio
```bash
apt-get install kamailio kamailio-websocket-modules kamailio-mysql-modules kamailio-tls-modules kamailio-presence-modules mysql-server
cd WEBRTC-to-SIP
cp etc/kamailio/* /etc/kamailio/
kamdbctl create
```
Select yes (Y) to all options.

```bash
kamctl add websip websip
/etc/init.d/kamailio restart
```

## Install WEBRTC client
```sh
apt-get install nginx
cd WEBRTC-to-SIP
cp etc/nginx/sites-available/default /etc/nginx/sites-available/
cp -r client/* /var/www/html/
```

## Install TURN server
```sh
apt-get install coturn
cp etc/default/coturn /etc/default/
cp etc/turn* /etc/
/etc/init.d/coturn restart
```

## Testing
You should now be able to go to https://webrtcnginxserver/ and call legacy SIP clients.
