# spotirec
A Python command-line tool for Linux that can record live audio, specifically with information retrieval from Spotify – INTENDED FOR PERSONAL USE ONLY.

####################
# SpotiRec – Setup #
####################

## 1. System-Pakete

sudo apt install ffmpeg pulseaudio-utils python3-gi python3-pip python3-venv


#2. Projektordner

mkdir -p ~/spotirec && cd ~/spotirec
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
pip install pydbus pyyaml requests mutagen


#3. Virtuelles Audio-Sink (nach jedem Reboot!)


pactl load-module module-null-sink sink_name=spotirec sink_properties=device.description=SpotiRec

#Dann pavucontrol öffnen → Reiter Wiedergabe → Spotify auf SpotiRec umleiten.


#4. Credentials
Spotify Developer Account Login und API-Key erstellen

mkdir -p ~/.config/spotirec
nano ~/.config/spotirec/credentials.yaml

Inhalt:

client_id: "DEINE_CLIENT_ID"
client_secret: "DEIN_CLIENT_SECRET"


#5. Config


nano ~/spotirec/config.yaml

#Inhalt:

output_dir: "~/spotirec/recordings"
mp3_bitrate: "320k"
sink_name: "spotirec"


#6. Code-Files


~/spotirec/spotify_meta.py ablegen

#Inhalt

import os, base64, requests
from dotenv import load_dotenv

load_dotenv()

CLIENT_ID = os.getenv("SPOTIFY_CLIENT_ID")
CLIENT_SECRET = os.getenv("SPOTIFY_CLIENT_SECRET")

def get_token():
    creds = base64.b64encode(f"{CLIENT_ID}:{CLIENT_SECRET}".encode()).decode()
    r = requests.post("https://accounts.spotify.com/api/token",
        headers={"Authorization": f"Basic {creds}"},
        data={"grant_type": "client_credentials"})
    return r.json()["access_token"]

def get_track_info(track_id):
    token = get_token()
    r = requests.get(f"https://api.spotify.com/v1/tracks/{track_id}",
        headers={"Authorization": f"Bearer {token}"})
    return r.json()


~/spotirec/watcher.py

#Inahlt
#!/usr/bin/env python3
"""Spotify Recorder – MPRIS Watcher + Capture + Tagging."""

import argparse
import os
import re
import signal
import subprocess
import sys
import threading
import queue
from pathlib import Path
from datetime import datetime

import yaml
import requests
from gi.repository import GLib
from pydbus import SessionBus

from spotify_meta import get_track_info

# ---------- CLI ----------
def parse_args():
    p = argparse.ArgumentParser(description="Spotify Recorder")
    g1 = p.add_mutually_exclusive_group(required=True)
    g1.add_argument("-a", "--album", action="store_true")
    g1.add_argument("-p", "--playlist", action="store_true")
    g2 = p.add_mutually_exclusive_group(required=True)
    g2.add_argument("--mp3", action="store_true")
    g2.add_argument("--flac", action="store_true")
    return p.parse_args()

ARGS = parse_args()

# ---------- Config ----------
CONFIG_PATH = Path(__file__).parent / "config.yaml"
with open(CONFIG_PATH) as f:
    CFG = yaml.safe_load(f)

MODE = "playlist" if ARGS.playlist else "album"
FORMAT = "mp3" if ARGS.mp3 else "flac"
MP3_BITRATE = CFG.get("mp3_bitrate", "320k")
SINK_NAME = CFG["sink_name"]

if MODE == "album":
    OUTPUT_DIR = Path(os.path.expanduser(CFG["output_dir"]))
else:
    OUTPUT_DIR = Path.cwd()
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

TMP_BASE = Path(os.path.expanduser(CFG["output_dir"]))
TMP_DIR = TMP_BASE / ".tmp"
TMP_DIR.mkdir(parents=True, exist_ok=True)

