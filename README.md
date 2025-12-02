#!/usr/bin/env python3
import socket
import threading
import select
import struct
import hashlib
import os

# Encryption: simple XOR for demo, replace with real crypto for security!
def xor_encrypt(data, key):
    return bytes([b ^ key[i % len(key)] for i, b in enumerate(data)])

# Shadowsocks v3 basic header parser/generator
def parse_ss_header(sock):
    addrtype = ord(sock.recv(1))
    if addrtype == 1:  # IPv4
        addr = socket.inet_ntoa(sock.recv(4))
    elif addrtype == 3:  # Domain
        length = ord(sock.recv(1))
        addr = sock.recv(length).decode()
    elif addrtype == 4:  # IPv6
        addr = socket.inet_ntop(socket.AF_INET6, sock.recv(16))
    else:
        raise Exception("Unsupported addrtype: %d" % addrtype)
    port = struct.unpack('>H', sock.recv(2))[0]
    return (addrtype, addr, port)

def create_remote_socket(addr, port):
    family = socket.AF_INET6 if ':' in addr else socket.AF_INET
    s = socket.socket(family, socket.SOCK_STREAM)
    s.connect((addr, port))
    return s

def handle_client(local, password):
    try:
        # Decrypt data for demonstration (not secure)
        key = hashlib.md5(password.encode()).digest()
        initial = local.recv(4096)
        data = xor_encrypt(initial, key)

        # parse SS header and connect
        from io import BytesIO
        bio = BytesIO(data)
        addrtype = ord(bio.read(1))
        if addrtype == 1:
            addr = socket.inet_ntoa(bio.read(4))
        elif addrtype == 3:
            domain_len = ord(bio.read(1))
            addr = bio.read(domain_len).decode()
        else:
            local.close()
            return
        port = struct.unpack('>H', bio.read(2))[0]
        remote = create_remote_socket(addr, port)

        # Forward all remaining data
        def forward(src, dst, enc):
            while True:
                ready, _, _ = select.select([src], [], [], 60)
                if src in ready:
                    data = src.recv(4096)
                    if not data: break
                    dst.sendall(enc(data, key))
        t1 = threading.Thread(target=forward, args=(local, remote, lambda d, k: xor_encrypt(d, k)))
        t2 = threading.Thread(target=forward, args=(remote, local, lambda d, k: xor_encrypt(d, k)))
        t1.start()
        t2.start()
        t1.join()
        t2.join()
    finally:
        local.close()

def run_shadowsocks_server(host='0.0.0.0', port=8388, password="secret"):
    print(f"Shadowsocks v3 demo proxy running on {host}:{port}")
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind((host, port))
    server.listen(5)
    while True:
        client, _ = server.accept()
        threading.Thread(target=handle_client, args=(client, password), daemon=True).start()

if __name__ == '__main__':
    run_shadowsocks_server()Removed according to regulations.
