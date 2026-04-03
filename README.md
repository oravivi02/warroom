import asyncio
import json
import time
import os
import urllib.parse
import urllib.request
import urllib.error
import re
from typing import List, Optional
from datetime import datetime

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel

# ==========================================
# ⚙️ קונפיגורציה והגדרות גלובליות
# ==========================================
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY", "").strip()
MAPTILER_KEY = os.getenv("MAPTILER_KEY", "1ZewpyCj7brOgryKeolM").strip()
MAPBOX_STYLE = f"https://api.maptiler.com/maps/streets-v4/style.json?key={MAPTILER_KEY}"
GEMINI_MODEL = os.getenv("GEMINI_MODEL", "gemini-1.5-flash").strip()

_gemini_client = None
_gemini_model = None

genai_new = None
genai_old = None
try:
    import google.genai as genai_new
except ImportError:
    try:
        import warnings
        with warnings.catch_warnings():
            warnings.simplefilter("ignore")
            import google.generativeai as genai_old
    except ImportError:
        pass

if GEMINI_API_KEY and GEMINI_API_KEY != "YOUR_GEMINI_KEY":
    if genai_new is not None:
        _gemini_client = genai_new.Client(api_key=GEMINI_API_KEY)
    elif genai_old is not None:
        genai_old.configure(api_key=GEMINI_API_KEY)
        _gemini_model = genai_old.GenerativeModel(GEMINI_MODEL)

# בסיס נתונים של ערים מרכזיות בישראל
CITY_COORDINATES = {
    "תל אביב": (32.0853, 34.7818),
    "ירושלים": (31.7683, 35.2137),
    "חיפה": (32.7940, 34.9896),
    "באר שבע": (31.2518, 34.7913),
    "אשדוד": (31.8014, 34.6435),
    "נתניה": (32.3215, 34.8532),
    "פתח תקווה": (32.0840, 34.8878),
    "ראשון לציון": (31.9730, 34.7925),
    "חולון": (32.0158, 34.7872),
    "בני ברק": (32.0846, 34.8352),
    "רמת גן": (32.0681, 34.8245),
    "אשקלון": (31.6688, 34.5744),
    "רחובות": (31.8948, 34.8119),
    "הרצליה": (32.1640, 34.8447),
    "כפר סבא": (32.1750, 34.9070),
    "מודיעין": (31.8981, 35.0055),
    "לוד": (31.9516, 34.8891),
    "רמלה": (31.9253, 34.8667),
    "נהריה": (33.0058, 35.0946),
    "עכו": (32.9211, 35.0729),
    "קריית שמונה": (33.2075, 35.5721),
    "טבריה": (32.7959, 35.5310),
    "צפת": (32.9645, 35.4985),
    "אילת": (29.5581, 34.9482),
    "עפולה": (32.6078, 35.2897),
    "בית שאן": (32.4978, 35.4989),
    "חדרה": (32.4345, 34.9195),
    "נצרת": (32.6996, 35.3039),
    "שדרות": (31.5253, 34.5968),
    "אופקים": (31.3124, 34.6204),
    "נתיבות": (31.4222, 34.5888),
    "דימונה": (31.0695, 35.0336),
    "ירוחם": (30.9870, 34.9277),
    "מצפה רמון": (30.6101, 34.8012),
    "קריית גת": (31.6100, 34.7700),
    "קריית מלאכי": (31.7300, 34.7500),
    "יבנה": (31.8760, 34.7400),
    "גדרה": (31.8145, 34.7750),
    "מזכרת בתיה": (31.8530, 34.8400),
    "צורן": (32.2833, 34.9500),
    "טייבה": (32.2667, 35.0000),
    "טירה": (32.2333, 34.9500),
    "קלנסווה": (32.2833, 34.9833),
    "באקה אל-גרבייה": (32.4167, 35.0000),
    "אום אל-פחם": (32.5167, 35.1500),
    "קרית אתא": (32.8000, 35.1167),
    "קרית ביאליק": (32.8167, 35.0833),
    "קרית מוצקין": (32.8333, 35.0833),
    "קרית ים": (32.8500, 35.0667),
    "מעלות": (33.0167, 35.2833),
    "כרמיאל": (32.9167, 35.2833),
    "טמרה": (32.8500, 35.2000),
    "סח'נין": (32.8667, 35.3000),
    "דליית אל-כרמל": (32.7000, 35.0500),
    "עוספיה": (32.7167, 35.0667),
    "מגדל העמק": (32.6833, 35.2333),
    "נצרת עילית": (32.7167, 35.3333),
    "מעיליא": (33.0333, 35.2667),
    "פסוטה": (33.0833, 35.3167),
    "אביבים": (33.0869, 35.4699),
    "אביבים בצפון": (33.0869, 35.4699),
}

# פוליגונים לאזורים גיאוגרפיים עיקריים (רשימת [lng, lat])
AREA_POLYGONS = {
    "גוש דן": [
        [34.7818, 32.0853], [34.8352, 32.0846], [34.8245, 32.0681],
        [34.7872, 32.0158], [34.7925, 31.9730], [34.7400, 31.9760],
        [34.7200, 32.0400], [34.7500, 32.0700], [34.7818, 32.0853]
    ],
    "חיפה והסביבה": [
        [34.9500, 32.8200], [35.0729, 32.9211], [35.1167, 32.8000],
        [35.0833, 32.8167], [35.0833, 32.8333], [35.0667, 32.8500],
        [34.9896, 32.7940], [34.9500, 32.8200]
    ],
    "ירושלים והסביבה": [
        [35.1500, 31.8200], [35.2137, 31.7683], [35.3000, 31.7500],
        [35.2500, 31.7000], [35.1800, 31.7200], [35.1200, 31.7600],
        [35.1500, 31.8200]
    ],
    "הגליל": [
        [35.2000, 32.9000], [35.5721, 33.2075], [35.6000, 33.1000],
        [35.5000, 32.9000], [35.3000, 32.8000], [35.2000, 32.9000]
    ],
    "הנגב": [
        [34.6204, 31.3124], [34.7913, 31.2518], [35.0336, 31.0695],
        [34.9277, 30.9870], [34.8012, 30.6101], [34.5888, 31.4222],
        [34.6204, 31.3124]
    ],
    "עוטף עזה": [
        [34.5000, 31.6000], [34.5968, 31.5253], [34.5744, 31.6688],
        [34.5200, 31.7000], [34.5000, 31.6000]
    ],
}

