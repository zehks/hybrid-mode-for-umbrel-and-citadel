**DISCLAIMER**

I am not taking any responsibility if something goes wrong. Don't issue **any** command if you are not sure about it. **Actually, nobody should issue any command if they do not know what it does.** I am not doing any troubleshooting.

This guide does not cover all security considerations such as restricting SSH to private key login only or denying logins from other IP than yours.

**PREREQUISITES**

- To be able to SSH in your Lightning node
- To be able to SSH in your VPS
- To have a static IP in your VPS
- To have access to VPS security configuration

**ASSUMPTIONS**

- You having a current working RunCitadel or GetUmbrel instance
- You are running Lightning Network Daemon on port 9735

**SETTING UP AN ENCRYPTED AND SECURE TUNNEL**

For that purpose we are going to make use of native Tailscale app from
the appstore.

1. Go ahead to Tailscale\'s website and sign up ([[www.tailscale.com]](http://www.tailscale.com)).
2. Then install Tailscale app in your node and launch it. Web browser will pop up a window to log in.

Now you should see a new machine in your Tailscale\'s account admin panel (website). This machine should be your node.

3. SSH in your VPS and install Tailscale:

> curl -fsSL https://tailscale.com/install.sh \| sh

Watch your VPS terminal\'s output. Look for any error there. If not, it should say you must log in through a link. Copy that link and paste it into your web browser. Log in. You should now see two machines in your Tailscale admin panel. Big yikes!

**TESTING VPN TUNNEL**

On your VPS terminal you should be able to issue the command \"tailscale status\", which should output two machines. One for your VPS and another for your node. Look at the node\'s IP in that terminal output. Issue the command "tailscale ping \<ip_address\>\".

Remember to replace \"\<ip_address\>\" with the node\'s actual VPN IP address. If the terminal replies with a \"pong\", you are done on the VPN Tunnel setting.

**FORWARDING TRAFFIC PACKETS**

Now you have set up your vpn tunnel and your two machines are able to see each other and talk. Fine! We need now to tell the VPS to forward traffic to the node. For that purpose we are going to use \"iptables\".

On the VPS:

We are going to need some information here.

1. Interfaces names. Typically \"eth0\" will be your Internet\'s interface and \"tailscale0\" your VPN interface. You can issue the command "ip address show".

2. VPS\' VPN IP address.

3. Node\'s VPN IP address. Both IP can be found on \"tailscale status\".

From the previous tweet, \"eth0\" will be replaced with \"vps_internet_iface\" and \"tailscale0\" with \"vps_vpn_iface\".

VPS\' VPN IP will be \"vps_vpn_ip\" and your node\'s VPN IP \"node_vpn_ip\". So when issuing the following commands, you need to replace the values.

Issue the following commands on your VPS:

> sudo iptables -A PREROUTING -t nat -i \<vps_internet_iface\> -p tcp -m tcp \--dport 9735 -j DNAT \--to \<node_vpn_ip\>

And again the same command replacing both \"tcp\" with \"udp\":

> sudo iptables -A PREROUTING -t nat -i \<vps_internet_iface\> -p udp -m udp \--dport 9735 -j DNAT \--to \<node_vpn_ip\>

The last iptables entry is the following one:

>   sudo iptables -t nat -A POSTROUTING -d \<node_vpn_ip\> -o \<vps_vpn_iface\> -j MASQUERADE

Then issue the command:

> sudo sysctl -w net.ipv4.ip_forward=1

That last command says to Linux to act as a router (i.e. to forward packets).

**TESTING BEFORE CONFIGURING LND**

Head to some online service for checking open ports like [https://portchecker.co/check](https://portchecker.co/check) and check connectivity. You must put in there your VPS public IP and port number 9735. If it is opened, you are ready to go to the next step.

If it is closed, you need to troubleshoot. Some points to check:

-   Is your VPS provider allowing traffic in from port 9735?

-   Is your VPS firewall (such as ufw) allowing traffic from port 9735?
    > sudo ufw status

-   Is your VPS forwarding traffic to your node's VPN IP?
    > Check iptables and net.ipv4.ip_forward from the previous point.

-   Is your VPN tunnel active?
    > tailscale status
    > tailscale ping \<ip\>
