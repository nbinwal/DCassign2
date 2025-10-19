
# Distributed Chat Room with Distributed Mutual Exclusion (DME)

This project implements a 3-node distributed "chat room" application for the Distributed Computing (CCZG 526) Assignment II.  
It uses a **Distributed Mutual Exclusion (DME) Protocol** to ensure only one user can write (`post`) to the shared file resource.

## 1. System Architecture and Configuration

The system is split across three cloud lab nodes, as seen in the lab environment:

| Node Name | Role | Code Modules | Protocol | Port Usage |
| :--- | :--- | :--- | :--- | :--- |
| **Master** | File Server (Resource Manager) | `chat_server.py` | Handles File I/O (`view`/`post` APIs) | TCP **8000** |
| **Node-1 (Client)** | User Application | `client.py`, `dme_middleware.py` | Runs **Ricart-Agrawala DME** (Middleware) | TCP **8001** |
| **Node-2 (Client)** | User Application | `client.py`, `dme_middleware.py` | Runs **Ricart-Agrawala DME** (Middleware) | TCP **8001** |

### IP Configuration (REQUIRED ACTION)

**You MUST replace the following placeholder IPs with the actual private IPs of your lab machines in the code files (`client.py` and `dme_middleware.py`).**

| Node Role | Placeholder IP | Used In Code |
| :--- | :--- | :--- |
| Master (Server) | `10.0.0.1` | `client.py` (as `SERVER_IP`) |
| Node-1 (Client 1) | `10.0.0.2` | `dme_middleware.py` (as `PEERS`) |
| Node-2 (Client 2) | `10.0.0.3` | `dme_middleware.py` (as `PEERS`) |

-----

## 2. Code Modules

The DME algorithm is kept in a separate module from the application code.

### A. Distributed Application Code

#### `chat_server.py` (Master Node)

```python
# chat_server.py (Run on Master Node: e.g., 10.0.0.1)
import socket
import threading
import os

SERVER_HOST = '0.0.0.0'
SERVER_PORT = 8000
LOG_FILE = 'chat_log.txt'

def initialize_log():
    if not os.path.exists(LOG_FILE):
        with open(LOG_FILE, 'w') as f:
            f.write("--- Chat Room Initialized ---\n")

def handle_client(conn, addr):
    try:
        data = conn.recv(1024).decode()
        parts = data.split(' ', 1)
        command = parts[0]
            
        if command == 'VIEW':
            with open(LOG_FILE, 'r') as f:
                content = f.read()
            conn.sendall(content.encode())

        elif command == 'POST':
            text_to_append = parts[1]
            with open(LOG_FILE, 'a') as f:
                f.write(text_to_append + '\n')
            conn.sendall("POST_SUCCESS".encode())
            
    except Exception as e:
        conn.sendall(f"SERVER_ERROR: {e}".encode())
    finally:
        conn.close()

def start_server():
    initialize_log()
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.bind((SERVER_HOST, SERVER_PORT))
    server_socket.listen(5)
    print(f"Chat Server listening on {SERVER_HOST}:{SERVER_PORT}")

    while True:
        conn, addr = server_socket.accept()
        threading.Thread(target=handle_client, args=(conn, addr)).start()

if __name__ == "__main__":
    start_server()
```

#### `client.py` (Client Nodes - **Includes Server Liveness Check**)

