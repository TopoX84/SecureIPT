# Secure/Strict IPTables Rules

**You will need to run this shell script as Root to get the full benefits**

A Shell script that sets up your firewall (IPtables) to completely mitigate common Denial of Service attacks.

A big enough *Distributed* Denial of Service attack will not be stopped by this, but it raises the bar *significantly* (TDS) In theory if your server is fast enough to throw away the packets it should never go "down" and you can give IPtables a leg-up towards this end by setting its CPU Priority to be extremely high. 

`renice {priority} pid`

`-20 is the Highest Priority, Lowest "Niceness"`

`19 is the Lowest Priority, Highest Niceness"`

Your CPU usage will be negligent unless you're being DDoS'd and at that point the resources you were reserving for clients aren't being used anyway in most cases.
  
# Extra Ports

SIP allows you to specify 3 Extra ports of your choice (TCP) to be opened bidirectionally.

These can be changed with any text editor.

```
ExtraOne="true"
ExtraOneP=8888

ExtraTwo="true"
ExtraTwoP=28018

ExtraThree="false"
ExtraThreeP=0
```

#Important Section!


**DO NOT** manually enter these commands unless you **FULLY** understand what you are doing and how it will affect your **current rules!!**

If you don't know exactly what your current rules are, Just run the shell script. You risk seriously breaking your VPS/Dedi if you mis-configure your firewall. (You might block SSH ports, For instance)

#Rule 1: Limit New Connections

```
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m limit --limit 50/minute --limit-burst 200 -j ACCEPT
```

We are Adding an INPUT rule.

We are applying this to any new TCP connections.

We then limit the amount of "packets" that can be sent in a burst to 200.

When that limit is reached, We limit further attempts to 50 "packets"

(In essence, You speed up for no reason, You get slowed down)

We Jump to ACCEPT the packet and send it to its destination without further questioning.

#Rule 2: Limit Existing Connections

```
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -m limit --limit 50/second --limit-burst 50 -j ACCEPT
```

We are Adding an INPUT rule.

This is going to apply to already established connections (UDP,TCP) and their related "connections".

We limit their requests to 50 packets a second and don't allow them to "burst" packets.

(This "assumes" your websites main payload will be delivered during the initial connection)

We Jump to ACCEPT the packet and send it to its destination without further questioning.

#Rule 3: Dump Invalid/Malformed Packets.

```
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP 
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,SYN FIN,SYN -j DROP 
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags SYN,RST SYN,RST -j DROP 
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,RST FIN,RST -j DROP 
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags FIN,ACK FIN -j DROP 
sudo iptables -A INPUT -i eth0 -p tcp -m tcp --tcp-flags ACK,URG URG -j DROP
```

We are adding several INPUT rules.

These rules only affect TCP related connections.

These rules only apply to Ethernet Interface: `eth0`

We check various TCP Flags to make sure the packet is Valid/Complete and not trying to hang us.

If the packet meets any of these conditions, we throw it away because its shit.



#Rule 3: Limit Portscanning.

It is bad to outright block the option to query your ports, 

But you also don't want 40k requests in 1 second.

This is the solution to that.

```
sudo iptables -N PORT_SCANNING

sudo iptables -A PORT_SCANNING -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j RETURN

sudo iptables -A PORT_SCANNING -j DROP
```

We create a new chain to monitor PORT SCANNING attempts.

We make sure any "real" port scan attempts are Complete/Valid. (tcp-flags)

We limit the amount of times any IP can ask for a specific port. (--limit 1/s)

Drop any other Port Scanning attempt.

Chains can create LOG FILES, You should look into this if that is useful in your situation.

# Rule 4: Block LAND Attacks

```
sudo iptables -A INPUT -s 10.0.0.0/8 -j DROP
sudo iptables -A INPUT -s 169.254.0.0/16 -j DROP
sudo iptables -A INPUT -s 172.16.0.0/12 -j DROP
sudo iptables -A INPUT -s 127.0.0.0/8 -j DROP
sudo iptables -A INPUT -s 192.168.0.0/24 -j DROP
sudo iptables -A INPUT -s 224.0.0.0/4 -j DROP
sudo iptables -A INPUT -d 224.0.0.0/4 -j DROP
sudo iptables -A INPUT -s 240.0.0.0/5 -j DROP
sudo iptables -A INPUT -d 240.0.0.0/5 -j DROP
sudo iptables -A INPUT -s 0.0.0.0/8 -j DROP
sudo iptables -A INPUT -d 0.0.0.0/8 -j DROP
sudo iptables -A INPUT -d 239.255.255.0/24 -j DROP
sudo iptables -A INPUT -d 255.255.255.255 -j DROP
```

We are creating an INPUT rule.

We check to see if the source/dest is a private IP and drop it.

We check to see if the source/dest is from a private SubNet and drop it.


# Rule 5: Block XMAS Packets.

`sudo iptables -A INPUT -p tcp --tcp-flags ALL FIN,PSH,URG -j DROP`

We are creating an INPUT rule.

It only applys to TCP connections.

We check certain TCP flags are true and DROP the packet.

#Rule 6: Blocking Smurf Attacks

`sudo iptables -A INPUT -p icmp -m limit --limit 1/second --limit-burst 2 -j ACCEPT`

We are creating an INPUT rule.

It only applys to ICMP packets.

It applys to all Connections. New/Existing.

We limit the amount of requests for an ICMP packet a client can make per second.

We allow a little room for icmp to breathe. (burst)

`sudo iptables -A INPUT -p icmp -j DROP`

We are creating an INPUT rule.

It applys to all Connections. New/Existing.

It only applys to ICMP packets.

We throw away the ICMP packets. (DROP)

#Rule 7: The More Advanced SYN Filter

Okay, So you managed to get around the malformed packet filter, Good for you. Now RIP.

` sudo iptables -D INPUT -p tcp --dport 80 -m state --state NEW -m limit --limit 50/minute --limit-burst 200 -j ACCEPT`
` sudo iptables -A INPUT -p tcp -m state --state NEW -m limit --limit 2/second --limit-burst 2 -j ACCEPT`

If your having real problems with SYN floods, Then you can use this to SEVERELY limit new connections. 
(!MUST! be run in that order)


#Rule 8: NO UDP except DNS please!
```
sudo iptables -A INPUT -p udp --sport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A OUTPUT -p udp --sport 53 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp -j DROP
sudo iptables -A OUTPUT -p udp -j DROP
```
We are creating an INPUT rule.

The rule only applys to UDP packets.

The rule only applys to Port 53 (DNS)

We accept UDP packets go to the DNS server.

We accept UDP packets coming from the DNS server.

We DROP any other UDP packet. 




# Extra Security

You should also install Denyhosts and Fail2Ban because these programs deal with various Authentication Failures and then ban IP Addresses appropriately. (For 3 days by default)

You should research both of these tools, Because there are a lot of beneficial features for businesses big and small.

```
sudo apt-get update
sudo apt-get install denyhosts
sudo apt-get install fail2ban
```

Leave the settings as default unless you've changed your SSH port, 
otherwise you need to Google how to configure these programs.

```
sudo service denyhosts restart
sudo service fail2ban restart
```

