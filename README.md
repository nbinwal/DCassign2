---
# Distributed Chat Room with Distributed Mutual Exclusion (DME)

This project implements a 3-node distributed "chat room" application, part of the Distributed Computing (CCZG 526) Assignment II. It allows project team members to exchange text messages/comments/notes and uses a **Distributed Mutual Exclusion (DME) Protocol** to ensure only one user can write (post) to the shared file at any time.

## 1. System Architecture and Roles

The system is implemented on a 3-node cloud lab environment, with roles separated as required.

| Node Name | Role | Code Modules | Port Usage | Protocol |
| :--- | :--- | :--- | :--- | :--- |
| **Master** | File Server | `chat_server.py` | TCP **8000** | Handles File I/O (VIEW/POST) |
| **Node-1 (Client)** | User Application | `client.py`, `dme_middleware.py` | TCP **8001** | Runs Chat App & Ricart-Agrawala DME |
| **Node-2 (Client)** | User Application | `client.py`, `dme_middleware.py` | TCP **8001** | Runs Chat App & Ricart-Agrawala DME |

**Shared Resource:** A single shared file (`chat_log.txt`) is stored on the Master node.

## 2. Configuration Setup (Critical)

**Before running, you MUST replace the placeholder IPs with the actual private IPs of your lab machines in the following code sections:**

| Node Role | Placeholder IP | Used In Code |
| :--- | :--- | :--- |
| Master (Server) | `10.0.0.1` | `client.py` (as `SERVER_IP`) |
| Node-1 (Client 1) | `10.0.0.2` | `dme_middleware.py` (as `PEERS`) |
| Node-2 (Client 2) | `10.0.0.3` | `dme_middleware.py` (as `PEERS`) |

-----

## 3. Code Modules

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
                print(f"[{addr[0]}] VIEW requested.")
                try:
                    with open(LOG_FILE, 'r') as f:
                        content = f.read()
                    conn.sendall(content.encode())
                except Exception as e:
                    conn.sendall(f"SERVER_ERROR: {e}".encode())

            elif command == 'POST':
                text_to_append = parts[1]
                print(f"[{addr[0]}] POST request received. Appending: {text_to_append.strip()}")
                try:
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
````

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
    """Implements the 'view' command."""
    print("--- CHAT LOG START ---")
    response = send_to_server("VIEW")
    if response.startswith("CLIENT_ERROR") or response.startswith("SERVER_ERROR"):
        print(response)
    else:
        print(response.strip())
    print("--- CHAT LOG END ---")

def post_message(text):
    """Implements the 'post' command, requires DME for write access."""
    
    dme_middleware.request_cs()
    print("... Critical Section entered (Write Access Granted) ...")

    try:
        now = datetime.datetime.now()
        timestamp = now.strftime("%d %b %I:%M%p")
        log_entry = f"{timestamp} {USER_ID}: {text.strip()}" 
        
        print(f"[{USER_ID}] Posting message to server: '{text}'")
        response = send_to_server(f"POST {log_entry}")
        
        if response == "POST_SUCCESS":
            print(f"[{USER_ID}] Post successful.")
        else:
            print(f"[{USER_ID}] Post failed: {response}")

    except Exception as e:
        print(f"An error occurred during POST operation: {e}")
        
    finally:
        dme_middleware.release_cs()
        print("... Critical Section released (Write Access Revoked) ...")

def run_cli():
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

PEERS = {
    '10.0.0.2': 8001,
    '10.0.0.3': 8001,
}
MY_IP = None 
MY_PORT = 8001
OTHER_PEERS = {}

clock = 0
requesting_cs = False
reply_count = 0
deferred_requests = {}
reply_lock = threading.Lock()
cs_event = threading.Event()
dme_socket = None

def dme_log(message):
    print(f"[DME-LOG | {MY_IP}] {message}")

def send_reply(peer_ip, peer_port):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((peer_ip, peer_port))
        sock.sendall(f"REPLY {MY_IP}".encode())
        sock.close()
        dme_log(f"Sent REPLY to {peer_ip}")
    except Exception as e:
        dme_log(f"Error sending REPLY to {peer_ip}: {e}")

def handle_dme_message(conn, addr):
    global clock, requesting_cs, reply_count, deferred_requests, OTHER_PEERS
    
    try:
        data = conn.recv(1024).decode().split()
        if not data: return

        msg_type = data[0]
        peer_ip = data[1]

        with reply_lock:
            if len(data) > 2:
                peer_time = int(data[2])
                clock = max(clock, peer_time) + 1

            if msg_type == 'REQUEST':
                peer_time = int(data[2])
                dme_log(f"Received REQUEST from {peer_ip} at time {peer_time}")

                should_defer = (
                    requesting_cs and 
                    (peer_time > clock or (peer_time == clock and peer_ip > MY_IP))
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
                    cs_event.set()
                    dme_log("All replies received. Critical Section granted.")

    except Exception as e:
        dme_log(f"Error in DME message handler: {e}")
    finally:
        conn.close()

def dme_listener():
    global dme_socket
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
    listener_thread = threading.Thread(target=dme_listener, daemon=True)
    listener_thread.start()

def request_cs():
    global clock, requesting_cs, reply_count
    
    with reply_lock:
        clock += 1
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

---

## 4. Execution Commands

### Step 4.1: Stop Firewalls

```bash
sudo systemctl stop firewalld
# or
sudo ufw disable
```

### Step 4.2: Start the Server

```bash
python3 chat_server.py
```

### Step 4.3: Start the Clients

| Node   | Command                           |
| :----- | :-------------------------------- |
| Node-1 | `python3 client.py 10.0.0.2 Lucy` |
| Node-2 | `python3 client.py 10.0.0.3 Joel` |

## 5. Testing the System

### Test Case 1: View Consistency

| Node | Command               | Verification                                                             |
| :--- | :-------------------- | :----------------------------------------------------------------------- |
| Lucy | `view`                | Should display log content.                                              |
| Joel | `view` (concurrently) | Must display the same content immediately (read access is unrestricted). |

### Test Case 2: Contending Posts (DME Verification)

1. Lucy: `post "I am going first"`
2. Joel: `post "No, I am second"`
3. Verify: Logs should show proper DME coordination, deferred requests, and ordered posting.

### Test Case 3: Log Format

* Command: `post "Checking timestamp format"`
* Verify: The new entry should look like
  `17 Oct 02:57PM Lucy: Checking timestamp format`

---

## Contribution Table Example

| Sl. No. | Name           | ID NO   | Contribution                                                                                                  |
| :------ | :------------- | :------ | :------------------------------------------------------------------------------------------------------------ |
| 1       | [Student Name] | [ID NO] | Designed and implemented `dme_middleware.py` (Ricart-Agrawala Algorithm) and generated DME verification logs. |
| 2       | [Student Name] | [ID NO] | Implemented `chat_server.py`, the CLI in `client.py`, and managed cloud machine setup and testing.            |
| 3       | ...            | ...     | ...                                                                                                           |

---