```python
# client.py (Run on Client Node-1 and Node-2)
import socket
import datetime
import sys
import dme_middleware
import time

# --- CONFIGURATION (UPDATE THESE IPs) ---
SERVER_IP = '10.0.0.1' # Master Node IP
SERVER_PORT = 8000 # Server Port

if len(sys.argv) < 3:
    print("Usage: python client.py <MY_IP> <USER_ID>")
    sys.exit(1)

MY_IP = sys.argv[1]
USER_ID = sys.argv[2]
NODE_PROMPT = f"{USER_ID}_machine> "

# --- ADDED FEATURE: SERVER LIVENESS CHECK ---
def check_server_liveness():
    """Checks if the Master server is reachable."""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(3) # Set a 3-second timeout
        sock.connect((SERVER_IP, SERVER_PORT))
        sock.close()
        print(f"[CLIENT-STATUS] Connection to Server ({SERVER_IP}:{SERVER_PORT}) successful.")
        return True
    except Exception as e:
        print(f"[CLIENT-ERROR] Failed to connect to Server at {SERVER_IP}:{SERVER_PORT}.")
        print("Please ensure the Master node is running and the firewall is disabled.")
        return False

# --- FILE SERVER API ---
def send_to_server(message):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((SERVER_IP, SERVER_PORT))
        sock.sendall(message.encode())
        response = sock.recv(4096).decode()
        sock.close()
        return response
    except Exception as e:
        return f"CLIENT_ERROR: {e}"

def view_log():
    response = send_to_server("VIEW")
    print("--- CHAT LOG START ---")
    if not (response.startswith("CLIENT_ERROR") or response.startswith("SERVER_ERROR")):
        print(response.strip())
    print("--- CHAT LOG END ---")

def post_message(text):
    # DME: Request Critical Section
    dme_middleware.request_cs()
    print("... Critical Section entered ...")

    try:
        now = datetime.datetime.now()
        timestamp = now.strftime("%d %b %I:%M%p")
        log_entry = f"{timestamp} {USER_ID}: {text.strip()}" 
        
        response = send_to_server(f"POST {log_entry}")
        
        if response == "POST_SUCCESS":
            print(f"[{USER_ID}] Post successful.")
        else:
            print(f"[{USER_ID}] Post failed: {response}")

    except Exception as e:
        print(f"An error occurred during POST operation: {e}")
        
    finally:
        # DME: Release Critical Section
        dme_middleware.release_cs()
        print("... Critical Section released ...")

def run_cli():
    # 1. Check server liveness before starting DME
    if not check_server_liveness():
        sys.exit(1)

    # 2. Start DME Listener
    dme_middleware.MY_IP = MY_IP 
    dme_middleware.start_dme()
    
    print(f"Welcome, {USER_ID}! Type 'view' or 'post <text>'.")
    
    while True:
        try:
            command = input(NODE_PROMPT).strip()
            if command == 'exit': break
            if command == 'view': view_log()
            elif command.startswith('post '):
                text = command[5:].strip().strip('"')
                if text: post_message(text)
            
        except EOFError: break
        except Exception as e: print(f"An unexpected error occurred: {e}")

if __name__ == "__main__":
    run_cli()
```

### B. Distributed Mutual Exclusion Code

#### `dme_middleware.py` (Client Nodes)

```python
# dme_middleware.py (Run on Client Node-1 and Node-2)
import socket
import threading
import sys
import time

# --- CONFIGURATION (UPDATE THESE IPs) ---
PEERS = {
    '10.0.0.2': 8001, # Node-1 IP:Port
    '10.0.0.3': 8001, # Node-2 IP:Port
}
MY_IP = None 
MY_PORT = 8001
OTHER_PEERS = {}

# --- DME STATE ---
clock = 0
requesting_cs = False
reply_count = 0
deferred_requests = {} 
reply_lock = threading.Lock()
cs_event = threading.Event()

def dme_log(message):
    print(f"[DME-LOG | {MY_IP}] {message}")

def send_reply(peer_ip, peer_port):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((peer_ip, peer_port))
        sock.sendall(f"REPLY {MY_IP}".encode())
        sock.close()
    except Exception as e:
        dme_log(f"Error sending REPLY to {peer_ip}: {e}")

def handle_dme_message(conn, addr):
    global clock, requesting_cs, reply_count, deferred_requests
    try:
        data = conn.recv(1024).decode().split()
        if not data: return
        msg_type, peer_ip = data[0], data[1]

        with reply_lock:
            if len(data) > 2:
                peer_time = int(data[2])
                clock = max(clock, peer_time) + 1

            if msg_type == 'REQUEST':
                peer_time = int(data[2])
                
                should_defer = (
                    requesting_cs and 
                    (peer_time > clock or (peer_time == clock and peer_ip > MY_IP))
                )
                
                if should_defer:
                    deferred_requests[peer_ip] = peer_time
                    dme_log(f"Received REQUEST from {peer_ip}. Deferred.")
                else:
                    send_reply(peer_ip, PEERS[peer_ip])
            
            elif msg_type == 'REPLY':
                reply_count += 1
                dme_log(f"Received REPLY from {peer_ip}. Count: {reply_count}/{len(OTHER_PEERS)}")
                
                if requesting_cs and reply_count == len(OTHER_PEERS):
                    cs_event.set()

    except Exception as e:
        dme_log(f"Error in DME message handler: {e}")
    finally:
        conn.close()

def dme_listener():
    dme_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        dme_socket.bind((MY_IP, MY_PORT))
        dme_socket.listen(len(PEERS) - 1)
        dme_log(f"DME listener started on {MY_IP}:{MY_PORT}")
        while True:
            conn, addr = dme_socket.accept()
            threading.Thread(target=handle_dme_message, args=(conn, addr)).start()
    except Exception as e:
        dme_log(f"DME Listener error: {e}")
        
def start_dme():
    global OTHER_PEERS
    OTHER_PEERS = {ip: port for ip, port in PEERS.items() if ip != MY_IP}
    threading.Thread(target=dme_listener, daemon=True).start()

def request_cs():
    global clock, requesting_cs, reply_count
    with reply_lock:
        clock += 1 
        requesting_cs = True
        reply_count = 0
        cs_event.clear()

    for ip, port in OTHER_PEERS.items():
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.connect((ip, port))
            sock.sendall(f"REQUEST {MY_IP} {clock}".encode())
            sock.close()
        except Exception as e:
            dme_log(f"Error sending REQUEST to {ip}: {e}")
            
    if len(OTHER_PEERS) > 0:
        cs_event.wait() 

def release_cs():
    global requesting_cs, deferred_requests
    with reply_lock:
        requesting_cs = False
        dme_log("Releasing CS. Sending deferred replies.")
        peers_to_reply = list(deferred_requests.keys())
        deferred_requests = {}
        
        for peer_ip in peers_to_reply:
            send_reply(peer_ip, PEERS[peer_ip])
```

