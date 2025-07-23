# 🎓 Final Year Project – Python-Based Botnet Simulation

## 📌 Project Title
**Development of a Botnet Command & Control (C2) System for Educational and Research Purposes**

## 👨‍🏫 Supervisor:
**

## 👨‍🎓 Student:
**Ayoub Faddil**
**oussma bekich**

---

## 📖 Project Description

This project simulates a **Command and Control (C2) botnet** system, entirely written in Python, intended **for educational use only**. It demonstrates how centralized servers can control multiple compromised machines (bots), allowing the execution of remote commands, file transfer, data exfiltration, and more.

The goal is to study real-world attack mechanisms and strengthen understanding of defensive strategies used in cybersecurity environments, such as in SOC (Security Operations Centers).

---

## 🧱 Architecture

- **Server (`server.py`)**: A GUI-based control center using Tkinter that allows an operator to manage and interact with all connected bots.
- **Client (`client.py`)**: The agent that runs on a victim machine and connects back to the server to receive and execute commands.

---

## 🔧 Features

### Server (Botnet Control Center)
- Built with `Tkinter` for graphical interface
- Manages multiple bots (clients) via threading
- Live shell interaction
- Log output per command type
- Secure file upload/download system
- Tabbed GUI with logs per action
- Custom styling for dark mode GUI

### Client (Bot Agent)
- Connects to the server using sockets
- Keylogger with timestamped logs
- Screenshot and webcam snapshot
- File upload and download
- Remote shell commands
- Credential theft (browsers, SSH keys, AWS)
- Automatic persistence on Windows startup

---


---

## 🚀 How to Run

### 🖥️ 1. Server

python server.py

### 🖥️ 1. Client
python client.py

##🛠 Technologies Used
Python 3.x

Tkinter (GUI)

Sockets (Networking)

OpenCV (Webcam capture)

PyAutoGUI (Screenshots)

Pynput (Keylogger)

Winreg (Persistence)

SQLite3 (Credential extraction)

ZipFile & OS modules (File handling)

##✅ Educational Value
This project demonstrates:

Reverse shell and C2 communication

Malware persistence and stealth

Automation of credential and data theft

Safe simulation of offensive cybersecurity practices

##📬 Contact
If you have questions or suggestions:

📧 ayoub8faddil@gmail.com


##📄 To explore the technical implementation in detail, please refer to the `server.py` and `client.py` files provided in this repository.
