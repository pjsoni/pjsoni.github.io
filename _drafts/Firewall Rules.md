Firewall Rules
You assume role of IT ecurity expert who specializs on building corporate network security. I'm building firewall rules from scratch. The network is segmented and have multiple vlans. 

vLAN Topology:
Name | vlan# | Notes
Default | vlan1 | Trusted core Network
Iot Network | vlan10 | IoT devices with internet access
Guest Network | vlan20
IoT No Internet | vlan40
proxmox-mgmt | vlan90
proxmox-cluster  | vlan99
DMZ | vlan30 | Isolated vlan just to host cloudflare tunnel

Default network contains NAS, Homeassistant, DNS Hosts, Docker Hosts, Personal devices

Rules to be enforced
Block access to all known DOH and DOT hosts
Allow Proxmox cluster Vlan 90 to Synology NAS NFS protocol
Allow tunnel host to reverse proxy
Allow sonos to homeassistant
block direct access to external DNS
block other gateway except network
Block all intervlan communication

Constrains
Only allow IPv4
Block all IPv6


network object lists
Name    | Type
mDNS  | Port
HomeAssistant | IP
MQTT Ports | Ports
DNS | Port
DNS Hosts | IP
Sonos Devices | IP
Sonos Ports | Ports
Admin-PC | IP
Proxmox Ports | ports
HTTP  | Ports
Tunnel host | IP
Reverse Proxy | IP

Give me clear conscise and meaningful firewall rules. Suggest any additional rules which I may missed.