-----

## 3. Setup and Execution Steps

### Step 3.1: Stop Firewalls (Crucial for Cloud Lab)

Run **one** of the following commands on **ALL THREE NODES** (Master, Node-1, and Node-2):

```bash
# Option 1 (for firewalld/CentOS/RHEL)
sudo systemctl stop firewalld

# Option 2 (for ufw/Ubuntu/Debian)
sudo ufw disable
```

### Step 3.2: System Startup

| Node | Action | Command |
| :--- | :--- | :--- |
| **Master (10.0.0.1)** | **Start File Server** | `python3 chat_server.py` |
| **Node-1 (10.0.0.2)** | **Start Client 1 (Lucy)** | `python3 client.py 10.0.0.2 Lucy` |
| **Node-2 (10.0.0.3)** | **Start Client 2 (Joel)** | `python3 client.py 10.0.0.3 Joel` |

-----

## 4. Test Case Execution Commands

These commands are entered into the client prompts (`Lucy_machine>` or `Joel_machine>`).

### Test Case 1: Server Liveness Check (Verification of Added Feature)

| Step | Node | Command | Verification |
| :--- | :--- | :--- | :--- |
| **1.** | **Master** | Stop the server (`Ctrl+C`). | Server stops. |
| **2.** | **Node-1** | Rerun client: `python3 client.py 10.0.0.2 Lucy` | Client **fails to launch** and outputs: `[CLIENT-ERROR] Failed to connect to Server...`. |
| **3.** | **Master** | **Restart the server** (`python3 chat_server.py`). | Client functionality restored. |

### Test Case 2: DME Protocol Contention (Critical Test)

This test proves mutual exclusion and requires capturing DME logs.

| Step | Node | Command (at prompt) | Expected Action/Log Output |
| :--- | :--- | :--- | :--- |
| **1.** | **Node-1 (Lucy)** | `post "I am going first"` | Lucy requests CS. |
| **2.** | **Node-2 (Joel)** | **IMMEDIATELY** run: `post "I am going second"` | Joel requests CS and **blocks**. Lucy's DME log shows Joel's request was **Deferred**. |
| **3.** | **Wait** | *(Wait for Lucy's post to complete)* | Lucy releases CS, sends the deferred REPLY. Joel unblocks, enters CS, and completes his post. |
| **4.** | **Node-1 (Lucy)** | `view` | Final log shows Msg 1 followed by Msg 2 (enforced sequential order). |

### Test Case 3: View During Contention

This test verifies read access (`view`) is non-blocking.

| Step | Node | Command (at prompt) | Verification |
| :--- | :--- | :--- | :--- |
| **1.** | **Node-1 (Lucy)** | `post "A long comment"` | Lucy requests CS (blocks). |
| **2.** | **Node-2 (Joel)** | **IMMEDIATELY** run: `view` | Joel's `view` executes instantly, showing the log *before* Lucy's new comment is visible. |

### Test Case 4: View Consistency & Format

| Step | Node | Command (at prompt) | Verification |
| :--- | :--- | :--- | :--- |
| **1.** | **Node-1 (Lucy)** | `view` | Displays the full log. |
| **2.** | **Node-2 (Joel)** | `view` | Displays the **identical** content. |
| **3.** | **Node-2 (Joel)** | `post "Final check of the required format"` | Log includes the correct format: `[Time] Joel: Final check...`. |
````
