# oracle-wireguard-docker

FUNZIONI

* privacy: l'infrastruttura è interamente gestita e controllata senza dipendere da intermediari commerciali.
* kernel-native speed: wireguard sfrutta il modulo nativo del kernel linux per ridurre latenza e utilizzo della cpu.
* isolamento via docker: il deployment tramite wg easy mantiene il servizio isolato dall'ambiente host.
* hardening dell'autenticazione: l'accesso alla dashboard di gestione utilizza password protette tramite hashing bcrypt.
* dashboard su https: la web ui viene servita tramite reverse proxy caddy con certificato tls valido rilasciato da lets encrypt e rinnovato automaticamente.
* firewall host restrittivo: iptables utilizza una policy di default drop, consentendo esplicitamente solo il traffico necessario.
* interfaccia web riconfigurabile: la dashboard permette di generare rapidamente file .conf e codici qr per i dispositivi mobile.

ARCHITETTURA

Il progetto utilizza una macchina virtuale arm64 su oracle cloud infrastructure, con ubuntu 22.04 lts e il piano always free tier.

* cloud provider: oracle cloud infrastructure (oci)
* compute instance: ampere arm64 / ubuntu 22.04 lts (always free tier)
* reverse proxy / tls: caddy con certificati lets encrypt automatici
* duckdns con sottodominio gratuito puntato all'ip pubblico dell'istanza

porte di rete:

22/tcp — accesso amministrativo ssh
51820/udp — traffico cifrato vpn wireguard
80/tcp — validazione certificato lets encrypt
443/tcp — dashboard web tramite caddy
51821/tcp — web dashboard wg easy, accessibile solo da localhost

=====> La dashboard wg easy non è esposta direttamente su internet. Il servizio viene eseguito tramite bind su 127.0.0.1:51821 ed è raggiungibile dall'esterno esclusivamente attraverso il reverse proxy caddy tramite https. <====

DEPLOYMENT

1 configurazione network su oracle cloud

La vcn su oci deve permettere il traffico necessario verso l'istanza. Nella ingress security list sono quindi presenti le seguenti regole:

| protocollo | porta | motivo |
| tcp | 22 | connessione amministrativa ssh |
| udp | 51820 | tunneling vpn wireguard |
| tcp | 80 | validazione certificato lets encrypt |
| tcp | 443 | interfaccia web di amministrazione tramite https |

2 configurazione dell'host linux

L'host linux gestisce l'ip forwarding, il supporto kernel per wireguard e il firewall locale.

#bash
#abilito l'ip forwarding a livello di kernel
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

sudo modprobe wireguard

#hardening del firewall host
#aggiungo prima le regole necessarie
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

#imposto infine la policy di default restrittiva
sudo iptables -P INPUT DROP
sudo netfilter-persistent save

qui avevo erroneamente impostato la policy restrittiva prima di dare il comando per la porta 22, ciò mi ha chiaramente chiuso fuori da ssh, rimediato con un semplice reboot in quanto ancora non avevo reso persistenti le modifiche.

3 autenticazione

La dashboard wg easy utilizza un hash bcrypt per proteggere la password di accesso.

npx bcryptjs-cli <mia_password>

L'hash generato viene utilizzato nella configurazione del container.

4 container docker

Il servizio viene eseguito all'interno di un container docker dedicato. La dashboard viene esposta solamente su localhost e non è quindi direttamente raggiungibile da internet.

docker run -d 
--name=wg-easy 
-e WG_HOST=<IP_PUBBLICO_SERVER> 
-e PASSWORD_HASH='<HASH>' 
-v ~/.wg-easy:/etc/wireguard 
-p 51820:51820/udp 
-p 127.0.0.1:51821:51821/tcp 
--cap-add=NET_ADMIN 
--device=/dev/net/tun:/dev/net/tun 
--restart unless-stopped 
ghcr.io/wg-easy/wg-easy

Lo stato del container può essere verificato con:

docker ps

5 dns 

Poiché l'istanza non dispone di un dominio proprio, viene utilizzato un sottodominio gratuito duckdns puntato all'ip pubblico della macchina.

Il dominio viene utilizzato da caddy per ottenere il certificato tls necessario alla connessione https.

nel mio caso:

notsoopenvpn.duckdns.org

6 reverse proxy https

Caddy gestisce automaticamente il certificato tls tramite lets encrypt e inoltra le richieste verso la dashboard wg easy, che rimane accessibile solamente tramite localhost.

Installazione di caddy:

sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

La configurazione viene inserita nel file /etc/caddy/Caddyfile:

notsoopenvpn.duckdns.org {
reverse_proxy 127.0.0.1:51821
}

Riavvio del servizio:

sudo systemctl restart caddy

CONNESSIONE DEI CLIENT

La gestione dei client avviene direttamente dalla web ui di wg easy.

L'interfaccia è raggiungibile tramite:

notsoopenvpn.duckdns.org

Da qui ci si può autenticare, creare nuovi client e generare le relative configurazioni.

Per i dispositivi desktop si può scaricare il file .conf e importarlo nell'applicazione wireguard.

Per i dispositivi mobile si può scansionare il qr code generato dalla dashboard e importare direttamente la configurazione nell'app wireguard.
