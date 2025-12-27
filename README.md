# Mail Relay Configuration
0. Create mailcow host & install mailcow
1. Create EC2 instance using stack.yaml
2. Add DNS (Route 53) entries:
    - `mail.alexfricker.com` (A Record) - Elastic IP
    - `alexfricker.com` (MX Record) "10 mail.alexfricker.com"
    - `alexfricker.com` (SPF Record) "v=spf1 mx include:amazonses.com -all"
    - `_dmarc.alexfricker.com` (TXT Record) "v=DMARC1; p=quarantine; rua=mailto:dmarc@alexfricker.com; adkim=s; aspf=s; pct=100"
3. SSH to EC2 instance & generate keys:
    - umask 077
    - wg genkey | tee aws.key | wg pubkey > aws.pub
    - sudo nano /etc/wireguard/wg0.conf
        - copy contents of aws.key
3. SSH to local mailcow & generate keys
    - sudo apt update
    - sudo apt install -y wireguard iptables-persistent
    - umask 077
    - wg genkey | tee home.key | wg pubkey > home.pub
    - sudo nano /etc/wireguard/wg0.conf
```
[Interface]
Address = 10.99.0.2/24
PrivateKey = <CONTENTS OF home.key>

[Peer]
PublicKey = <CONTENTS OF aws.pub>
Endpoint = <AWS_ELASTIC_IP>:51820
AllowedIPs = 10.99.0.1/32
PersistentKeepalive = 25
```
4. Update EC2 with home public key (home.pub - /etc/wireguard/wg0.conf)
5. Enable & start services:
    - sudo systemctl enable wg-quick@wg0
    - sudo systemctl start wg-quick@wg0
6. Confirm service is working:
    - sudo wg show
7. Update routing on home mailcow server (10025->25):
    - sudo sysctl -w net.ipv4.ip_forward=1
    - sudo iptables -t nat -A PREROUTING -i wg0 -p tcp --dport 10025 -j REDIRECT --to-ports 25
    - sudo iptables -A INPUT -i wg0 -p tcp --dport 25 -j ACCEPT
    - sudo sh -c "iptables-save > /etc/iptables/rules.v4"
8. Confirm routing is working:
    - (on home): tcpdump -ni wg0 tcp port 10025
    - (on AWS): nc -vz 10.99.0.2 10025
9. Set up SES & generate SMTP user & password
10. Add "Sender-dependent" transport to mailcow (System > Configuration > Routing)
    - host: email-smtp.us-east-1.amazonaws.com:587
    - username: <SMTP_USER>
    - password: <SMTP_PASSWORD>
11. Add email domain:
    - "relay non-local"