# ---------- Routing ----------
def route_spotify_to_sink():
    """Stellt sicher, dass Spotifys sink-input auf SINK_NAME läuft."""
    try:
        out = subprocess.run(["pactl", "list", "short", "sink-inputs"],
                             capture_output=True, text=True).stdout
        # Spotify-Input via Properties identifizieren
        full = subprocess.run(["pactl", "list", "sink-inputs"],
                              capture_output=True, text=True).stdout
        # Blöcke splitten
        blocks = re.split(r"\nSink Input #", "\n" + full)
        moved = False
        for b in blocks:
            if not b.strip():
                continue
            m = re.match(r"(\d+)", b)
            if not m:
                continue
            sid = m.group(1)
            if "application.name = \"spotify\"" in b.lower() or \
               "com.spotify.client" in b.lower():
                subprocess.run(["pactl", "move-sink-input", sid, SINK_NAME],
                               capture_output=True)
                print(f"[ROUTE] sink-input {sid} → {SINK_NAME}")
                moved = True
        if not moved:
            print("[ROUTE] kein Spotify-Stream gefunden (noch nicht aktiv?)")
    except Exception as e:
        print(f"[ROUTE] Fehler: {e}")

# ---------- Helpers ----------
SANITIZE_RE = re.compile(r'[/\\:\*\?"<>\|]')

def sanitize(s: str) -> str:
    return SANITIZE_RE.sub("_", s).strip()

def extract_year(date_str: str) -> str:
    if not date_str:
        return "0000"
    m = re.match(r"(\d{4})", date_str)
    return m.group(1) if m else "0000"

def enrich_from_spotify(meta: dict) -> dict:
    sid = meta.get("spotify_id")
    if not sid:
        return meta
    try:
        info = get_track_info(sid)
        if "error" in info:
            print(f"[META] API-Fehler: {info['error']}")
            return meta
        album = info.get("album", {})
        meta["year"] = extract_year(album.get("release_date", ""))
        meta["disc_number"] = info.get("disc_number", meta.get("disc_number", 1))
        meta["track_number"] = info.get("track_number", meta.get("track_number", 0))
        meta["isrc"] = info.get("external_ids", {}).get("isrc", "")
        meta["album"] = album.get("name", meta["album"])
        meta["album_artist"] = ", ".join(a["name"] for a in album.get("artists", []))
        images = album.get("images", [])
        meta["cover_url"] = images[0]["url"] if images else None
    except Exception as e:
        print(f"[META] enrich Fehler: {e}")
    return meta

def fetch_cover(url: str) -> bytes | None:
    if not url:
        return None
    try:
        r = requests.get(url, timeout=10)
        if r.status_code == 200:
            return r.content
    except Exception as e:
        print(f"[META] Cover-Download Fehler: {e}")
    return None

def write_tags(path: Path, meta: dict, cover: bytes | None):
    try:
        if path.suffix.lower() == ".flac":
            from mutagen.flac import FLAC, Picture
            audio = FLAC(str(path))
            audio["title"] = meta["title"]
            audio["artist"] = meta["artist"]
            audio["album"] = meta["album"]
            audio["albumartist"] = meta.get("album_artist", meta["artist"])
            audio["date"] = meta["year"]
            audio["tracknumber"] = str(meta.get("track_number", 0))
            audio["discnumber"] = str(meta.get("disc_number", 1))
            if meta.get("isrc"):
                audio["isrc"] = meta["isrc"]
            if cover:
                pic = Picture()
                pic.type = 3
                pic.mime = "image/jpeg"
                pic.desc = "Cover"
                pic.data = cover
                audio.clear_pictures()
                audio.add_picture(pic)
            audio.save()
        else:
            from mutagen.id3 import ID3, TIT2, TPE1, TPE2, TALB, TDRC, TRCK, TPOS, TSRC, APIC, ID3NoHeaderError
            try:
                audio = ID3(str(path))
            except ID3NoHeaderError:
                audio = ID3()
            audio["TIT2"] = TIT2(encoding=3, text=meta["title"])
            audio["TPE1"] = TPE1(encoding=3, text=meta["artist"])
            audio["TPE2"] = TPE2(encoding=3, text=meta.get("album_artist", meta["artist"]))
            audio["TALB"] = TALB(encoding=3, text=meta["album"])
            audio["TDRC"] = TDRC(encoding=3, text=meta["year"])
            audio["TRCK"] = TRCK(encoding=3, text=str(meta.get("track_number", 0)))
            audio["TPOS"] = TPOS(encoding=3, text=str(meta.get("disc_number", 1)))
            if meta.get("isrc"):
                audio["TSRC"] = TSRC(encoding=3, text=meta["isrc"])
            if cover:
                audio["APIC"] = APIC(encoding=3, mime="image/jpeg",
                                     type=3, desc="Cover", data=cover)
            audio.save(str(path))
        print(f"[TAG] ✓ {path.name}")
    except Exception as e:
        print(f"[TAG] Fehler: {e}")

