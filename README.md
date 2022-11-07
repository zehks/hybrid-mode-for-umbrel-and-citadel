# Disclaimer
I am not taking any responsibility if something goes wrong. Don't issue **any** command if you are not sure about it. **Actually, nobody should issue any command if they do not know what it does.** I am not doing any troubleshooting.

## Uncovered
This guide does not cover all security considerations such as restricting SSH to private key login only or denying logins from other IP than yours.
It does not cover neither to make the VPS configuration persistent so it keeps configuration when rebooted.

# Prerequisites

- To be able to SSH in your Lightning node
- To be able to SSH in your VPS
- To have a static IP in your VPS
- To have access to VPS security configuration

# Assumptions

- You having a current working RunCitadel or GetUmbrel instance
- You are running Lightning Network Daemon on port 9735

# Setting up an encrypted and secure tunnel

For that purpose we are going to make use of native Tailscale app from the appstore.

1. Go ahead to Tailscale\'s website and sign up ([[www.tailscale.com]](http://www.tailscale.com)).
2. Then install Tailscale app in your node and launch it. Web browser will pop up a window to log in.

Now you should see a new machine in your Tailscale\'s account admin panel (website). This machine should be your node.

3. SSH in your VPS and install Tailscale:

> curl -fsSL https://tailscale.com/install.sh \| sh

Watch your VPS terminal\'s output. Look for any error there. If not, it should say you must log in through a link. Copy that link and paste it into your web browser. Log in. You should now see two machines in your Tailscale admin panel. Big yikes!

# Testing your VPN tunnel

On your VPS terminal you should be able to issue the command \"tailscale status\", which should output two machines. One for your VPS and another for your node. Look at the node\'s IP in that terminal output. Issue the command "tailscale ping \<ip_address\>\".

Remember to replace \"\<ip_address\>\" with the node\'s actual VPN IP address. If the terminal replies with a \"pong\", you are done on the VPN Tunnel setting.

# Forwarding traffic

Now you have set up your vpn tunnel and your two machines are able to see each other and talk. Fine! We need now to tell the VPS to forward traffic to the node. For that purpose we are going to use \"iptables\".

**On the VPS:**

We are going to need some information here.

1. Interfaces names. Typically \"eth0\" will be your Internet\'s interface and \"tailscale0\" your VPN interface. You can issue the command "ip address show".
2. VPS\' VPN IP address.
3. Node\'s VPN IP address. Both IP can be found on \"tailscale status\".

Following the example, \"eth0\" will be replaced with \"vps_internet_iface\" and \"tailscale0\" with \"vps_vpn_iface\".

VPS\' VPN IP will be \"vps_vpn_ip\" and your node\'s VPN IP \"node_vpn_ip\". So when issuing the following commands, you need to replace the values.

Issue the following commands on your VPS:

> sudo iptables -A PREROUTING -t nat -i \<vps_internet_iface\> -p tcp -m tcp \--dport 9735 -j DNAT \--to \<node_vpn_ip\>

And again the same command replacing both \"tcp\" with \"udp\":

> sudo iptables -A PREROUTING -t nat -i \<vps_internet_iface\> -p udp -m udp \--dport 9735 -j DNAT \--to \<node_vpn_ip\>

The last iptables entry is the following one:

> sudo iptables -t nat -A POSTROUTING -d \<node_vpn_ip\> -o \<vps_vpn_iface\> -j MASQUERADE

Then issue the command:

> sudo sysctl -w net.ipv4.ip_forward=1

That last command says to Linux to act as a router (i.e. to forward packets).

# Testing before moving on LND configuration

