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
# No need to redirect on port 80
# sudo firewall-cmd --zone=FedoraServer --add-forward-port=port=80:proto=udp:toport=7080 --permanent
sudo firewall-cmd --reload

# start
sudo systemctl enable --now pihole.service
```

## Tailscale

Enable the service.

```shell
sudo dnf install -y tailscale
sudo cp ./tailscaled.service /usr/lib/systemd/system
sudo systemctl enable --now tailscaled.service
```

Enable IP forwarding.

```shell
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# allow the rewriting
sudo firewall-cmd --permanent --add-masquerade
```

Enable UDP optimization.
Note, this does not persist through reboot!

```shell
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
sudo ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off
```

Start the service.

```shell
sudo tailscale up --ssh --advertise-routes=192.168.1.0/24
```