# מיפוי סוגי איומים
THREAT_MAPPING = {
    0: {"type": "צבע אדום", "color": "#ef4444"},
    1: {"type": "חדירת כלי טיס עוין", "color": "#f97316"},
    2: {"type": "חשש לחדירת מחבלים", "color": "#eab308"},
    3: {"type": "רעידת אדמה", "color": "#eab308"},
    4: {"type": "אירוע חומרים מסוכנים", "color": "#a855f7"},
    5: {"type": "צונאמי", "color": "#0ea5e9"},
    6: {"type": "אירוע חומרים מסוכנים", "color": "#a855f7"},
    7: {"type": "ירי בלתי קונבנציונלי", "color": "#facc15"},
}

# ==========================================
# 🕐 היסטוריית 24 שעות
# ==========================================
alerts_history: list[dict] = []  # רשימה גלובלית של כל האזעקות ב-24 שעות

def add_to_history(event: dict):
    """מוסיף אירוע להיסטוריה ומנקה ישנים מעל 24 שעות"""
    global alerts_history
    now = time.time()
    alerts_history = [a for a in alerts_history if now - a.get("timestamp", 0) < 86400]
    alerts_history.append(event)

# ==========================================
# 🗺️ כלי גיאוגרפיה
# ==========================================
def is_within_israel(lat: float, lng: float) -> bool:
    return 29.0 <= lat <= 34.0 and 34.0 <= lng <= 36.0

def extract_city_from_text(text: str) -> Optional[tuple]:
    if not text:
        return None
    for city, (lat, lng) in CITY_COORDINATES.items():
        if city in text:
            return (lat, lng)
    return None

DEFAULT_CITY_COORDINATES = (32.0853, 34.7818)
GEOCODE_CACHE: dict = {}
GEOCODE_CACHE_PATH = os.path.join(os.path.dirname(__file__), "city_coords_cache.json")
_NOMINATIM_MIN_INTERVAL_S = 1.0
_nominatim_last_request_ts = 0.0

def _load_geocode_cache_from_disk():
    try:
        if not os.path.exists(GEOCODE_CACHE_PATH):
            return
        with open(GEOCODE_CACHE_PATH, "r", encoding="utf-8") as f:
            raw = json.load(f) or {}
        for k, v in raw.items():
            if isinstance(v, list) and len(v) == 2:
                try:
                    GEOCODE_CACHE[k] = (float(v[0]), float(v[1]))
                except Exception:
                    GEOCODE_CACHE[k] = None
            else:
                GEOCODE_CACHE[k] = None
    except Exception:
        return

def _flush_geocode_cache_to_disk():
    try:
        tmp_path = GEOCODE_CACHE_PATH + ".tmp"
        payload = {}
        for k, v in GEOCODE_CACHE.items():
            payload[k] = [v[0], v[1]] if v else None
        with open(tmp_path, "w", encoding="utf-8") as f:
            json.dump(payload, f, ensure_ascii=False)
        os.replace(tmp_path, GEOCODE_CACHE_PATH)
    except Exception:
        return

_load_geocode_cache_from_disk()

def _normalize_city_for_coords(city: str) -> str:
    city = (city or "").strip()
    if not city:
        return ""
    if "," in city:
        city = city.split(",", 1)[0].strip()
    city = city.replace("–", "-").replace("—", "-")
    city = re.sub(r"\s+", " ", city).strip()
    city = re.sub(r"\s*(בצפון|במרכז|בדרום|במזרח|במערב)\s*$", "", city).strip()
    city = re.sub(r"\s*(צפון|מרכז|דרום|מזרח|מערב)\s*$", "", city).strip()
    city = re.sub(r"\s*(מועצה אזורית|אזור|מחוז)\s*$", "", city).strip()
    return city

def _geocode_city_nominatim(city: str) -> Optional[tuple]:
    city = (city or "").strip()
    if not city:
        return None
    normalized = _normalize_city_for_coords(city)
    cache_key = (normalized or city).strip().lower()
    if cache_key in GEOCODE_CACHE:
        return GEOCODE_CACHE[cache_key]
    global _nominatim_last_request_ts
    now = time.time()
    wait_s = (_NOMINATIM_MIN_INTERVAL_S - (now - _nominatim_last_request_ts))
    if wait_s > 0:
        time.sleep(wait_s)
    _nominatim_last_request_ts = time.time()
    headers = {"User-Agent": "WarRoomBot/1.0 (contact: https://openstreetmap.org)"}
    query_city = urllib.parse.quote(normalized or city)
    url = (
        "https://nominatim.openstreetmap.org/search"
        f"?format=json&limit=1&countrycodes=il&q={query_city}"
    )
    coords = None
    try:
        req = urllib.request.Request(url, headers=headers, method="GET")
        with urllib.request.urlopen(req, timeout=7) as resp:
            raw = resp.read().decode("utf-8", errors="ignore")
            data = json.loads(raw) if raw else []
            if isinstance(data, list) and data:
                lat = float(data[0].get("lat"))
                lng = float(data[0].get("lon"))
                coords = (lat, lng)
    except Exception:
        coords = None
    GEOCODE_CACHE[cache_key] = coords
    _flush_geocode_cache_to_disk()
    return coords

def resolve_city_coordinates(city: str) -> tuple:
    city = (city or "").strip()
    if not city:
        return DEFAULT_CITY_COORDINATES
    if city in CITY_COORDINATES:
        return CITY_COORDINATES[city]
    city_key = next((c for c in CITY_COORDINATES if c in city or city in c), None)
    if city_key:
        return CITY_COORDINATES[city_key]
    normalized = _normalize_city_for_coords(city)
    if normalized:
        if normalized in CITY_COORDINATES:
            return CITY_COORDINATES[normalized]
        city_key = next((c for c in CITY_COORDINATES if c in normalized or normalized in c), None)
        if city_key:
            return CITY_COORDINATES[city_key]
    coords = _geocode_city_nominatim(city) or _geocode_city_nominatim(normalized)
    return coords if coords else DEFAULT_CITY_COORDINATES

def get_area_polygon(cities: list) -> Optional[list]:
    """מחזיר פוליגון אזור לפי שמות ערים, או בונה קונבקס הול מנקודות"""
    if not cities:
        return None
    city_lower = " ".join(cities).lower()
    for area_name, polygon in AREA_POLYGONS.items():
        if area_name in city_lower or any(c in city_lower for c in area_name.split()):
            return polygon
    points = []
    for city in cities:
        coords = resolve_city_coordinates(city)
        if coords:
            points.append([coords[1], coords[0]])  # [lng, lat]
    if len(points) < 2:
        if len(points) == 1:
            lng, lat = points[0]
            r = 0.05
            return [
                [lng - r, lat - r], [lng + r, lat - r],
                [lng + r, lat + r], [lng - r, lat + r],
                [lng - r, lat - r]
            ]
        return None
    min_lng = min(p[0] for p in points)
    max_lng = max(p[0] for p in points)
    min_lat = min(p[1] for p in points)
    max_lat = max(p[1] for p in points)
    pad = 0.05
    return [
        [min_lng - pad, min_lat - pad],
        [max_lng + pad, min_lat - pad],
        [max_lng + pad, max_lat + pad],
        [min_lng - pad, max_lat + pad],
        [min_lng - pad, min_lat - pad],
    ]

