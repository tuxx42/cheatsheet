# Cheatsheet
Collection of cheats

## IP Masquerading / Forwarding with multiple interfaces

The following script was used to forward traffic between <code>tun0</code> <-> <code>tun1</code>
<code>tun0</code> is NordVPN, while <code>tun1</code> is an openvpn server with multiple clients.

The intent is to allow all the clients to share the NordVPN connection.

In order to achieve the NordVPN connection was instaniated using:

<pre>
URL='https://nordvpn.com/wp-admin/admin-ajax.php?action=servers_recommendations&filters={%22country_id%22:228}' | jq -r '.[0].hostname'
sudo openvpn --pull-filter ignore redirect-gateway --config "$(wget -qO - $URL).udp.ovpn"
</pre>

This will ensure that the redirect-gateway is not pushed.

Add the route markers 100 and 101 to the routing table <code>rt_table</code>

<pre>
tuxx@lightningfood:~$ cat /etc/iproute2/rt_tables
#
# reserved values
#
255	local
254	main
253	default
0	unspec
100	tun0
101	tun1
#
# local
#
#1	inr.ruhep
</pre>

The following script enables IP masquerading and routes all traffic from <code>tun1</code> to <code>tun0</code>

<pre>
#!/bin/bash

IN=tun1
OUT=tun0

if [ "$1" == "-d" ]; then
  OP="-D"
  OP2="delete"
else
  OP="-A"
  OP2="add"
fi

iptables $OP FORWARD -i $IN -o $OUT -j ACCEPT
iptables $OP FORWARD -i $IN -o $OUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -t nat $OP POSTROUTING -o $OUT -j MASQUERADE

iptables $OP FORWARD -i $IN -o $OUT -j ACCEPT
iptables $OP FORWARD -i $OUT -o $IN -j ACCEPT
echo 1 > /proc/sys/net/ipv4/ip_forward

ip route $OP2 default dev $OUT table 100
ip rule $OP2 iif $IN table 100
</pre>
