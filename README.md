-----

# Distributed Chat Room with Distributed Mutual Exclusion (DME)

[cite\_start]This project implements a 3-node distributed "chat room" application, part of the Distributed Computing (CCZG 526) Assignment II[cite: 2]. [cite\_start]It allows project team members to exchange text messages/comments/notes [cite: 5] [cite\_start]and uses a **Distributed Mutual Exclusion (DME) Protocol** to ensure only one user can write (post) to the shared file at any time[cite: 19, 20].

## 1\. System Architecture and Roles

[cite\_start]The system is implemented on a 3-node cloud lab environment [cite: 7][cite\_start], with roles separated as required[cite: 39].

| Node Name | Role | Code Modules | Port Usage | Protocol |
| :--- | :--- | :--- | :--- | :--- |
| **Master** | File Server | `chat_server.py` | TCP **8000** | [cite\_start]Handles File I/O (VIEW/POST) [cite: 16] |
| **Node-1 (Client)** | User Application | `client.py`, `dme_middleware.py` | TCP **8001** | [cite\_start]Runs Chat App & Ricart-Agrawala DME [cite: 12] |
| **Node-2 (Client)** | User Application | `client.py`, `dme_middleware.py` | TCP **8001** | [cite\_start]Runs Chat App & Ricart-Agrawala DME [cite: 12] |

[cite\_start]**Shared Resource:** A single shared file (`chat_log.txt`) is stored on the Master node[cite: 9, 10].

## 2\. Configuration Setup (Critical)

**Before running, you MUST replace the placeholder IPs with the actual private IPs of your lab machines in the following code sections:**

| Node Role | Placeholder IP | Used In Code |
| :--- | :--- | :--- |
| Master (Server) | `10.0.0.1` | `client.py` (as `SERVER_IP`) |
| Node-1 (Client 1) | `10.0.0.2` | `dme_middleware.py` (as `PEERS`) |
| Node-2 (Client 2) | `10.0.0.3` | `dme_middleware.py` (as `PEERS`) |

-----

## 3\. Code Modules

### A. Distributed Application Code

#### `chat_server.py` (Master Node)

```python
# chat_server.py (Run on Master Node: e.g., 10.0.0.1)
import socket
import threading
import os

SERVER_HOST = '0.0.0.0' # Listen on all interfaces
SERVER_PORT = 8000
LOG_FILE = 'chat_log.txt'

def initialize_log():
    """Ensure the log file exists."""
    if not os.path.exists(LOG_FILE):
        with open(LOG_FILE, 'w') as f:
            f.write("--- Chat Room Initialized ---\n")
        print(f"Created new log file: {LOG_FILE}")

def handle_client(conn, addr):
    """Handles view and post commands from a client."""
    print(f"Connection from {addr}")
    try:
        while True:
            data = conn.recv(1024).decode()
            if not data:
                break
            
            parts = data.split(' ', 1)
            command = parts[0]
            
            if command == 'VIEW':
                # [cite_start]No mutual exclusion for VIEW [cite: 18]
                print(f"[{addr[0]}] VIEW requested.")
                try:
                    with open(LOG_FILE, 'r') as f:
                        content = f.read()
                    conn.sendall(content.encode())
                except Exception as e:
                    conn.sendall(f"SERVER_ERROR: {e}".encode())

            elif command == 'POST':
                # [cite_start]Access controlled by client-side DME [cite: 19]
                text_to_append = parts[1]
                print(f"[{addr[0]}] POST request received. Appending: {text_to_append.strip()}")
                try:
                    # [cite_start]Append the text to the file [cite: 21]
                    with open(LOG_FILE, 'a') as f:
                        f.write(text_to_append + '\n')
                    conn.sendall("POST_SUCCESS".encode())
                except Exception as e:
                    conn.sendall(f"SERVER_ERROR: {e}".encode())
            
            else:
                conn.sendall("UNKNOWN_COMMAND".encode())

    except Exception as e:
        print(f"Error handling connection from {addr}: {e}")
    finally:
        print(f"Connection closed from {addr}")
        conn.close()

def start_server():
    initialize_log()
    
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.bind((SERVER_HOST, SERVER_PORT))
    server_socket.listen(5)
    print(f"Chat Server listening on {SERVER_HOST}:{SERVER_PORT}")

    while True:
        conn, addr = server_socket.accept()
        client_thread = threading.Thread(target=handle_client, args=(conn, addr))
        client_thread.start()

if __name__ == "__main__":
    start_server()
```

#### `client.py` (Client Nodes)