# ==========================================
# 🧠 מנוע עיבוד המידע
# ==========================================
class WarEvent(BaseModel):
    id: str
    text: str
    lat: float
    lng: float
    type: str
    reliability: float
    timestamp: float
    origin_lat: float | None = None
    origin_lng: float | None = None
    target_lat: float | None = None
    target_lng: float | None = None

async def process_raw_intel(raw_text: str, source_type: str):
    if not (GEMINI_API_KEY and GEMINI_API_KEY != "YOUR_GEMINI_KEY") or (
        _gemini_client is None and _gemini_model is None
    ):
        return None
    prompt = f"""
    Analyze this intel report: "{raw_text}"
    Source: {source_type}
    1. Translate to professional Hebrew (military context).
    2. Extract city/region and provide approximate Lat/Lng in Israel/Middle East.
    3. Categorize: 'תקיפה', 'תנועת כוחות', 'התרעה'.
    4. Rate reliability (0-1) based on source.
    Return ONLY JSON: {{"text": "...", "lat": 0.0, "lng": 0.0, "type": "...", "reliability": 0.0}}
    """
    try:
        if _gemini_client is not None:
            response = await asyncio.to_thread(
                _gemini_client.models.generate_content,
                model=GEMINI_MODEL,
                contents=prompt,
            )
        else:
            response = await asyncio.to_thread(_gemini_model.generate_content, prompt)
        text = (getattr(response, "text", None) or "").strip()
        data = json.loads(text.replace("```json", "").replace("```", ""))
        if not is_within_israel(data.get("lat", 0), data.get("lng", 0)):
            data["lat"] = 31.0461
            data["lng"] = 34.8516
        data["id"] = str(time.time())
        data["timestamp"] = time.time()
        return data
    except Exception as e:
        print(f"Error processing intel: {e}")
        return None

# ==========================================
# 📡 ניהול תקשורת בזמן אמת (WebSockets)
# ==========================================
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)

    async def broadcast(self, message: dict):
        disconnected = []
        for connection in self.active_connections:
            try:
                await connection.send_json(message)
            except Exception:
                disconnected.append(connection)
        for c in disconnected:
            self.disconnect(c)

manager = ConnectionManager()

# ==========================================
# 🚀 יצירת האפליקציה (חייב לפני כל decorator)
# ==========================================
async def _lifespan(app):
    asyncio.create_task(start_osint_pipeline())
    asyncio.create_task(fetch_homefront_command_alerts())
    yield

app = FastAPI(lifespan=_lifespan)

static_dir = os.path.join(os.path.dirname(__file__), "static")
if os.path.isdir(static_dir):
    app.mount("/static", StaticFiles(directory=static_dir), name="static")

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        # שלח היסטוריית 24 שעות לחיבור חדש
        now = time.time()
        history_clean = [a for a in alerts_history if now - a.get("timestamp", 0) < 86400]
        for hist_event in history_clean[-50:]:
            try:
                await websocket.send_json({**hist_event, "is_history": True})
            except Exception:
                break
        while True:
            await websocket.receive_text()
    except WebSocketDisconnect:
        manager.disconnect(websocket)

# ==========================================
# 🔴 סורק פיקוד העורף (כל שנייה)
# ==========================================

# מעקב אחר אזעקות פעילות לצורך מחיקה
active_alerts: dict = {}  # alert_id -> event_data

