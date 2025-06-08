# PI

## Pihole

```shell
# setup
mkdir -p /home/jkeam/dev/projects/pi-hole/podman/dnsmasq
mkdir -p /home/jkeam/dev/projects/pi-hole/podman/pihole
sudo cp ./pihole.service /etc/systemd/system/

# firewall
sudo firewall-cmd --list-all --zone=FedoraServer
sudo firewall-cmd --zone=FedoraServer --add-forward-port=port=53:proto=tcp:toport=7053 --permanent
sudo firewall-cmd --zone=FedoraServer --add-forward-port=port=53:proto=udp:toport=7053 --permanent
sudo firewall-cmd --zone=FedoraServer --add-forward-port=port=67:proto=udp:toport=7067 --permanent
sudo firewall-cmd --zone=FedoraServer --add-forward-port=port=80:proto=udp:toport=7080 --permanent
sudo firewall-cmd --reload

# start
sudo systemctl enable --now pihole.service
```

## Tailscale

```shell
sudo cp ./tailscaled.service /usr/lib/systemd/system
sudo systemctl enable --now tailscaled.service
```