Head to some online service for checking open ports like [https://portchecker.co/check](https://portchecker.co/check) and check connectivity. You must put in there your VPS public IP and port number 9735. If it is opened, you are ready to go to the next step.

If it is closed, you need to troubleshoot. Some points to check:

- Is your VPS provider allowing traffic in from port 9735?
- Is your VPS firewall (such as ufw) allowing traffic from port 9735?
> sudo ufw status
- Is your VPS forwarding traffic to your node's VPN IP?
> Check iptables and net.ipv4.ip_forward from the previous point.
- Is your VPN tunnel active?
> tailscale status
> 
> tailscale ping \<ip\>

# Configuring LND

We have set up a secure and direct link between the VPS and our node. We have checked that the Internet (i.e everybody) is able to reach our node on port 9735 using the VPS public IP.

Now it is time to tell our node to announce that public IP and enable hybrid mode.
SSH in your node and locate your citadel installation. By default if you used the official image for Raspberry that Umbrel or Citadel gives, the path should be on ~/citadel or ~/umbrel. 
We are editing a template configuration file which is located under *templates* folder.
> nano ~/citadel/templates/lnd-sample.conf

On this file we are adding two lines under *[Application Options]* section.
> externalip=<vpn_public_ip>:9735
>
> nat=false

Then, at the end of the file under *[tor]* section:
> tor.skip-proxy-for-clearnet-targets=true
>
> tor.streamisolation=false

**Remember that you have to save the file. To do so, press *ctrl+O* then *ctrl+X* to exit from the nano editor.**

Now you could replicate those changes under *~/citadel/lnd/lnd.conf* so you don't have to restart your node.
If so, you have to restart only the Lightning Network service which is running as a Docker container.
> sudo docker restart lightning

Check for any error on the logs.
> sudo docker logs -n10 -f lightning

Check if LND is reporting your new URI with a clearnet address.
> sudo docker exec -it lightning lncli getinfo | jq '.uris'

You should get an output similar to this one, see that there are two URI (\<pubkey\>@\<address\>:\<port\>) with different addresses (one .onion address and a IP).

> [  
    "03339a8dca2023a3e2a7b4ee99f6a7be87d04bfd32775d3b7f85b0d6d30c457626@35.180.18.38:9735",  
    "03339a8dca2023a3e2a7b4ee99f6a7be87d04bfd32775d3b7f85b0d6d30c457626@gznwi4govx7c4rur3gpgbekixjahhe7lgfwt3r2qwm6auqjeme5pguyd.onion:9735"  
]

# The OG Real testing

Download a mobile lightning wallet such as Blixt. Try to add a new peer (your node) using the clearnet URI. If you succeed: congratulations! You have set up hybrid mode on your node.
You can find this option under the menu (top-right button), inside Lightning Network Peers section. 

## Pro tip
To make your life easier, login to Ride The Lightning or other GUI and display the clearnet's URI QR so you only have to point your camera to it from Blixt's app.


# Additional configurtions
- Set your VPS ufw to allow 41641/udp to help Tailscale connecting your two machines directly without using its third-party relay. 
- Set your VPS security rules (on webgui admin panel from your provider) to allow only inbound connections at port 9735/tcp+udp. You can also restrict port 22/tcp to your home's public IP. This way, your VPS will only allow connections to LND and SSH from your home.






# Buy me a coffe
Did this guide help you somehow?
Do you want to send me some sats?

- Onion lightning address: tips@ff22k4s6eytfodhuri6dafvvmhclrn66gebbfi3u7xkhbnla3rjiz7id.onion  
- Clearnet URI: 03339a8dca2023a3e2a7b4ee99f6a7be87d04bfd32775d3b7f85b0d6d30c457626@35.180.18.38:9735  
- Onion URI: 03339a8dca2023a3e2a7b4ee99f6a7be87d04bfd32775d3b7f85b0d6d30c457626@gznwi4govx7c4rur3gpgbekixjahhe7lgfwt3r2qwm6auqjeme5pguyd.onion:9735
- PayNym: +royalsunset01A                                                                                                                                                - PayNym: PM8TJV8RAHuSXKKLR56ZvvHT2oupphiXJis4dTpN3rckE3yoamywHbdgoUebpSsM1ZqEMNViu4tbxCrTzXsjuRvST89Xv4gdhJgbYZhL1687hBs4E6vB