async def fetch_homefront_command_alerts():
    sent_alerts = set()
    use_proxy = False
    prev_alert_ids: set = set()

    while True:
        try:
            current_alert_ids: set = set()

            if not use_proxy:
                url = "https://www.oref.org.il/WarningMessages/alert/alerts.json"
                headers = {
                    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
                    'Referer': 'https://www.oref.org.il/',
                    'X-Requested-With': 'XMLHttpRequest',
                    'Accept': 'application/json, text/plain, */*',
                    'Accept-Language': 'he-IL,he;q=0.9,en-US;q=0.8,en;q=0.7',
                    'Connection': 'keep-alive',
                    'Cache-Control': 'no-cache',
                    'Pragma': 'no-cache',
                }
                req = urllib.request.Request(url, headers=headers, method='GET')

                try:
                    with urllib.request.urlopen(req, timeout=5) as resp:
                        if resp.getcode() == 200:
                            raw_data = resp.read().decode('utf-8', errors='ignore')
                            if raw_data and raw_data.strip() and raw_data not in ('[]', '{}'):
                                data = json.loads(raw_data)
                                if isinstance(data, list):
                                    for alert in data:
                                        alert_id = alert.get('id', '') or str(alert.get('time', ''))
                                        if not alert_id:
                                            continue
                                        current_alert_ids.add(alert_id)

                                        if alert_id not in sent_alerts:
                                            cities = alert.get('data', [])
                                            if isinstance(cities, str):
                                                cities = [c.strip() for c in cities.split(',')]

                                            threat_code = alert.get('cat', 0)
                                            threat_info = THREAT_MAPPING.get(threat_code, {"type": "התרעה כללית", "color": "#ef4444"})

                                            area_points = []
                                            for raw_city in cities:
                                                raw_city = (raw_city or "").strip()
                                                if not raw_city:
                                                    continue
                                                a_lat, a_lng = resolve_city_coordinates(raw_city)
                                                area_points.append({"city": raw_city, "lat": a_lat, "lng": a_lng})

                                            polygon = get_area_polygon([p["city"] for p in area_points])

                                            if area_points:
                                                center_lat = sum(p["lat"] for p in area_points) / len(area_points)
                                                center_lng = sum(p["lng"] for p in area_points) / len(area_points)
                                                area_event = {
                                                    "id": f"oref_area_{alert_id}",
                                                    "alert_id": alert_id,
                                                    "text": f"⚠️ התראה אזורית\n{threat_info['type']} - " + ", ".join(p["city"] for p in area_points),
                                                    "lat": center_lat,
                                                    "lng": center_lng,
                                                    "type": threat_info['type'],
                                                    "color": threat_info['color'],
                                                    "reliability": 0.99,
                                                    "timestamp": alert.get('time', time.time()),
                                                    "source": "פיקוד העורף - אזורי",
                                                    "area": area_points,
                                                    "polygon": polygon,
                                                }
                                                active_alerts[alert_id] = area_event
                                                add_to_history(area_event)
                                                await manager.broadcast(area_event)

                                            for city in cities[:5]:
                                                city = (city or "").strip()
                                                if not city:
                                                    continue
                                                lat, lng = resolve_city_coordinates(city)
                                                event = {
                                                    "id": f"oref_{alert_id}_{city}",
                                                    "alert_id": alert_id,
                                                    "text": f"🚨 פיקוד העורף\n{threat_info['type']} - {city}\n{datetime.fromtimestamp(alert.get('time', time.time())).strftime('%H:%M:%S')}",
                                                    "lat": lat,
                                                    "lng": lng,
                                                    "type": threat_info['type'],
                                                    "color": threat_info['color'],
                                                    "reliability": 0.98,
                                                    "timestamp": alert.get('time', time.time()),
                                                    "source": "פיקוד העורף",
                                                    "polygon": get_area_polygon([city]),
                                                }
                                                add_to_history(event)
                                                await manager.broadcast(event)
                                            sent_alerts.add(alert_id)
                                            if len(sent_alerts) > 1000:
                                                sent_alerts = set(list(sent_alerts)[-500:])

                except urllib.error.HTTPError as e:
                    if e.code in (403, 429):
                        use_proxy = True
                    else:
                        print(f"HTTP error: {e.code}")
                except Exception as e:
                    print(f"שגיאה ישירה: {e}")
                    use_proxy = True

            # ───── מחיקת אזעקות שנסתיימו ─────
            ended_ids = prev_alert_ids - current_alert_ids
            for ended_id in ended_ids:
                if ended_id in active_alerts:
                    ended_event = active_alerts.pop(ended_id)
                    await manager.broadcast({
                        "action": "remove_alert",
                        "alert_id": ended_id,
                        "ids_to_remove": [
                            f"oref_area_{ended_id}",
                            *[f"oref_{ended_id}_{p['city']}" for p in ended_event.get("area", [])]
                        ]
                    })
            prev_alert_ids = current_alert_ids

            if use_proxy:
                try:
                    proxy_url = "https://api.tzevaadom.co.il/notifications"
                    headers = {'User-Agent': 'Mozilla/5.0 (compatible; WarRoomBot/1.0)'}
                    req = urllib.request.Request(proxy_url, headers=headers, method='GET')
                    with urllib.request.urlopen(req, timeout=5) as resp:
                        if resp.getcode() == 200:
                            data = json.loads(resp.read().decode('utf-8'))
                            for alert in data:
                                alert_id = alert.get('notificationId', str(time.time()))
                                if alert_id not in sent_alerts:
                                    cities = alert.get('cities', [])
                                    if not cities:
                                        continue
                                    threat_code = alert.get('category', 0)
                                    threat_info = THREAT_MAPPING.get(threat_code, {"type": "התרעה כללית", "color": "#ef4444"})
                                    area_points = []
                                    for raw_city in cities:
                                        raw_city = (raw_city or "").strip()
                                        if not raw_city:
                                            continue
                                        a_lat, a_lng = resolve_city_coordinates(raw_city)
                                        area_points.append({"city": raw_city, "lat": a_lat, "lng": a_lng})
                                    polygon = get_area_polygon([p["city"] for p in area_points])
                                    if area_points:
                                        center_lat = sum(p["lat"] for p in area_points) / len(area_points)
                                        center_lng = sum(p["lng"] for p in area_points) / len(area_points)
                                        area_event = {
                                            "id": f"tzevaadom_area_{alert_id}",
                                            "alert_id": alert_id,
                                            "text": f"⚠️ פיקוד העורף (פרוקסי)\n{threat_info['type']} - " + ", ".join(p["city"] for p in area_points),
                                            "lat": center_lat,
                                            "lng": center_lng,
                                            "type": threat_info['type'],
                                            "color": threat_info['color'],
                                            "reliability": 0.97,
                                            "timestamp": time.time(),
                                            "source": "פיקוד העורף - אזורי",
                                            "area": area_points,
                                            "polygon": polygon,
                                        }
                                        active_alerts[alert_id] = area_event
                                        add_to_history(area_event)
                                        await manager.broadcast(area_event)
                                    for city in cities[:5]:
                                        city = (city or "").strip()
                                        if not city:
                                            continue
                                        lat, lng = resolve_city_coordinates(city)
                                        event = {
                                            "id": f"tzevaadom_{alert_id}_{city}",
                                            "alert_id": alert_id,
                                            "text": f"🚨 פיקוד העורף (פרוקסי)\n{threat_info['type']} - {city}\n{datetime.now().strftime('%H:%M:%S')}",
                                            "lat": lat,
                                            "lng": lng,
                                            "type": threat_info['type'],
                                            "color": threat_info['color'],
                                            "reliability": 0.95,
                                            "timestamp": time.time(),
                                            "source": "פיקוד העורף (פרוקסי)",
                                            "polygon": get_area_polygon([city]),
                                        }
                                        add_to_history(event)
                                        await manager.broadcast(event)
                                    sent_alerts.add(alert_id)
                except Exception as e:
                    print(f"שגיאה בפרוקסי: {e}")

        except Exception as e:
            print(f"שגיאה כללית: {e}")

        await asyncio.sleep(1)

