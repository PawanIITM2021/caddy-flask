# Deployment aplikacji Flask na VPS z Caddy - Kompletny tutorial

## 1. Kupno i konfiguracja VPS

### Wybór providera
- **IONOS**: ~2 EUR/miesiąc za podstawowy VPS
- **Hetzner**: ~3 EUR/miesiąc 
- **DigitalOcean**: ~4 USD/miesiąc

### Specyfikacja VPS (wystarczająca dla 4 projektów Flask):
- 1 vCPU
- 1GB RAM
- 20GB SSD
- Ubuntu 22.04 LTS

## 2. Pierwsza konfiguracja serwera

### Połączenie z serwerem
```bash
ssh root@TWÓJ_IP_SERWERA
```

### Aktualizacja systemu
```bash
apt update && apt upgrade -y
```

### Utworzenie użytkownika (bezpieczeństwo)
```bash
adduser flask_user
usermod -aG sudo flask_user
```

### Przejście na nowego użytkownika
```bash
su - flask_user
```

## 3. Instalacja niezbędnych pakietów

### Python i pip
```bash
sudo apt install python3 python3-pip python3-venv git -y
```

### Instalacja Caddy
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

## 4. Przygotowanie struktury katalogów

```bash
mkdir -p ~/apps/{projekt1,projekt2,projekt3,projekt4}
mkdir ~/logs
```

## 5. Deployment pierwszej aplikacji Flask

### Przykład dla projekt1
```bash
cd ~/apps/projekt1
```

### Klonowanie repozytorium (lub upload plików)
```bash
git clone https://github.com/twoj-username/projekt1.git .
# lub scp -r ./projekt1 flask_user@TWÓJ_IP:~/apps/projekt1/
```

### Tworzenie środowiska wirtualnego
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Instalacja Gunicorn
```bash
pip install gunicorn
```

### Test aplikacji
```bash
gunicorn --bind 0.0.0.0:5001 app:app
```

## 6. Konfiguracja systemd dla każdej aplikacji

### Tworzenie pliku serwisu dla projekt1
```bash
sudo nano /etc/systemd/system/projekt1.service
```

### Zawartość pliku serwisu:
```ini
[Unit]
Description=Projekt1 Flask Application
After=network.target

[Service]
Type=simple
User=flask_user
WorkingDirectory=/home/flask_user/apps/projekt1
ExecStart=/home/flask_user/apps/projekt1/venv/bin/gunicorn --bind 127.0.0.1:5001 app:app
Restart=always
RestartSec=3
Environment=FLASK_ENV=production

[Install]
WantedBy=multi-user.target
```

### Uruchomienie serwisu
```bash
sudo systemctl daemon-reload
sudo systemctl enable projekt1
sudo systemctl start projekt1
sudo systemctl status projekt1
```

### Powtórz dla pozostałych projektów (porty 5002, 5003, 5004)

## 7. Konfiguracja Caddy

### Edycja Caddyfile
```bash
sudo nano /etc/caddy/Caddyfile
```

### Zawartość Caddyfile:
```
# Projekt 1
projekt1.twoja-domena.pl {
    reverse_proxy localhost:5001
    log {
        output file /var/log/caddy/projekt1.log
    }
}

# Projekt 2
projekt2.twoja-domena.pl {
    reverse_proxy localhost:5002
    log {
        output file /var/log/caddy/projekt2.log
    }
}

# Projekt 3
projekt3.twoja-domena.pl {
    reverse_proxy localhost:5003
    log {
        output file /var/log/caddy/projekt3.log
    }
}

# Projekt 4
projekt4.twoja-domena.pl {
    reverse_proxy localhost:5004
    log {
        output file /var/log/caddy/projekt4.log
    }
}

# Alternatywnie - wszystko na jednej domenie z ścieżkami:
# twoja-domena.pl {
#     handle /projekt1/* {
#         reverse_proxy localhost:5001
#     }
#     handle /projekt2/* {
#         reverse_proxy localhost:5002
#     }
#     handle /projekt3/* {
#         reverse_proxy localhost:5003
#     }
#     handle /projekt4/* {
#         reverse_proxy localhost:5004
#     }
# }
```

### Restart Caddy
```bash
sudo systemctl reload caddy
```

## 8. Konfiguracja domeny

### Opcja 1: Własna domena
1. Kup domenę (np. na nazwa.pl, OVH)
2. Ustaw rekordy DNS:
   - A record: projekt1.twoja-domena.pl → IP_SERWERA
   - A record: projekt2.twoja-domena.pl → IP_SERWERA
   - itd.

### Opcja 2: Darmowa subdomena
- Użyj serwisów jak FreeDNS, No-IP
- Lub skonfiguruj Cloudflare Tunnel

### Opcja 3: Dostęp przez IP (do testów)
```
# W Caddyfile użyj IP zamiast domeny:
IP_SERWERA:80 {
    handle /projekt1/* {
        reverse_proxy localhost:5001
    }
    # itd...
}
```

## 9. Automatyzacja deploymentu

### Skrypt update.sh
```bash
nano ~/update_app.sh
```

```bash
#!/bin/bash
APP_NAME=$1
cd ~/apps/$APP_NAME
git pull origin main
source venv/bin/activate
pip install -r requirements.txt
sudo systemctl restart $APP_NAME
echo "Aplikacja $APP_NAME została zaktualizowana!"
```

```bash
chmod +x ~/update_app.sh
```

### Użycie:
```bash
./update_app.sh projekt1
```

## 10. Monitorowanie i logi

### Sprawdzanie statusu aplikacji
```bash
sudo systemctl status projekt1
sudo systemctl status projekt2
# itd...
```

### Oglądanie logów
```bash
sudo journalctl -u projekt1 -f
sudo tail -f /var/log/caddy/projekt1.log
```

### Sprawdzanie zasobów
```bash
htop
df -h
free -h
```

## 11. Bezpieczeństwo

### Konfiguracja firewall
```bash
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

### Automatyczne aktualizacje bezpieczeństwa
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

## 12. Backup

### Prosty skrypt backup
```bash
nano ~/backup.sh
```

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
tar -czf ~/backup_$DATE.tar.gz ~/apps/
echo "Backup utworzony: backup_$DATE.tar.gz"
```

## Podsumowanie kosztów

- **VPS IONOS**: ~2 EUR/miesiąc
- **Domena**: ~10-15 EUR/rok (opcjonalnie)
- **Łącznie**: ~2-3 EUR/miesiąc

## Zalety tego rozwiązania

✅ Bardzo niski koszt  
✅ Pełna kontrola nad serwerem  
✅ Automatyczne SSL (Caddy)  
✅ Łatwe zarządzanie wieloma aplikacjami  
✅ Dobra wydajność  
✅ Możliwość skalowania  

## Wsparcie

Jeśli masz problemy z którymś krokiem, sprawdź:
- Logi systemd: `sudo journalctl -u nazwa_serwisu`
- Logi Caddy: `sudo journalctl -u caddy`
- Status serwisów: `sudo systemctl status nazwa_serwisu`

Powodzenia z deploymentem! 🚀