```python
# client.py (Run on Client Node-1 and Node-2)
import socket
import datetime
import sys
import dme_middleware

# --- CONFIGURATION (UPDATE THESE IPs) ---
SERVER_IP = '10.0.0.1' # Master Node IP

if len(sys.argv) < 3:
    print("Usage: python client.py <MY_IP> <USER_ID>")
    sys.exit(1)

MY_IP = sys.argv[1]
USER_ID = sys.argv[2]
NODE_PROMPT = f"{USER_ID}_machine> "

# --- FILE SERVER API ---
def send_to_server(message):
    """Generic function to communicate with the file server."""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((SERVER_IP, 8000))
        sock.sendall(message.encode())
        response = sock.recv(4096).decode()
        sock.close()
        return response
    except Exception as e:
        return f"CLIENT_ERROR: Could not connect to server or connection failed: {e}"

def view_log():
    [cite_start]"""Implements the 'view' command[cite: 14]."""
    print("--- CHAT LOG START ---")
    response = send_to_server("VIEW")
    if response.startswith("CLIENT_ERROR") or response.startswith("SERVER_ERROR"):
        print(response)
    else:
        print(response.strip())
    print("--- CHAT LOG END ---")

def post_message(text):
    [cite_start]"""Implements the 'post' command, requires DME for write access[cite: 15]."""
    
    # [cite_start]1. DME: Request Critical Section [cite: 20]
    dme_middleware.request_cs()
    print("... Critical Section entered (Write Access Granted) ...")

    try:
        # [cite_start]2. Prepare the log entry (Client time, ID, Text) [cite: 24]
        now = datetime.datetime.now()
        timestamp = now.strftime("%d %b %I:%M%p") # e.g., 17 Oct 02:57PM
        log_entry = f"{timestamp} {USER_ID}: {text.strip()}" 
        
        # [cite_start]3. RPC/API: Send POST command to server [cite: 16]
        print(f"[{USER_ID}] Posting message to server: '{text}'")
        response = send_to_server(f"POST {log_entry}")
        
        if response == "POST_SUCCESS":
            print(f"[{USER_ID}] Post successful.")
        else:
            print(f"[{USER_ID}] Post failed: {response}")

    except Exception as e:
        print(f"An error occurred during POST operation: {e}")
        
    finally:
        # 4. DME: Release Critical Section
        dme_middleware.release_cs()
        print("... Critical Section released (Write Access Revoked) ...")

def run_cli():
    # Start DME Listener in the background
    dme_middleware.start_dme()
    
    print(f"Welcome, {USER_ID}! Type 'view' or 'post <text>'. Type 'exit' to quit.")
    
    while True:
        try:
            command = input(NODE_PROMPT).strip()
            if not command: continue

            if command == 'exit': break
                
            if command == 'view':
                view_log()
                
            elif command.startswith('post '):
                text = command[5:].strip().strip('"')
                if text:
                    post_message(text)
                else:
                    print("Error: 'post' command requires text input.")

            else:
                print(f"Unknown command: {command}")

        except EOFError: break
        except Exception as e: print(f"An unexpected error occurred: {e}")

if __name__ == "__main__":
    dme_middleware.MY_IP = MY_IP 
    run_cli()
```

### B. Distributed Mutual Exclusion Code

#### `dme_middleware.py` (Client Nodes)