def build_output_path(meta: dict) -> Path:
    ext = FORMAT
    if MODE == "playlist":
        fname = sanitize(f"{meta['artist']} - {meta['title']}")
        return OUTPUT_DIR / f"{fname}.{ext}"
    else:
        album_dir = OUTPUT_DIR / sanitize(meta.get("album_artist", meta["artist"])) / \
                    sanitize(f"{meta['year']} {meta['album']}")
        track_no = f"{meta.get('track_number', 0):02d}"
        fname = sanitize(f"{track_no} - {meta['artist']} - {meta['title']}")
        return album_dir / f"{fname}.{ext}"

# ---------- Convert Worker ----------
convert_q: queue.Queue = queue.Queue()

def convert_worker():
    while True:
        item = convert_q.get()
        if item is None:
            break
        wav_path, meta = item
        try:
            meta = enrich_from_spotify(meta)
            cover = fetch_cover(meta.get("cover_url"))

            final_path = build_output_path(meta)
            final_path.parent.mkdir(parents=True, exist_ok=True)

            if FORMAT == "flac":
                cmd = ["ffmpeg", "-y", "-i", str(wav_path),
                       "-c:a", "flac", "-compression_level", "5",
                       str(final_path)]
            else:
                cmd = ["ffmpeg", "-y", "-i", str(wav_path),
                       "-c:a", "libmp3lame", "-b:a", MP3_BITRATE,
                       str(final_path)]
            r = subprocess.run(cmd, capture_output=True)
            if r.returncode == 0 and final_path.exists():
                wav_path.unlink()
                write_tags(final_path, meta, cover)
                print(f"[CONVERT] ✓ {final_path}")
            else:
                print(f"[CONVERT] ✗ {wav_path.name} – WAV behalten")
                print(r.stderr.decode()[-300:])
        except Exception as e:
            print(f"[CONVERT] Fehler: {e}")
        finally:
            convert_q.task_done()

threading.Thread(target=convert_worker, daemon=True).start()

# ---------- Recorder ----------
class Recorder:
    def __init__(self):
        self.proc: subprocess.Popen | None = None
        self.current_wav: Path | None = None
        self.current_meta: dict | None = None

    def start(self, meta: dict):
        # Routing nochmal sicherstellen
        route_spotify_to_sink()

        ts = datetime.now().strftime("%Y%m%d_%H%M%S")
        self.current_wav = TMP_DIR / f"capture_{ts}.wav"
        self.current_meta = meta

        cmd = ["parec", "--device", SINK_NAME + ".monitor",
               "--rate=44100", "--channels=2", "--format=s16le",
               "--file-format=wav", str(self.current_wav)]
        self.proc = subprocess.Popen(cmd, stdout=subprocess.DEVNULL,
                                     stderr=subprocess.DEVNULL)
        print(f"[REC] ▶ {meta['artist']} – {meta['title']}")

    def stop(self, discard=False):
        if not self.proc:
            return
        self.proc.send_signal(signal.SIGINT)
        try:
            self.proc.wait(timeout=3)
        except subprocess.TimeoutExpired:
            self.proc.kill()
        self.proc = None

        if discard or not self.current_wav or not self.current_wav.exists():
            if self.current_wav and self.current_wav.exists():
                self.current_wav.unlink()
            print("[REC] ⏹ verworfen")
            self.current_wav = None
            self.current_meta = None
            return

        print(f"[REC] ⏹ → queue: {self.current_meta['title']}")
        convert_q.put((self.current_wav, self.current_meta))
        self.current_wav = None
        self.current_meta = None

