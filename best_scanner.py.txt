import socket
import ipaddress
import threading
import platform
import subprocess
from queue import Queue
from time import time
from tabulate import tabulate

print("🚀 Advanced Network Scanner (Educational Use Only)")

# ---------------- Settings ----------------
MAX_THREADS = 100
COMMON_PORTS = [21, 22, 23, 25, 53, 80, 110, 135, 139, 143, 443, 445, 3306, 3389]
timeout = 1
lock = threading.Lock()
queue = Queue()
results = []

# ------------- Ping Host (Cross-platform) -------------
def ping_host(ip):
    system = platform.system()
    if system == "Windows":
        cmd = ["ping", "-n", "1", "-w", "1000", str(ip)]
    else:
        cmd = ["ping", "-c", "1", "-W", "1", str(ip)]
    return subprocess.call(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL) == 0

# ------------- Port Scan -----------------
def scan_port(ip, port):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.settimeout(timeout)
        try:
            sock.connect((ip, port))
            with lock:
                results.append((str(ip), port, "Open"))
        except:
            pass  # silently ignore closed ports

# ------------- Thread Worker -------------
def worker():
    while not queue.empty():
        ip, port = queue.get()
        scan_port(ip, port)
        queue.task_done()

# ------------- Main Scanner -------------
def main():
    target_subnet = input("Enter target subnet (e.g., 192.168.1.0/24): ")
    try:
        network = ipaddress.ip_network(target_subnet, strict=False)
    except ValueError:
        print("Invalid subnet format.")
        return

    print(f"\n🔍 Scanning subnet {target_subnet}...\n")
    live_hosts = []
    start_time = time()

    # Step 1: Ping sweep
    for ip in network.hosts():
        if ping_host(ip):
            live_hosts.append(ip)

    if not live_hosts:
        print("❌ No live hosts found.")
        return

    print(f"[+] Live hosts found: {len(live_hosts)}")
    for ip in live_hosts:
        print(f" - {ip}")

    # Step 2: Port scanning on live hosts
    print("\n🚪 Scanning common ports...\n")
    for ip in live_hosts:
        for port in COMMON_PORTS:
            queue.put((ip, port))

    threads = []
    for _ in range(min(MAX_THREADS, queue.qsize())):
        t = threading.Thread(target=worker)
        t.daemon = True
        t.start()
        threads.append(t)

    queue.join()

    # Output results
    print("\n📊 Scan Results:\n")
    if results:
        print(tabulate(results, headers=["Host", "Port", "Status"]))
    else:
        print("No open ports found.")

    print(f"\n✅ Scan completed in {round(time() - start_time, 2)} seconds.\n")

if __name__ == "__main__":
    main()
