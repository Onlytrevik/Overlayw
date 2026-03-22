"""
TikTok Auction Server v4
- HTTP Server startet zuerst, Browser öffnet erst wenn bereit
- Session ID für Gift Events
"""

import asyncio
import json
import os
import time
import webbrowser
import threading
import http.server
import socket
import websockets
from TikTokLive import TikTokLiveClient
from TikTokLive.events import GiftEvent, ConnectEvent, DisconnectEvent

# ══════════════════════════════════════════════════
#  ▼▼▼  HIER EINTRAGEN  ▼▼▼
TIKTOK_USERNAME  = ""   # z.B. "@deinname"
SESSION_ID       = ""   # Aus Browser kopieren (F12 → Application → Cookies → sessionid)
AUCTION_DURATION = 60   # Sekunden
MINIMUM_COINS    = 1    # Mindest-Coins
SNIPE_DELAY      = 30   # Snipe-Schutz Sekunden
# ══════════════════════════════════════════════════

WS_PORT   = 8765
HTTP_PORT = 8080

connected_clients = set()

# ── Prüfe ob Port frei ist ─────────────────────
def port_free(port):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        return s.connect_ex(('localhost', port)) != 0

def wait_for_port(port, timeout=10):
    """Wartet bis der Port erreichbar ist"""
    start = time.time()
    while time.time() - start < timeout:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            if s.connect_ex(('localhost', port)) == 0:
                return True
        time.sleep(0.2)
    return False

# ── HTTP Server ────────────────────────────────
def start_http():
    folder = os.path.dirname(os.path.abspath(__file__))
    os.chdir(folder)

    class QuietHandler(http.server.SimpleHTTPRequestHandler):
        def log_message(self, *args): pass
        def end_headers(self):
            # CORS erlauben damit WebSocket funktioniert
            self.send_header('Access-Control-Allow-Origin', '*')
            super().end_headers()

    # Falls Port belegt → anderen versuchen
    global HTTP_PORT
    for port in [8080, 8081, 8082, 8090, 9090]:
        if port_free(port):
            HTTP_PORT = port
            break

    server = http.server.HTTPServer(("localhost", HTTP_PORT), QuietHandler)
    server.serve_forever()

# ── WebSocket Broadcast ────────────────────────
async def broadcast(data: dict):
    if not connected_clients:
        return
    msg = json.dumps(data, ensure_ascii=False)
    await asyncio.gather(
        *[c.send(msg) for c in list(connected_clients)],
        return_exceptions=True
    )

async def ws_handler(websocket):
    connected_clients.add(websocket)
    print(f"  [Overlay] Verbunden ✅ ({len(connected_clients)} Fenster)")
    try:
        await websocket.send(json.dumps({
            "type":     "config",
            "minimum":  MINIMUM_COINS,
            "snipe":    SNIPE_DELAY,
            "duration": AUCTION_DURATION,
        }))
        await websocket.wait_closed()
    finally:
        connected_clients.discard(websocket)

# ── Main ───────────────────────────────────────
async def main():
    global TIKTOK_USERNAME, SESSION_ID

    if not TIKTOK_USERNAME:
        name = input("TikTok Benutzername (ohne @): ").strip().lstrip("@")
        TIKTOK_USERNAME = "@" + name

    if not SESSION_ID:
        print("\n  ⚠️  Session ID nötig für Gift-Erkennung!")
        print("  → tiktok.com im Browser öffnen")
        print("  → F12 → Application → Cookies → tiktok.com → sessionid")
        print("  (Kannst du auch leer lassen, aber Gifts werden dann nicht erkannt)\n")
        SESSION_ID = input("  Session ID (Enter zum Überspringen): ").strip()

    print("\n" + "═"*50)
    print("  TikTok Auction Overlay v4")
    print("═"*50)
    print(f"  Account:  {TIKTOK_USERNAME}")
    print(f"  Session:  {'✅ gesetzt' if SESSION_ID else '⚠️  nicht gesetzt (Gifts evtl. nicht erkannt)'}")
    print(f"  Settings: {AUCTION_DURATION}s | Min {MINIMUM_COINS} Coins | Snipe {SNIPE_DELAY}s")
    print("═"*50)

    # 1) HTTP Server im Hintergrund starten
    print("\n  [1/3] Starte Webserver...")
    t = threading.Thread(target=start_http, daemon=True)
    t.start()

    # 2) Warten bis HTTP wirklich läuft
    if wait_for_port(HTTP_PORT, timeout=8):
        print(f"  [2/3] Webserver läuft auf Port {HTTP_PORT} ✅")
    else:
        print(f"  [2/3] ⚠️  Webserver Timeout — trotzdem weiter...")

    url = f"http://localhost:{HTTP_PORT}/overlay.html"
    print(f"  [3/3] Öffne Overlay: {url}")
    webbrowser.open(url)
    print(f"\n  Falls Vivaldi das Overlay nicht öffnet:")
    print(f"  → Manuell eingeben: {url}\n")

    # 3) TikTok Client
    client = TikTokLiveClient(unique_id=TIKTOK_USERNAME)
    if SESSION_ID:
        try:
            client.web.set_session_id(SESSION_ID)
        except Exception:
            pass

    @client.on(ConnectEvent)
    async def on_connect(event):
        print(f"  ✅ TikTok LIVE verbunden!")
        await broadcast({"type": "status", "connected": True})

    @client.on(DisconnectEvent)
    async def on_disconnect(event):
        print(f"  ❌ TikTok getrennt")
        await broadcast({"type": "status", "connected": False})

    @client.on(GiftEvent)
    async def on_gift(event: GiftEvent):
        try:
            # Nur fertige Geschenke (kein Streak-Zwischenwert)
            try:
                if event.gift.info.type == 1 and event.gift.is_repeating == 1:
                    return
                count = event.gift.repeat_count or 1
                coins = (event.gift.info.diamond_count or 0) * count
                gname = event.gift.info.name or "Geschenk"
            except Exception:
                # Fallback für ältere API-Versionen
                coins = getattr(event.gift, 'diamond_count', 0) * getattr(event, 'repeat_count', 1)
                gname = "Geschenk"
                count = 1

            username = (getattr(event.user, 'nickname', '') or
                       getattr(event.user, 'unique_id', '') or "Unbekannt").strip()

            avatar = ""
            try:
                avatar = event.user.avatar_thumb.urls[0]
            except Exception:
                pass

            if coins < MINIMUM_COINS:
                return

            print(f"  🎁 {username}: {gname} ×{count} = {coins} Coins")
            await broadcast({
                "type": "bid", "username": username,
                "avatar": avatar, "coins": coins, "gift": gname,
            })

        except Exception as e:
            print(f"  [Gift Fehler] {e}")

    # WebSocket starten
    ws_server = await websockets.serve(ws_handler, "localhost", WS_PORT)
    print(f"  WebSocket bereit (Port {WS_PORT})")
    print(f"  Jetzt live gehen auf TikTok! Warte auf Verbindung...\n")

    try:
        await client.connect()
    except Exception as e:
        print(f"\n  ⚠️  TikTok Fehler: {e}")
        print("  → Bist du gerade live?")
        print("  → Ist die Session ID korrekt?")
        print("\n  Drücke Enter zum Beenden...")
        input()
    finally:
        ws_server.close()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n  Server gestoppt.")
