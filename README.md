# 🌟 Cloudflare Tunnel - Setup dla Flask i Node.js z subdomenami


## 📋 Przykłady
- `api.twoja-domena.pl` → Flask API (port 5000)
- `app.twoja-domena.pl` → Node.js App (port 3000)
- Automatyczne SSL
- Zero konfiguracji DNS

---

## Krok 1: Przygotowanie aplikacji

### Flask API (~/flask-api/)

**app.py**:
```python
from flask import Flask, jsonify
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

@app.route('/')
def home():
    return jsonify({
        "message": "Flask API działa!",
        "service": "api",
        "port": 5000
    })

@app.route('/users')
def users():
    return jsonify({
        "users": [
            {"id": 1, "name": "Jan Kowalski"},
            {"id": 2, "name": "Anna Nowak"}
        ]
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy", "service": "flask-api"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

**requirements.txt**:
```
Flask==2.3.3
flask-cors==4.0.0
gunicorn==21.2.0
```

**start.sh**:
```bash
#!/bin/bash
cd ~/flask-api
source venv/bin/activate
gunicorn --bind 127.0.0.1:5000 --workers 2 app:app
```

### Node.js App (~/node-app/)

**app.js**:
```javascript
const express = require('express');
const cors = require('cors');
const app = express();
const PORT = 3000;

app.use(cors());
app.use(express.json());

app.get('/', (req, res) => {
    res.json({
        message: "Node.js App działa!",
        service: "webapp",
        port: 3000
    });
});

app.get('/products', (req, res) => {
    res.json({
        products: [
            { id: 1, name: "Laptop", price: 2999 },
            { id: 2, name: "Telefon", price: 1299 }
        ]
    });
});

app.get('/health', (req, res) => {
    res.json({ status: "healthy", service: "node-app" });
});

