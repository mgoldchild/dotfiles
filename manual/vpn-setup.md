NOTE: this memo is VPN of L2TP/IPSec PSK
based on https://github.com/hwdsl2/setup-ipsec-vpn
and network-manager-l2tp-gnome was not working for me that's why i used vpn cmd manually

# check IEK algorithm type

result was below in my vpn server
it means IKE algorithm is aes128-sha1-modp1024!

```
sudo ike-scan $VPN_SERVER_IP --trans="(1=7,14=128,2=2,3=1,4=2)"
XXX.XXX.XXX.XXX  Main Mode Handshake returned HDR=(CKY-R=5911387ba9cc4c1e) SA=(Enc=AES KeyLength=128 Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=1800)
```

# to load vpn credential 

```
$ cd ~/dotifiles
$ sudo direnv allow && sudo direnv exec . bash setup/vpn.sh
# echo $VPN_SERVER_IP
```

# strongSwan (IPSec) 

execute cmd bellow as sudoer

```
# cat > /etc/ipsec.conf <<EOF
# ipsec.conf - strongSwan IPsec configuration file

# basic configuration

config setup
  # strictcrlpolicy=yes
  # uniqueids = no

# Add connections here.

# Sample VPN connections

conn %default
  ikelifetime=60m
  keylife=20m
  rekeymargin=3m
  keyingtries=1
  keyexchange=ikev1
  authby=secret
  ike=$VPN_ALGORITHM
  esp=$VPN_ALGORITHM

conn myvpn
  keyexchange=ikev1
  left=%defaultroute
  auto=add
  authby=secret
  type=transport
  leftprotoport=17/1701
  rightprotoport=17/1701
  right=$VPN_SERVER_IP
EOF

cat > /etc/ipsec.secrets <<EOF
: PSK "$VPN_IPSEC_PSK"
EOF

```

update permission

```
$ chmod 600 /etc/ipsec.secrets
```

# xl2tpd (L2TP) 

execute cmd bellow as sudoer

```
# cat > /etc/xl2tpd/xl2tpd.conf <<EOF
[lac myvpn]
lns = $VPN_SERVER_IP
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
EOF

cat > /etc/ppp/options.l2tpd.client <<EOF
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-chap
noccp
noauth
mtu 1280
mru 1280
noipdefault
defaultroute
usepeerdns
connect-delay 5000
name $VPN_USER
password $VPN_PASSWORD
EOF
```

update permission

```
$ chmod 600 /etc/ppp/options.l2tpd.client
```

# add control file of IPSec

```
# mkdir -p /var/run/xl2tpd
# touch /var/run/xl2tpd/l2tp-control
```