# ==========================================
# 🔁 OSINT Pipeline
# ==========================================
async def start_osint_pipeline():
    seen_urls: set = set()
    backoff_s = 15
    israel_target = ("ישראל (מרכז)", 32.0853, 34.7818)
    country_centroids = {
        "Israel": (31.0461, 34.8516), "Lebanon": (33.8547, 35.8623),
        "Syria": (34.8021, 38.9968), "Iran": (32.4279, 53.6880),
        "Iraq": (33.2232, 43.6793), "Yemen": (15.5527, 48.5164),
        "Jordan": (30.5852, 36.2384), "Egypt": (26.8206, 30.8025),
        "Turkey": (38.9637, 35.2433), "Russia": (61.5240, 105.3188),
        "United States": (37.0902, -95.7129), "United Kingdom": (55.3781, -3.4360),
        "France": (46.2276, 2.2137), "Germany": (51.1657, 10.4515),
        "Saudi Arabia": (23.8859, 45.0792), "Cyprus": (35.1264, 33.4299),
    }

    def gdelt_url(query, timespan="30m", maxrecords=25):
        params = {"query": query, "mode": "ArtList", "format": "json",
                  "maxrecords": str(maxrecords), "timespan": timespan, "sort": "datedesc"}
        return "https://api.gdeltproject.org/api/v2/doc/doc?" + urllib.parse.urlencode(params)

    def fetch_json(url):
        req = urllib.request.Request(url, headers={"User-Agent": "warroom/1.0"}, method="GET")
        with urllib.request.urlopen(req, timeout=20) as resp:
            return json.loads(resp.read().decode("utf-8", errors="replace"))

    def score_israel_alert(text):
        t = (text or "").lower()
        score = 0
        hits = []
        israel_tokens = ["israel", "tel aviv", "jerusalem", "galilee", "haifa", "beersheba", "israeli",
                         "גוש דן", "תל אביב", "ירושלים", "חיפה", "באר שבע", "ישראל"]
        if any(tok in t for tok in israel_tokens):
            score += 3
            hits.append("israel_mention")
        else:
            return (0, hits)
        strong = [
            ("air raid siren", 5), ("siren", 4), ("red alert", 6), ("rocket", 5),
            ("missile", 6), ("ballistic", 5), ("drone", 5), ("uav", 5),
            ("interception", 4), ("iron dome", 6), ("launch", 3), ("strike", 2),
            ("attack", 2), ("כטב", 6), ("טיל", 6), ("רקטה", 6), ("אזעקה", 7),
            ("צבע אדום", 8), ("יירוט", 6), ("כיפת ברזל", 7), ("שיגור", 7)
        ]
        for tok, pts in strong:
            if tok in t:
                score += pts
                hits.append(tok)
        weak_context = ["election", "minister", "diplomacy", "economy", "stock", "analysis", "opinion"]
        if any(tok in t for tok in weak_context):
            score -= 3
        return (max(score, 0), hits)

    def reliability_for_source(domain, source_country, score):
        base = 0.35
        if score >= 14: base = 0.75
        elif score >= 10: base = 0.6
        elif score >= 7: base = 0.5
        if source_country in {"Israel", "United States", "United Kingdom"}: base += 0.05
        if domain.endswith(".gov") or domain.endswith(".mil"): base += 0.1
        return float(max(0.1, min(base, 0.95)))

    q = '(rocket OR missile OR drone OR UAV OR "air raid" OR siren OR "red alert" OR interception OR "iron dome" OR launch OR "air defense") AND (Israel OR Israeli OR "Tel Aviv" OR Jerusalem OR Galilee OR Haifa)'

    while True:
        try:
            url = gdelt_url(q, timespan="30m", maxrecords=50)
            data = await asyncio.to_thread(fetch_json, url)
            articles = data.get("articles") or []
            backoff_s = 15
        except urllib.error.HTTPError as e:
            if getattr(e, "code", None) == 429:
                wait_s = min(backoff_s, 10 * 60)
                await asyncio.sleep(wait_s)
                backoff_s = min(max(backoff_s * 2, 30), 10 * 60)
                continue
            await asyncio.sleep(min(backoff_s, 120))
            backoff_s = min(max(backoff_s * 2, 30), 10 * 60)
            continue
        except Exception as e:
            print(f"GDELT error: {e}")
            await asyncio.sleep(min(backoff_s, 120))
            backoff_s = min(max(backoff_s * 2, 30), 10 * 60)
            continue

        new_items = []
        for a in articles:
            a_url = (a.get("url") or "").strip()
            if not a_url or a_url in seen_urls:
                continue
            seen_urls.add(a_url)
            new_items.append(a)

        for a in reversed(new_items):
            title = (a.get("title") or "").strip()
            domain = (a.get("domain") or "").strip()
            source_country = (a.get("sourcecountry") or "").strip()
            seen_date = (a.get("seendate") or "").strip()
            language = (a.get("language") or "").strip()
            o_lat, o_lng = country_centroids.get(source_country, (31.0461, 34.8516))
            _, t_lat, t_lng = israel_target
            intel_data = None
            if GEMINI_API_KEY and GEMINI_API_KEY != "YOUR_GEMINI_KEY":
                intel_data = await process_raw_intel(title, "GDELT")
            if intel_data and intel_data.get("lat") and intel_data.get("lng"):
                if is_within_israel(intel_data["lat"], intel_data["lng"]):
                    t_lat, t_lng = intel_data["lat"], intel_data["lng"]
            else:
                city_coords = extract_city_from_text(title)
                if city_coords:
                    t_lat, t_lng = city_coords
            text = f"{title}\n{domain} | {source_country} | {language} | {seen_date}\n{a_url}".strip()
            score, hits = score_israel_alert(title)
            if score < 10:
                continue
            event = {
                "id": str(time.time()),
                "text": text,
                "lat": t_lat,
                "lng": t_lng,
                "type": "אזעקה/שיגור (OSINT)",
                "reliability": reliability_for_source(domain, source_country, score),
                "timestamp": time.time(),
                "origin_lat": o_lat,
                "origin_lng": o_lng,
                "target_lat": t_lat,
                "target_lng": t_lng,
                "source": "OSINT",
            }
            add_to_history(event)
            await manager.broadcast(event)

        await asyncio.sleep(90)