app.listen(PORT, '127.0.0.1', () => {
    console.log(`Node.js app running on http://127.0.0.1:${PORT}`);
});
```

**package.json**:
```json
{
  "name": "node-app",
  "version": "1.0.0",
  "description": "Simple Node.js app",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

**start.sh**:
```bash
#!/bin/bash
cd ~/node-app
npm start
```

---

## Krok 2: Instalacja aplikacji

### Setup Flask API:
```bash
# Tworzenie folderu i środowiska
mkdir ~/flask-api
cd ~/flask-api

# Kopiuj pliki app.py, requirements.txt, start.sh

# Tworzenie środowiska wirtualnego
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Test aplikacji
python app.py &
curl http://localhost:5000
# Zatrzymaj: pkill -f "python app.py"
```

### Setup Node.js App:
```bash
# Instalacja Node.js (jeśli nie masz)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Tworzenie projektu
mkdir ~/node-app
cd ~/node-app

# Kopiuj pliki app.js, package.json, start.sh

# Instalacja zależności
npm install

# Test aplikacji
npm start &
curl http://localhost:3000
# Zatrzymaj: pkill -f "node app.js"
```

---

## Krok 3: Instalacja Cloudflare Tunnel

### Pobieranie cloudflared:
```bash
# Download i instalacja
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Sprawdź wersję
cloudflared --version
```

### Logowanie do Cloudflare:
```bash
# Uruchomi przeglądarkę do logowania
cloudflared tunnel login

# Plik credentials zostanie zapisany automatycznie
```

---

## Krok 4: Konfiguracja tunnel

### Tworzenie tunnel:
```bash
# Utwórz nowy tunnel
cloudflared tunnel create moje-aplikacje

# Zapisz tunnel ID (pojawi się w output)
# Przykład: Created tunnel moje-aplikacje with id: 12345678-1234-1234-1234-123456789012
```

### Konfiguracja DNS:
```bash
# Dodaj DNS rekordy dla subdomen
cloudflared tunnel route dns moje-aplikacje api.twoja-domena.pl
cloudflared tunnel route dns moje-aplikacje app.twoja-domena.pl
```

### Plik konfiguracyjny:
```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

**~/.cloudflared/config.yml**:
```yaml
tunnel: 12345678-1234-1234-1234-123456789012  # Twój tunnel ID
credentials-file: /home/junior/.cloudflared/12345678-1234-1234-1234-123456789012.json

ingress:
  # Flask API
  - hostname: api.twoja-domena.pl
    service: http://127.0.0.1:5000
    originRequest:
      httpHostHeader: api.twoja-domena.pl
  
  # Node.js App  
  - hostname: app.twoja-domena.pl
    service: http://127.0.0.1:3000
    originRequest:
      httpHostHeader: app.twoja-domena.pl
  
  # Catch-all (wymagany)
  - service: http_status:404
```

---

## Krok 5: Skrypty zarządzania

### Skrypt startu aplikacji:
```bash
nano ~/start-apps.sh
```

```bash
#!/bin/bash

echo "🚀 Uruchamianie aplikacji..."

# Zatrzymaj poprzednie procesy
pkill -f "gunicorn.*5000" 2>/dev/null || true
pkill -f "node.*3000" 2>/dev/null || true

# Uruchom Flask API
echo "📱 Uruchamiam Flask API (port 5000)..."
cd ~/flask-api
source venv/bin/activate
nohup gunicorn --bind 127.0.0.1:5000 --workers 2 app:app > ~/logs/flask-api.log 2>&1 &

# Uruchom Node.js App
echo "🌐 Uruchamiam Node.js App (port 3000)..."
cd ~/node-app
nohup npm start > ~/logs/node-app.log 2>&1 &

echo "✅ Aplikacje uruchomione!"
echo "📊 Sprawdź logi:"
echo "   Flask: tail -f ~/logs/flask-api.log"
echo "   Node.js: tail -f ~/logs/node-app.log"
```

### Skrypt systemd dla tunnel:
```bash
sudo nano /etc/systemd/system/cloudflare-tunnel.service
```

```ini
[Unit]
Description=Cloudflare Tunnel
After=network.target

[Service]
Type=simple
User=junior
ExecStart=/usr/local/bin/cloudflared tunnel run
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Skrypt systemd dla aplikacji:
```bash
sudo nano /etc/systemd/system/my-apps.service
```

```ini
[Unit]
Description=My Flask and Node.js Apps
After=network.target

[Service]
Type=forking
User=junior
ExecStart=/home/junior/start-apps.sh
ExecStop=/bin/bash -c 'pkill -f "gunicorn.*5000"; pkill -f "node.*3000"'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

## Krok 6: Uruchomienie całego systemu

### Przygotowanie logów:
```bash
mkdir ~/logs
chmod +x ~/start-apps.sh
```

### Włączenie serwisów:
```bash
# Włącz i uruchom aplikacje
sudo systemctl daemon-reload
sudo systemctl enable my-apps
sudo systemctl start my-apps

# Sprawdź status
sudo systemctl status my-apps

# Włącz i uruchom tunnel
sudo systemctl enable cloudflare-tunnel
sudo systemctl start cloudflare-tunnel

# Sprawdź status
sudo systemctl status cloudflare-tunnel
```

### Test działania:
```bash
# Test lokalny
curl http://localhost:5000
curl http://localhost:3000

# Test przez tunnel (po chwili)
curl https://api.twoja-domena.pl
curl https://app.twoja-domena.pl
```

---

## Krok 7: Skrypty zarządzania

### Monitor aplikacji:
```bash
nano ~/monitor.sh
```

```bash
#!/bin/bash

echo "🔍 Status aplikacji:"

# Sprawdź procesy
echo "📱 Flask API (port 5000):"
if pgrep -f "gunicorn.*5000" > /dev/null; then
    echo "   ✅ Działa"
    curl -s http://localhost:5000/health | grep -o '"status":"[^"]*"' || echo "   ⚠️ Nie odpowiada"
else
    echo "   ❌ Nie działa"
fi

echo "🌐 Node.js App (port 3000):"
if pgrep -f "node.*3000" > /dev/null; then
    echo "   ✅ Działa" 
    curl -s http://localhost:3000/health | grep -o '"status":"[^"]*"' || echo "   ⚠️ Nie odpowiada"
else
    echo "   ❌ Nie działa"
fi

echo "☁️ Cloudflare Tunnel:"
if pgrep -f "cloudflared tunnel run" > /dev/null; then
    echo "   ✅ Działa"
else
    echo "   ❌ Nie działa"
fi

echo ""
echo "🌐 Zewnętrzne adresy:"
echo "   https://api.twoja-domena.pl"
echo "   https://app.twoja-domena.pl"

echo ""
echo "📊 Zasoby systemu:"
echo "   CPU: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | awk -F'%' '{print $1}')%"
echo "   RAM: $(free | grep Mem | awk '{printf("%.1f%%", $3/$2 * 100.0)}')"
```

### Restart aplikacji:
```bash
nano ~/restart-apps.sh
```

```bash
#!/bin/bash

echo "🔄 Restart aplikacji..."

case $1 in
  flask)
    echo "🔄 Restartuję Flask API..."
    pkill -f "gunicorn.*5000"
    cd ~/flask-api
    source venv/bin/activate
    nohup gunicorn --bind 127.0.0.1:5000 --workers 2 app:app > ~/logs/flask-api.log 2>&1 &
    ;;
  node)
    echo "🔄 Restartuję Node.js App..."
    pkill -f "node.*3000"
    cd ~/node-app
    nohup npm start > ~/logs/node-app.log 2>&1 &
    ;;
  tunnel)
    echo "🔄 Restartuję Cloudflare Tunnel..."
    sudo systemctl restart cloudflare-tunnel
    ;;
  all)
    echo "🔄 Restartuję wszystko..."
    sudo systemctl restart my-apps
    sudo systemctl restart cloudflare-tunnel
    ;;
  *)
    echo "Użycie: $0 {flask|node|tunnel|all}"
    ;;
esac
```

```bash
chmod +x ~/monitor.sh ~/restart-apps.sh
```

---

## Krok 8: Automatyczne uruchamianie po restarcie

```bash
# Włącz auto-start
sudo systemctl enable my-apps
sudo systemctl enable cloudflare-tunnel

# Test restartu (opcjonalnie)
sudo reboot

# Po restarcie sprawdź
./monitor.sh
```

---

## 🎯 Podsumowanie Cloudflare Tunnel

### ✅ Zalety:
- **5 linii konfiguracji** zamiast 50+
- **Automatyczne SSL** - zero konfiguracji
- **Darmowe subdomeny** na własnej domenie
- **Działa za NAT** - nie potrzebujesz publicznego IP
- **Zero reverse proxy** - Cloudflare załatwia wszystko

### 📊 Co osiągnąłeś:
```
https://api.twoja-domena.pl  → Flask API (localhost:5000)
https://app.twoja-domena.pl  → Node.js App (localhost:3000)
```

### 💸 Koszt:
- **Domena**: ~$10/rok
- **VPS**: ~2 EUR/miesiąc  
- **Cloudflare Tunnel**: DARMOWY
- **SSL**: DARMOWY

### 🚀 Użycie:
```bash
./start-apps.sh      # Uruchom aplikacje
./monitor.sh         # Sprawdź status
./restart-apps.sh all # Restart wszystkiego
```

**To jest najprostrze rozwiązanie na świecie dla subdomen! 🌟**