recorder = Recorder()

# ---------- MPRIS Watcher ----------
bus = SessionBus()
last_track_id = None
first_track_seen = False

def get_meta(props):
    md = props.get("Metadata", {})
    trackid_raw = str(md.get("mpris:trackid", ""))
    m = re.search(r"track[:/]([A-Za-z0-9]+)$", trackid_raw)
    spotify_id = m.group(1) if m else None
    return {
        "track_id": trackid_raw,
        "spotify_id": spotify_id,
        "title": str(md.get("xesam:title", "Unknown")),
        "artist": ", ".join(md.get("xesam:artist", ["Unknown"])),
        "album": str(md.get("xesam:album", "Unknown")),
        "track_number": int(md.get("xesam:trackNumber", 0)),
        "disc_number": int(md.get("xesam:discNumber", 1)),
        "year": extract_year(str(md.get("xesam:contentCreated", ""))),
        "length_us": int(md.get("mpris:length", 0)),
    }

def on_properties_changed(iface, changed, invalidated):
    global last_track_id, first_track_seen
    if "Metadata" not in changed and "PlaybackStatus" not in changed:
        return

    spotify = bus.get("org.mpris.MediaPlayer2.spotify",
                      "/org/mpris/MediaPlayer2")
    props = {"Metadata": spotify.Metadata,
             "PlaybackStatus": spotify.PlaybackStatus}
    meta = get_meta(props)
    status = props["PlaybackStatus"]

    if not meta["track_id"]:
        return

    if meta["track_id"] != last_track_id:
        if last_track_id is not None:
            recorder.stop(discard=not first_track_seen)
        else:
            print(f"[INFO] Initial-Track erkannt, wird verworfen: "
                  f"{meta['artist']} – {meta['title']}")

        last_track_id = meta["track_id"]

        if status == "Playing":
            if first_track_seen:
                recorder.start(meta)
            else:
                first_track_seen = True

def main():
    # Beim Start einmal routen
    route_spotify_to_sink()

    spotify = bus.get("org.mpris.MediaPlayer2.spotify",
                      "/org/mpris/MediaPlayer2")
    spotify.PropertiesChanged.connect(on_properties_changed)
    print(f"[WATCHER] aktiv – Modus: {MODE}, Format: {FORMAT}")
    print(f"[WATCHER] Output: {OUTPUT_DIR}")
    print("[WATCHER] erster laufender Track wird verworfen, Aufnahme ab Track 2")
    loop = GLib.MainLoop()
    try:
        loop.run()
    except KeyboardInterrupt:
        print("\n[WATCHER] beende…")
        recorder.stop(discard=True)
        convert_q.put(None)

if __name__ == "__main__":
    main()


#7. Ordnerstruktur (final)


~/spotirec/
├── .venv/
├── config.yaml
├── spotify_meta.py
├── watcher.py
└── recordings/          (wird automatisch erstellt)

~/.config/spotirec/
└── credentials.yaml


#8. Starten

Spotify starten -> Track starten

pactl load-module module-null-sink sink_name=spotirec 
sink_properties=device.description=SpotiRec
source .venv/bin/activate
cd ~/spotirec oder das gewünschte Verzeichnis für den Playlist Mode
python watcher.py
spotirec -p --mp3 #-p Playlist Mode -a Album Mode --mp3 --flac

Spotify abspielen, erster Track wird verworfen, ab Track 2 läuft Aufnahme.
Beenden mit Ctrl+C.
