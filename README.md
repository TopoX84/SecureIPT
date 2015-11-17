# Secure IPTables

A Shell script for securing IPTables on a Standard Web-server.

You will need to give the script Root privledges to run it.


# Extra Security

You should also install Denyhosts and Fail2Ban because these programs deal with various Authentication Failures, and ban people appropriately. You should research both of them, Because there are a lot of beneficial features.

```
sudo apt-get update
sudo apt-get install denyhosts
sudo apt-get install fail2ban
```

Leave the settings as default unless you've changed your SSH port, 
Then you need to Google how to configure these programs.

```
sudo service denyhosts restart
sudo service fail2ban restart
```



# Description of Shell Script contents.

This is basically a rundown of what is contained in the Shell Script.
I'll try to be really brief when i explain what they do.
Don't manually enter these, They might not be in the right order. 
Use the Shell Script.

#Rule 1: Limit New Connections

```
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m limit --limit 50/minute --limit-burst 200 -j ACCEPT
```

We are Adding an INPUT rule.

 `-p tcp --dport 80 = We are looking for TCP traffic over port 80`
 
we are applying this to any new connections meeting the above conditions.
We then limit the amount of "packets" that can be sent to 200, And when that limit is reached, We limit further attempts to 50 "packets" (In essence, You speed up for no reason, We slow you down)
We Jump to ACCEPT the packet and send it to its destination without further questioning.

#Rule 2: Limit Existing Connections

```
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -m limit --limit 50/second --limit-burst 50 -j ACCEPT
```

We are Adding an INPUT rule
This is going to apply to already established connections and their related connections.
We limit their requests to 50 packets a second (A "normal" person shouldn't need more then that)
Again we Jump to ACCEPT those packets.

#Rule 3: Dump Ugly Packets.

```
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,SYN FIN,SYN -j DROP sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags SYN,RST SYN,RST -j DROP sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,RST FIN,RST -j DROP sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,ACK FIN -j DROP sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags ACK,URG URG -j DROP
```

There's "a-lot" going here, So i will sum it up by saying, We are dropping packets from lame script kiddies, and anything else that isn't complete or compliant. No self respecting firewall should be lacking these rules by default (Yet they all are) This should protect you from lame flood attacks, well at least the ones not launched by "professionals"

#Rule 3: Block Portscanning Attempts

```
sudo iptables -N PORT_SCANNING

sudo iptables -A PORT_SCANNING -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j RETURN

sudo iptables -A PORT_SCANNING -j DROP
```
We create a new chain to monitor PORT SCANNING attempts, And then DROP them all on their head if they flood us. You may want to also create a log file, But we won't do that because theirs quota considerations you need to make.

# Rule 4: Block LAND Attacks

`sudo iptables -A INPUT -s YOURSERVERIP/32 -j DROP`

This one isn't a copy and paste. This will protect you from Spoofed IP's pretending to be you, Although this should be caught already by the above rules its better to be safe then DoS'd

# Rule 5: Block XMAS Packets.

`sudo iptables -A INPUT -p tcp --tcp-flags ALL FIN,PSH,URG -j DROP`

Basically, A really cool and easy way to kill a server, Is send a whole heap of packets at it with every single option, To avoid DoSing ourselves, We first specify we want to check all flags and then we make exceptions for the options that are "Mandatory" and then drop anything else we deem suspect.

#Rule 6: Blocking Smurf Attacks

`sudo iptables -A INPUT -p icmp -m limit --limit 2/second --limit-burst 2 -j ACCEPT`

OR IF YOUR NOT USING IT, DROP IT I'll let you decide which one is right You also have the option of specifying what types of ICMP packets you want to drop, But you can read about that somewhere.

`sudo iptables -A INPUT -p icmp -j DROP`

First, Go read about what ICMP is. Then scratch your head about its legitimate existence. Anyway, What this rule does is limits the requests to what is still a a pretty high level of 1 every 2 seconds. You don't want your server to be spammed with these.

#Rule 7: The More Advanced SYN Filter

Open, So you managed to get around the malformed packet filter, Good for you.

` sudo iptables -A INPUT -p tcp -m state --state NEW -m limit --limit 2/second --limit-burst 2 -j ACCEPT`

If your having real problems with SYN floods, Then you can use this to SEVERELY limit new connections. If you already added the rule at the top you need to run this before you add the above rule:

sudo iptables -D INPUT -p tcp --dport 80 -m state --state NEW -m limit --limit 50/minute --limit-burst 200 -j ACCEPT

#Rule 8: NO UDP Please
```
sudo iptables -A INPUT -p udp --sport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A OUTPUT -p udp --sport 53 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp -j DROP
sudo iptables -A OUTPUT -p udp -j DROP
```
By now you should be catching on and be able to see that this will only let UDP traffic go to your DNS Server, And we just black hole anything else. You can further lock this down by changing some BIND settings (Not covered here)
