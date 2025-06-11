# Alma-Podman

# Fixe IP
nmcli device
nmcli connection modify eno1 ipv4.addresses 10.11.0.1/24
nmcli connection modify eno1 ipv4.gateway 10.11.0.254
nmcli connection modify eno1 ipv4.dns "10.11.0.254 8.8.8.8 1.1.1.1"
nmcli connection modify eno1 ipv4.method manual
nmcli connection down eno1 && nmcli connection up eno1

# Automatische IP
nmcli connection modify eno1 ipv4.method auto
nmcli connection up eno1

# Test
ip a
ip route
cat /etc/resolv.conf

# Modify Firewall
sudo firewall-cmd --list-all
sudo firewall-cmd --list-services
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
sudo tail -f /var/log/firewalld

# Mount tmpf -> /tmp
sudo vim /etc/fstab
    tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo mount -t tmpfs -o nosuid,nodev tmpfs /tmp
mount | grep /tmp
echo "test" > /tmp/bleibich.txt


# Podman - None root
sudo useradd -m podman
sudo usermod -aG wheel podman
sudo loginctl enable-linger $(id -u podman)
sudo -ui podman

vim .bashrc
    export XDG_RUNTIME_DIR=/run/user/1000
oder: machinectl shell podman@

# Bei Fehler:
Error: current system boot ID differs from cached boot ID; an unhandled reboot has occurred. Please delete directories "/tmp/storage-run-1001/containers" and "/tmp/storage-run-1001/libpod/tmp" and re-run Podman

rm -rf /tmp/storage-run-1001/containers
rm -rf /tmp/storage-run-1001/libpod/tmp
podman ps -a

# Info
/etc/containers/registries.conf
$HOME/.config/containers/registries.conf

mkdir -p $HOME/nginx
podman run -it --rm --name nginx -p 8080:80 docker.io/nginx:latest
podman run -it --rm --name nginx -v $HOME/nginx/:/usr/share/nginx/html -p 8080:80 docker.io/nginx:latest

# Quadlet
Nachlesen: https://docs.podman.io/en/v5.4.0/markdown/podman-systemd.unit.5.html

mkdir -p $HOME/.config/containers/systemd
cd $HOME/.config/containers/systemd

vim $HOME/.config/containers/systemd/nginx.container
[Unit]
Description=nginx Container
After=network-online.target

[Container]
Image=docker.io/nginx:latest
Volume=%h/nginx:/usr/share/nginx/html
PublishPort=8080:80
ContainerName=nginx

[Install]
WantedBy=multi-user.target default.target

systemctl --user daemon-reload
systemctl --user list-unit-files
systemctl --user start nginx
journalctl --user -xeu nginx.service

# Test Config
/usr/libexec/podman/quadlet -dryrun -user

# Podman - UniFi Controller
Nachlesen: https://docs.linuxserver.io/images/docker-unifi-network-application/#docker-compose-recommended-click-here-for-more-info

# Netzwerk erstellen
podman network ls
podman inspect unifi-db | grep -i network
podman inspect unifi-network-application | grep -i network
podman network create unifi-net
podman network ls

vim .config/containers/systemd/unifi.container

[Unit]
Description=Unifi Network Application
Wants=unifi-mongo.service
After=unifi-mongo.service

[Container]
Image=lscr.io/linuxserver/unifi-network-application:latest
ContainerName=unifi
Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=Etc/UTC
Environment=MONGO_USER=unifi
Environment=MONGO_PASS=letmeinplz
Environment=MONGO_HOST=unifi-db
Environment=MONGO_PORT=27017
Environment=MONGO_DBNAME=unifi
Environment=MONGO_AUTHSOURCE=admin
Environment=MEM_LIMIT=1024
Environment=MEM_STARTUP=1024
Network=unifi-net

PublishPort=8443:8443
PublishPort=3478:3478/udp
PublishPort=10001:10001/udp
PublishPort=8080:8080
PublishPort=1900:1900/udp
PublishPort=8843:8843
PublishPort=8880:8880
PublishPort=6789:6789
PublishPort=5514:5514/udp

Volume=%h/unifi/app:/config:Z

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target

vim .config/containers/systemd/unifi-mongo.container

[Container]
Image=docker.io/mongo:6.0
ContainerName=unifi-db
Environment=MONGO_INITDB_ROOT_USERNAME=unifi
Environment=MONGO_INITDB_ROOT_PASSWORD=letmeinplz
Environment=MONGO_INITDB_DATABASE=unifi
Volume=%h/unifi/db:/data/db:Z
Network=unifi-net

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target

# Firewall
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=3478/udp
sudo firewall-cmd --permanent --add-port=10001/udp
sudo firewall-cmd --permanent --add-port=1900/udp
sudo firewall-cmd --permanent --add-port=8843/tcp
sudo firewall-cmd --permanent --add-port=8880/tcp
sudo firewall-cmd --permanent --add-port=6789/tcp
sudo firewall-cmd --permanent --add-port=5514/udp
sudo firewall-cmd --reload

mkdir -p $HOME/unifi/app
mkdir -p $HOME/unifi/db

podman pull lscr.io/linuxserver/unifi-network-application:latest
podman pull docker.io/mongo:6:0

systemctl --user daemon-reload
systemctl --user start unifi-mongo
systemctl --user start unifi

# Debug
podman exec -it unifi-network-application bash