```python
# dme_middleware.py (Run on Client Node-1 and Node-2)
import socket
import threading
import time
import sys

# --- CONFIGURATION (UPDATE THESE IPs) ---
# List of all *CLIENT* nodes participating in DME (Node-1 and Node-2)
PEERS = {
    '10.0.0.2': 8001, # Node-1 IP:Port
    '10.0.0.3': 8001, # Node-2 IP:Port
}
# Your own IP/Port for DME communication (Set by client.py)
MY_IP = None 
MY_PORT = 8001 # Shared DME Port
OTHER_PEERS = {}

# --- DME STATE ---
clock = 0
requesting_cs = False
reply_count = 0
deferred_requests = {} # {peer_ip: request_time}
reply_lock = threading.Lock()
cs_event = threading.Event()
dme_socket = None

# --- LOGGING UTILITY ---
def dme_log(message):
    print(f"[DME-LOG | {MY_IP}] {message}")

# --- DME CORE FUNCTIONS ---
def send_reply(peer_ip, peer_port):
    """Sends a REPLY message to a peer."""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((peer_ip, peer_port))
        sock.sendall(f"REPLY {MY_IP}".encode())
        sock.close()
        dme_log(f"Sent REPLY to {peer_ip}")
    except Exception as e:
        dme_log(f"Error sending REPLY to {peer_ip}: {e}")

def handle_dme_message(conn, addr):
    """Processes incoming DME request/reply messages."""
    global clock, requesting_cs, reply_count, deferred_requests, OTHER_PEERS
    
    try:
        data = conn.recv(1024).decode().split()
        if not data: return

        msg_type = data[0]
        peer_ip = data[1]

        with reply_lock:
            if len(data) > 2:
                peer_time = int(data[2])
                clock = max(clock, peer_time) + 1 # Update clock

            if msg_type == 'REQUEST':
                peer_time = int(data[2])
                dme_log(f"Received REQUEST from {peer_ip} at time {peer_time}")

                # Ricart-Agrawala Decision Logic
                should_defer = (
                    requesting_cs and # I want CS
                    (peer_time > clock or (peer_time == clock and peer_ip > MY_IP)) # Peer is later or has higher ID
                )
                
                if should_defer:
                    deferred_requests[peer_ip] = peer_time
                    dme_log(f"Deferred request from {peer_ip}. My request time: {clock}")
                else:
                    send_reply(peer_ip, PEERS[peer_ip])
            
            elif msg_type == 'REPLY':
                reply_count += 1
                dme_log(f"Received REPLY from {peer_ip}. Count: {reply_count}/{len(OTHER_PEERS)}")
                
                if requesting_cs and reply_count == len(OTHER_PEERS):
                    cs_event.set() # Signal that CS is granted
                    dme_log("All replies received. Critical Section granted.")

    except Exception as e:
        dme_log(f"Error in DME message handler: {e}")
    finally:
        conn.close()

def dme_listener():
    """Listens for incoming DME messages (REQUEST/REPLY)."""
    global dme_socket
    dme_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        # We must use MY_IP here to bind to the correct interface
        dme_socket.bind((MY_IP, MY_PORT))
        dme_socket.listen(len(PEERS) - 1)
        dme_log(f"DME listener started on {MY_IP}:{MY_PORT}")
        while True:
            conn, addr = dme_socket.accept()
            threading.Thread(target=handle_dme_message, args=(conn, addr)).start()
    except Exception as e:
        dme_log(f"DME Listener error: {e}")
        
def start_dme():
    """Initializes the DME listener thread."""
    global OTHER_PEERS
    OTHER_PEERS = {ip: port for ip, port in PEERS.items() if ip != MY_IP}
    listener_thread = threading.Thread(target=dme_listener, daemon=True)
    listener_thread.start()

# --- APPLICATION INTERFACE ---
def request_cs():
    """Called by the application to request the Critical Section (for POST)."""
    global clock, requesting_cs, reply_count
    
    with reply_lock:
        clock += 1 # Update clock for new request
        request_time = clock
        requesting_cs = True
        reply_count = 0
        cs_event.clear()

    dme_log(f"Requesting CS at time {request_time}. Sending REQUEST to all peers.")

    for ip, port in OTHER_PEERS.items():
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.connect((ip, port))
            sock.sendall(f"REQUEST {MY_IP} {request_time}".encode())
            sock.close()
        except Exception as e:
            dme_log(f"Error sending REQUEST to {ip}: {e}")
            
    if len(OTHER_PEERS) == 0:
        dme_log("No other peers. CS granted immediately.")
        return 
    
    cs_event.wait() # Blocks until all replies are received

def release_cs():
    """Called by the application to release the Critical Section."""
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

## 4\. Execution Commands

### Step 4.1: Stop Firewalls (Must be done on all 3 nodes)

```bash
# Option 1 (for firewalld/CentOS/RHEL)
sudo systemctl stop firewalld

# Option 2 (for ufw/Ubuntu/Debian)
sudo ufw disable
```

### Step 4.2: Start the Server

On the **Master Node** (`10.0.0.1`):

```bash
python3 chat_server.py
```

### Step 4.3: Start the Clients

On **Node-1** and **Node-2**, execute the command passing the node's **IP** and a **User ID**.

| Node | Command |
| :--- | :--- |
| **Node-1** | `python3 client.py 10.0.0.2 Lucy` |
| **Node-2** | `python3 client.py 10.0.0.3 Joel` |

## 5\. Testing the System

Use these commands from the client prompts (` Lucy_machine>  ` or ` Joel_machine>  `) to verify functionality and DME.

### Test Case 1: View Consistency (Simultaneous Read)

| Node | Command | Verification |
| :--- | :--- | :--- |
| Lucy | `view` | Should display log content. |
| Joel | `view` (concurrently) | Must display the same content immediately. [cite\_start](Read access is unrestricted [cite: 18]) |

### Test Case 2: Contending Posts (DME Verification)

This is the primary test for the Distributed Mutual Exclusion protocol.

1.  **Lucy:** Execute `post "I am going first"`
2.  **Joel:** Immediately execute `post "No, I am second"`
3.  **Verification:**
      * Check **DME Logs** on both consoles. Lucy's log should show `CS granted` followed by Joel's request being `Deferred`. Joel's log should show it's blocked, waiting for the $\text{REPLY}$.
      * After Lucy's post completes, Joel's $\text{REPLY}$ is sent, and Joel's post executes.
      * The final log (`view` command) will show both messages correctly appended in the order determined by the DME algorithm.

### Test Case 3: Log Format

  * **Command (Any Client):** `post "Checking timestamp format"`
  * [cite\_start]**Verification:** The new entry in the chat log must be correctly formatted with the client's timestamp and ID[cite: 24].
      * [cite\_start]Example: `17 Oct 02:57PM Lucy: Checking timestamp format` [cite: 26, 27, 28]

-----

## Contribution Table Example

| Sl. No. | Name | ID NO | Contribution |
| :--- | :--- | :--- | :--- |
| 1 | [Student Name] | [ID NO] | Designed and implemented `dme_middleware.py` (Ricart-Agrawala Algorithm) and generated DME verification logs. |
| 2 | [Student Name] | [ID NO] | Implemented `chat_server.py`, the CLI in `client.py`, and managed cloud machine setup and testing. |
| 3 | ... | ... | ... |