# ==========================================
# 🗺️ ממשק המשתמש
# ==========================================
@app.get("/")
async def get():
    return HTMLResponse(f"""<!DOCTYPE html>
<html lang="he" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>WAR ROOM | מערכת ניטור מבצעית</title>
    <link rel="manifest" href="/static/manifest.json">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="WarRoom">
    <meta name="theme-color" content="#0ea5e9">
    <script src="https://unpkg.com/maplibre-gl@latest/dist/maplibre-gl.js"></script>
    <link href="https://unpkg.com/maplibre-gl@latest/dist/maplibre-gl.css" rel="stylesheet" />
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Assistant:wght@300;400;700&display=swap');
        * {{ box-sizing: border-box; }}
        body {{ font-family: 'Assistant', sans-serif; background-color: #020617; color: #f8fafc; overflow: hidden; margin: 0; padding: 0; }}
        #map {{ height: 100vh; width: 100%; }}

        /* ─── Sidebar ─── */
        .sidebar {{
            position: absolute;
            top: 0; right: 0;
            width: 100%; max-width: 380px;
            height: 100vh;
            background: rgba(15, 23, 42, 0.92);
            backdrop-filter: blur(12px);
            border-left: 1px solid #1e293b;
            overflow-y: auto;
            z-index: 10;
            box-shadow: -5px 0 25px rgba(0,0,0,0.5);
        }}
        @media (max-width: 640px) {{
            .sidebar {{ max-width: 100%; height: 42vh; bottom: 0; top: auto; border-left: none; border-top: 1px solid #1e293b; }}
            #map {{ height: 58vh; }}
        }}

        /* ─── Cards ─── */
        .event-card {{
            border-right: 4px solid #ef4444;
            background: #1e293b;
            margin: 8px 10px;
            padding: 12px 14px;
            border-radius: 6px;
            animation: slideIn 0.3s ease-out;
            transition: opacity 0.6s ease, transform 0.6s ease;
        }}
        .event-card.removing {{
            opacity: 0;
            transform: translateX(40px);
        }}
        @keyframes slideIn {{
            from {{ transform: translateX(60px); opacity: 0; }}
            to   {{ transform: translateX(0);   opacity: 1; }}
        }}
        .reliability-dot {{ height: 8px; width: 8px; border-radius: 50%; display: inline-block; }}

        /* ─── Map Style Switcher ─── */
        #map-style-switcher {{
            position: absolute;
            top: 10px; left: 10px;
            z-index: 20;
            background: rgba(15,23,42,0.92);
            border: 1px solid #334155;
            border-radius: 10px;
            padding: 6px 8px;
            display: flex;
            gap: 6px;
            align-items: center;
            backdrop-filter: blur(8px);
            box-shadow: 0 4px 20px rgba(0,0,0,0.4);
        }}
        #map-style-switcher button {{
            font-family: 'Assistant', sans-serif;
            font-size: 11px;
            padding: 4px 10px;
            border-radius: 6px;
            border: 1px solid #475569;
            background: transparent;
            color: #94a3b8;
            cursor: pointer;
            transition: all 0.2s;
            white-space: nowrap;
        }}
        #map-style-switcher button:hover {{
            background: #1e3a5f;
            color: #e2e8f0;
            border-color: #0ea5e9;
        }}
        #map-style-switcher button.active {{
            background: #0ea5e9;
            color: #fff;
            border-color: #0ea5e9;
            font-weight: 700;
        }}

        /* ─── History Button ─── */
        #history-btn {{
            position: absolute;
            bottom: 90px; left: 10px;
            z-index: 20;
            background: rgba(15,23,42,0.92);
            border: 1px solid #334155;
            border-radius: 10px;
            padding: 8px 12px;
            cursor: pointer;
            backdrop-filter: blur(8px);
            box-shadow: 0 4px 20px rgba(0,0,0,0.4);
            color: #cbd5e1;
            font-size: 12px;
            font-family: 'Assistant', sans-serif;
            display: flex;
            align-items: center;
            gap: 6px;
            transition: all 0.2s;
        }}
        #history-btn:hover {{ background: #1e293b; color: #f0f9ff; border-color: #0ea5e9; }}
        #history-btn .badge {{
            background: #0ea5e9;
            color: #fff;
            border-radius: 999px;
            padding: 1px 6px;
            font-size: 10px;
            font-weight: 700;
        }}

        /* ─── History Panel ─── */
        #history-panel {{
            position: absolute;
            bottom: 135px; left: 10px;
            z-index: 30;
            width: 320px;
            max-height: 400px;
            overflow-y: auto;
            background: rgba(10,18,35,0.97);
            border: 1px solid #334155;
            border-radius: 12px;
            backdrop-filter: blur(12px);
            box-shadow: 0 8px 32px rgba(0,0,0,0.6);
            display: none;
            padding: 10px;
        }}
        #history-panel h3 {{
            color: #0ea5e9;
            font-size: 13px;
            font-weight: 700;
            margin-bottom: 8px;
            padding-bottom: 6px;
            border-bottom: 1px solid #1e3a5f;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }}
        .hist-item {{
            padding: 7px 8px;
            border-radius: 6px;
            border-right: 3px solid #ef4444;
            background: #1e293b;
            margin-bottom: 5px;
            font-size: 11px;
            color: #cbd5e1;
        }}
        .hist-item .hist-time {{ color: #64748b; font-size: 10px; }}

        /* ─── Legend ─── */
        #legend {{
            position: fixed;
            left: 10px; bottom: 10px;
            z-index: 20;
            background: rgba(10,18,35,0.92);
            border: 1px solid #334155;
            border-radius: 10px;
            padding: 8px 12px;
            font-size: 11px;
            color: #cbd5e1;
            backdrop-filter: blur(8px);
            box-shadow: 0 4px 16px rgba(0,0,0,0.4);
        }}
        #legend .leg-title {{ font-weight: 700; color: #e2e8f0; margin-bottom: 4px; font-size: 12px; }}
        #legend .leg-row {{ display: flex; align-items: center; gap: 6px; margin: 2px 0; }}
        #legend .leg-dot {{ width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }}

        /* ─── Pulse marker ─── */
        .pulse-marker {{
            width: 16px; height: 16px;
            border-radius: 50%;
            animation: pulse 1.5s ease-out infinite;
        }}
        @keyframes pulse {{
            0%   {{ box-shadow: 0 0 0 0 rgba(239,68,68,0.6); }}
            70%  {{ box-shadow: 0 0 0 12px rgba(239,68,68,0); }}
            100% {{ box-shadow: 0 0 0 0 rgba(239,68,68,0); }}
        }}
    </style>
</head>
<body>

<!-- ═══════════════════════════════════════ SIDEBAR ═══════════════════════════════════════ -->
<div class="sidebar" id="sidebar">
    <div class="p-4 border-b border-slate-700 sticky top-0 bg-slate-900/95 backdrop-blur z-20">
        <div class="flex items-center justify-between">
            <h1 class="text-xl font-bold text-cyan-400 uppercase tracking-widest">War Room Live</h1>
            <div id="live-clock" class="text-lg font-mono text-cyan-300 bg-slate-800/50 px-3 py-1 rounded-lg border border-slate-600">00:00:00</div>
        </div>
        <p class="text-xs text-slate-400 mt-1 italic">מערכת ניטור OSINT מבוססת AI</p>
        <div id="active-count" class="mt-2 text-xs text-green-400 flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-green-400 animate-pulse inline-block"></span>
            <span>מחובר ופעיל</span>
        </div>
    </div>
    <div id="event-list" class="pb-20"></div>
</div>

<!-- ═══════════════════════════════════════ MAP ═══════════════════════════════════════ -->
<div id="map"></div>

<!-- ─── Map Style Switcher ─── -->
<div id="map-style-switcher">
    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#64748b" stroke-width="2">
        <polygon points="1 6 1 22 8 18 16 22 23 18 23 2 16 6 8 2 1 6"/>
        <line x1="8" y1="2" x2="8" y2="18"/><line x1="16" y1="6" x2="16" y2="22"/>
    </svg>
    <button class="active" data-style="streets">כבישים</button>
    <button data-style="dark">כהה</button>
    <button data-style="satellite">לוויין</button>
    <button data-style="topo">טופוגרפי</button>
    <button data-style="outdoor">שטח</button>
</div>

<!-- ─── History Button ─── -->
<button id="history-btn" onclick="toggleHistory()">
    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/>
    </svg>
    אזעקות 24 שעות
    <span class="badge" id="history-count">0</span>
</button>

<!-- ─── History Panel ─── -->
<div id="history-panel">
    <h3>
        <span>📋 היסטוריית 24 שעות</span>
        <button onclick="toggleHistory()" style="color:#64748b;font-size:16px;background:none;border:none;cursor:pointer;">✕</button>
    </h3>
    <div id="history-list"></div>
</div>

<!-- ─── Legend ─── -->
<div id="legend">
    <div class="leg-title">מפת איומים</div>
    <div class="leg-row"><div class="leg-dot" style="background:#ef4444"></div><span>צבע אדום</span></div>
    <div class="leg-row"><div class="leg-dot" style="background:#f97316"></div><span>חדירת כלי טיס</span></div>
    <div class="leg-row"><div class="leg-dot" style="background:#a855f7"></div><span>חומרים מסוכנים</span></div>
    <div class="leg-row"><div class="leg-dot" style="background:#0ea5e9"></div><span>צונאמי</span></div>
    <div class="leg-row"><div class="leg-dot" style="background:#eab308"></div><span>רעידת אדמה/מחבלים</span></div>
    <div class="leg-row"><div class="leg-dot" style="background:#facc15"></div><span>ירי בל"ק</span></div>
</div>

<script>
// ─────────────────────────────────────────────────────────────────────────────
// קונפיגורציה
// ─────────────────────────────────────────────────────────────────────────────
const MAPTILER_KEY = "{MAPTILER_KEY}";
const STYLES = {{
    streets:   `https://api.maptiler.com/maps/streets-v4/style.json?key=${{MAPTILER_KEY}}`,
    dark:      `https://api.maptiler.com/maps/darkmatter/style.json?key=${{MAPTILER_KEY}}`,
    satellite: `https://api.maptiler.com/maps/satellite/style.json?key=${{MAPTILER_KEY}}`,
    topo:      `https://api.maptiler.com/maps/topo-v2/style.json?key=${{MAPTILER_KEY}}`,
    outdoor:   `https://api.maptiler.com/maps/outdoor-v2/style.json?key=${{MAPTILER_KEY}}`,
}};

const THREAT_COLORS = {{
    "צבע אדום":                  "#ef4444",
    "חדירת כלי טיס עוין":        "#f97316",
    "אירוע חומרים מסוכנים":       "#a855f7",
    "חשש לצונאמי":                "#0ea5e9",
    "צונאמי":                    "#0ea5e9",
    "רעידת אדמה":                "#eab308",
    "חשש לחדירת מחבלים":          "#eab308",
    "ירי בלתי קונבנציונלי":       "#facc15",
    "אזעקה/שיגור (OSINT)":        "#38bdf8",
}};

function threatColor(type, fallback) {{
    if (fallback) return fallback;
    const t = (type || "").trim();
    if (THREAT_COLORS[t]) return THREAT_COLORS[t];
    if (t.includes("כטב") || t.includes("כלי טיס")) return "#f97316";
    if (t.includes("טיל") || t.includes("רקטה")) return "#ef4444";
    if (t.includes("חומרים מסוכנים")) return "#a855f7";
    if (t.includes("צונאמי")) return "#0ea5e9";
    if (t.includes("רעידת אדמה")) return "#eab308";
    return "#ef4444";
}}

// ─────────────────────────────────────────────────────────────────────────────
// מפה
// ─────────────────────────────────────────────────────────────────────────────
let currentStyleId = "streets";
const map = new maplibregl.Map({{
    container: 'map',
    style: STYLES.streets,
    center: [34.8516, 31.0461],
    zoom: 7,
    antialias: true
}});

// Style switcher
document.querySelectorAll('#map-style-switcher button').forEach(btn => {{
    btn.addEventListener('click', () => {{
        const styleId = btn.dataset.style;
        if (styleId === currentStyleId) return;
        currentStyleId = styleId;
        document.querySelectorAll('#map-style-switcher button').forEach(b => b.classList.remove('active'));
        btn.classList.add('active');
        const center = map.getCenter();
        const zoom   = map.getZoom();
        map.setStyle(STYLES[styleId]);
        map.once('styledata', () => {{
            map.setCenter(center);
            map.setZoom(zoom);
            restoreMapLayers();
        }});
    }});
}});

// שמירת פולינים/קווים לאחר שינוי סגנון
const persistedSources = {{}};
const persistedLayers  = [];
const lineFeatures     = [];

function restoreMapLayers() {{
    // שחזור מקורות
    for (const [id, data] of Object.entries(persistedSources)) {{
        if (!map.getSource(id)) {{
            map.addSource(id, {{ type: 'geojson', data }});
        }}
    }}
    // שחזור שכבות
    for (const layerSpec of persistedLayers) {{
        if (!map.getLayer(layerSpec.id)) {{
            try {{ map.addLayer(layerSpec); }} catch(e) {{}}
        }}
    }}
    // שחזור קווי OSINT
    initLinesSource();
}}

function initLinesSource() {{
    if (!map.getSource('alert-lines')) {{
        map.addSource('alert-lines', {{ type: 'geojson', data: {{ type: 'FeatureCollection', features: lineFeatures }} }});
        map.addLayer({{ id:'alert-lines-glow', type:'line', source:'alert-lines', paint:{{'line-color':['coalesce',['get','color'],'#ef4444'],'line-opacity':0.25,'line-width':8}} }});
        map.addLayer({{ id:'alert-lines-layer', type:'line', source:'alert-lines', layout:{{'line-cap':'round','line-join':'round'}}, paint:{{'line-color':['coalesce',['get','color'],'#ef4444'],'line-opacity':0.9,'line-width':2.5,'line-dasharray':[2,2]}} }});
    }}
}}

map.on('load', () => {{
    map.addControl(new maplibregl.NavigationControl(), 'top-right');
    initLinesSource();
}});

// ─────────────────────────────────────────────────────────────────────────────
// ניהול מרקרים ופוליגונים
// ─────────────────────────────────────────────────────────────────────────────
const markerMap   = {{}};  // event_id -> Marker
const polygonIds  = {{}};  // event_id -> [sourceId, layerIds...]

function addPolygonToMap(eventId, polygon, color) {{
    if (!polygon || polygon.length < 3) return;
    const sourceId = `poly-src-${{eventId}}`;
    const fillId   = `poly-fill-${{eventId}}`;
    const borderId = `poly-border-${{eventId}}`;

    const coords = polygon.filter(c => Array.isArray(c) && c.length === 2 && isFinite(c[0]) && isFinite(c[1]));
    if (coords.length < 3) return;
    const closed = [...coords];
    if (closed[0][0] !== closed[closed.length-1][0] || closed[0][1] !== closed[closed.length-1][1]) {{
        closed.push(closed[0]);
    }}

    const geojsonData = {{
        type: "FeatureCollection",
        features: [{{ type: "Feature", properties: {{}}, geometry: {{ type: "Polygon", coordinates: [closed] }} }}]
    }};

    persistedSources[sourceId] = geojsonData;

    if (!map.getSource(sourceId)) {{
        map.addSource(sourceId, {{ type: 'geojson', data: geojsonData }});
    }}
    if (!map.getLayer(fillId)) {{
        const fillSpec = {{ id: fillId, type: 'fill', source: sourceId, paint: {{ 'fill-color': color, 'fill-opacity': 0.18 }} }};
        map.addLayer(fillSpec);
        persistedLayers.push(fillSpec);
    }}
    if (!map.getLayer(borderId)) {{
        const borderSpec = {{ id: borderId, type: 'line', source: sourceId, paint: {{ 'line-color': color, 'line-width': 2.5, 'line-opacity': 0.95 }} }};
        map.addLayer(borderSpec);
        persistedLayers.push(borderSpec);
    }}

    polygonIds[eventId] = [sourceId, fillId, borderId];
}}

function removePolygonFromMap(eventId) {{
    const ids = polygonIds[eventId];
    if (!ids) return;
    const [sourceId, fillId, borderId] = ids;
    try {{ if (map.getLayer(fillId))   map.removeLayer(fillId);   }} catch(e) {{}}
    try {{ if (map.getLayer(borderId)) map.removeLayer(borderId); }} catch(e) {{}}
    try {{ if (map.getSource(sourceId)) map.removeSource(sourceId); }} catch(e) {{}}
    delete polygonIds[eventId];
    delete persistedSources[sourceId];
    const lIdx = persistedLayers.findIndex(l => l.id === fillId || l.id === borderId);
    if (lIdx > -1) persistedLayers.splice(lIdx, 2);
}}

function removeMarkerFromMap(eventId) {{
    const marker = markerMap[eventId];
    if (marker) {{
        marker.remove();
        delete markerMap[eventId];
    }}
}}

// ─────────────────────────────────────────────────────────────────────────────
// WebSocket
// ─────────────────────────────────────────────────────────────────────────────
const eventList   = document.getElementById('event-list');
const historyList = document.getElementById('history-list');
const histCount   = document.getElementById('history-count');
const histPanel   = document.getElementById('history-panel');
let   historyData = [];

const ws = new WebSocket("ws://" + window.location.host + "/ws");

ws.onmessage = (event) => {{
    const data = JSON.parse(event.data);

    // מחיקת אזעקות שנסתיימו
    if (data.action === "remove_alert") {{
        (data.ids_to_remove || []).forEach(id => {{
            removeMarkerFromMap(id);
            removePolygonFromMap(id);
            // גם ניסיון מחיקה לפי alert_id ישיר
            removePolygonFromMap(data.alert_id);
            removeMarkerFromMap(data.alert_id);
            const card = document.getElementById(`card-${{id}}`);
            if (card) {{
                card.classList.add('removing');
                setTimeout(() => card.remove(), 650);
            }}
        }});
        return;
    }}

    // היסטוריה
    if (data.is_history) {{
        addToHistory(data);
        return;
    }}

    addToHistory(data);
    addEventToUI(data);
}};

function addToHistory(data) {{
    // מנע כפילויות
    if (historyData.find(h => h.id === data.id)) return;
    historyData.push(data);

    const color = threatColor(data.type, data.color);
    const item = document.createElement('div');
    item.className = 'hist-item';
    item.style.borderRightColor = color;
    const ts = data.timestamp ? new Date(data.timestamp * 1000).toLocaleTimeString('he-IL') : '';
    item.innerHTML = `<div class="hist-time">${{ts}} — ${{data.type || 'איום'}}</div><div>${{(data.text || '').split('\\n')[0]}}</div>`;
    historyList.prepend(item);
    histCount.textContent = historyData.length;
}}

function toggleHistory() {{
    histPanel.style.display = histPanel.style.display === 'none' ? 'block' : 'none';
}}

function addEventToUI(data) {{
    const color = threatColor(data.type, data.color);

    // ─── Sidebar card ───
    const card = document.createElement('div');
    card.className = 'event-card';
    card.id = `card-${{data.id}}`;
    card.style.borderRightColor = color;
    card.innerHTML = `
        <div class="flex justify-between items-start mb-2">
            <span class="text-xs text-slate-400">${{new Date().toLocaleTimeString('he-IL')}}</span>
            <span class="text-xs px-2 py-0.5 rounded border" style="background:rgba(15,23,42,0.9);color:${{color}};border-color:${{color}}">${{data.type || 'איום'}}</span>
        </div>
        <p class="text-sm text-slate-100">${{(data.text || '').replace(/\\n/g,'<br>')}}</p>
        <div class="mt-2 flex items-center gap-2">
            <span class="text-[10px] text-slate-500 uppercase">אמינות:</span>
            <div class="flex-1 h-1 bg-slate-800 rounded">
                <div class="h-1 rounded" style="width:${{(data.reliability||0)*100}}%;background:${{color}}"></div>
            </div>
        </div>
    `;
    eventList.prepend(card);

    // ─── Marker ───
    const el = document.createElement('div');
    el.className = 'pulse-marker';
    el.style.background = color;
    el.style.boxShadow = `0 0 0 0 ${{color}}88`;
    const marker = new maplibregl.Marker(el)
        .setLngLat([data.lng, data.lat])
        .setPopup(new maplibregl.Popup({{offset:25}}).setHTML(
            `<div style="font-family:'Assistant',sans-serif;padding:4px 8px;max-width:240px">
                <strong style="color:${{color}}">${{data.type}}</strong>
                <p style="margin:4px 0;font-size:12px">${{(data.text||'').replace(/\\n/g,'<br>')}}</p>
             </div>`
        ))
        .addTo(map);
    markerMap[data.id] = marker;

    // ─── Polygon — רק עבור התראות פיקוד העורף (לא OSINT) ───
    const isOrefAlert = data.source && (data.source.includes("פיקוד העורף") || data.source.includes("oref"));
    if (isOrefAlert && data.polygon) {{
        addPolygonToMap(data.id, data.polygon, color);
    }}

    // ─── OSINT trajectory line ───
    if (data.source === "OSINT" && data.origin_lat != null && data.target_lat != null) {{
        lineFeatures.unshift({{
            type: "Feature",
            properties: {{ color }},
            geometry: {{ type: "LineString", coordinates: [[data.origin_lng, data.origin_lat],[data.target_lng, data.target_lat]] }}
        }});
        if (lineFeatures.length > 50) lineFeatures.length = 50;
        const src = map.getSource('alert-lines');
        if (src) src.setData({{ type: "FeatureCollection", features: lineFeatures }});
    }}

    map.flyTo({{ center: [data.lng, data.lat], zoom: 9, speed: 0.8 }});
}}

// ─── שעון ───
function updateClock() {{
    const now = new Date();
    document.getElementById('live-clock').textContent =
        [now.getHours(), now.getMinutes(), now.getSeconds()]
        .map(n => String(n).padStart(2,'0')).join(':');
}}
updateClock();
setInterval(updateClock, 1000);

// ─── SW ───
if ('serviceWorker' in navigator) {{
    navigator.serviceWorker.register('/static/sw.js').catch(() => {{}});
}}
</script>
</body>
</html>""")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
