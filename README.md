import base64
import datetime
import hashlib
import imaplib
import email
import smtplib
from email.mime.text import MIMEText
import tkinter as tk
from tkinter import messagebox, scrolledtext

# ==========================================
# 1. КРИПТИРАНЕ С ДИНАМИЧЕН КЛЮЧ (30 МИН)
# ==========================================
def get_30min_key(shared_secret: str) -> bytes:
    """Генерира ключ на базата на тайна дума + текущия 30-минутен прозорец."""
    now = datetime.datetime.utcnow()
    minute_slot = 0 if now.minute < 30 else 30
    time_window = now.strftime(f"%Y-%m-%d-%H:{minute_slot:02d}")

    combined = f"{shared_secret}_{time_window}".encode('utf-8')
    # Генерираме 256-битов ключ чрез SHA-256
    return hashlib.sha256(combined).digest()

def xor_crypt(data: bytes, key: bytes) -> bytes:
    """Просто и ефективно XOR шифроване/декриптиране."""
    return bytes([b ^ key[i % len(key)] for i, b in enumerate(data)])

def encrypt_message(message: str, shared_secret: str) -> str:
    key = get_30min_key(shared_secret)
    encrypted_bytes = xor_crypt(message.encode('utf-8'), key)
    # Превръщаме в текст от всякакви символи (Base64)
    return base64.b64encode(encrypted_bytes).decode('utf-8')

def decrypt_message(encrypted_b64: str, shared_secret: str) -> str:
    try:
        key = get_30min_key(shared_secret)
        encrypted_bytes = base64.b64decode(encrypted_b64.encode('utf-8'))
        decrypted_bytes = xor_crypt(encrypted_bytes, key)
        return decrypted_bytes.decode('utf-8')
    except Exception:
        return "[ГРЕШКА: Грешен ключ или изтекъл 30-минутен прозорец!]"

# ==========================================
# 2. МРЕЖОВА ЧАСТ (ИМЕЙЛ)
# ==========================================
def send_email_smtp(smtp_server, port, sender_email, password, receiver_email, subject, body):
    msg = MIMEText(body)
    msg['Subject'] = subject
    msg['From'] = sender_email
    msg['To'] = receiver_email

    with smtplib.SMTP_SSL(smtp_server, int(port)) as server:
        server.login(sender_email, password)
        server.sendmail(sender_email, receiver_email, msg.as_string())

def fetch_latest_email_imap(imap_server, email_addr, password):
    try:
        mail = imaplib.IMAP4_SSL(imap_server)
        mail.login(email_addr, password)
        mail.select('inbox')

        status, data = mail.search(None, '(SUBJECT "FLASH_CHAT_MSG")')
        mail_ids = data[0].split()

        if not mail_ids:
            return None

        latest_id = mail_ids[-1]
        status, data = mail.fetch(latest_id, '(RFC822)')

        for response_part in data:
            if isinstance(response_part, tuple):
                msg = email.message_from_bytes(response_part[1])
                if msg.is_multipart():
                    for part in msg.walk():
                        if part.get_content_type() == "text/plain":
                            return part.get_payload(decode=True).decode('utf-8')
                else:
                    return msg.get_payload(decode=True).decode('utf-8')
    except Exception as e:
        return None
    return None

# ==========================================
# 3. ГРАФИЧЕН ИНТЕРФЕЙС
# ==========================================
class FlashChatApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Flash Secure Chat")
        self.root.geometry("480x580")

        tk.Label(root, text="Вашият Имейл:").pack(anchor="w", padx=10)
        self.ent_my_email = tk.Entry(root, width=50)
        self.ent_my_email.pack(padx=10, pady=2)

        tk.Label(root, text="Парола за имейл (App Password):").pack(anchor="w", padx=10)
        self.ent_password = tk.Entry(root, show="*", width=50)
        self.ent_password.pack(padx=10, pady=2)

        tk.Label(root, text="Имейл на приятеля:").pack(anchor="w", padx=10)
        self.ent_friend_email = tk.Entry(root, width=50)
        self.ent_friend_email.pack(padx=10, pady=2)

        tk.Label(root, text="Обща тайна дума:").pack(anchor="w", padx=10)
        self.ent_secret = tk.Entry(root, show="*", width=50)
        self.ent_secret.pack(padx=10, pady=2)

        # Подразбиращи се SMTP/IMAP за Gmail (можеш да ги смениш)
        self.smtp_server = "smtp.gmail.com"
        self.imap_server = "imap.gmail.com"

        tk.Label(root, text="Чат история:").pack(anchor="w", padx=10, pady=(10, 0))
        self.chat_area = scrolledtext.ScrolledText(root, height=10, state='disabled')
        self.chat_area.pack(fill="both", padx=10, pady=5, expand=True)

        tk.Label(root, text="Съобщение:").pack(anchor="w", padx=10)
        self.ent_msg = tk.Entry(root, width=50)
        self.ent_msg.pack(fill="x", padx=10, pady=2)

        btn_frame = tk.Frame(root)
        btn_frame.pack(fill="x", padx=10, pady=5)

        tk.Button(btn_frame, text="Прати Криптирано", command=self.send_msg, bg="#4CAF50", fg="white").pack(side="left", padx=5)
        tk.Button(btn_frame, text="Провери за нови", command=self.check_msg, bg="#2196F3", fg="white").pack(side="left", padx=5)

    def log(self, text):
        self.chat_area.config(state='normal')
        self.chat_area.insert(tk.END, text + "\n")
        self.chat_area.config(state='disabled')
        self.chat_area.yview(tk.END)

    def send_msg(self):
        msg = self.ent_msg.get()
        secret = self.ent_secret.get()
        friend = self.ent_friend_email.get()
        my_email = self.ent_my_email.get()
        pwd = self.ent_password.get()

        if not all([msg, secret, friend, my_email, pwd]):
            messagebox.showerror("Грешка", "Попълни всички полета!")
            return

        encrypted = encrypt_message(msg, secret)

        try:
            send_email_smtp(self.smtp_server, 465, my_email, pwd, friend, "FLASH_CHAT_MSG", encrypted)
            self.log(f"[АЗ]: {msg}")
            self.log(f"  └─> Зашифровано: {encrypted}")
            self.ent_msg.delete(0, tk.END)
        except Exception as e:
            messagebox.showerror("Грешка при изпращане", str(e))

    def check_msg(self):
        secret = self.ent_secret.get()
        my_email = self.ent_my_email.get()
        pwd = self.ent_password.get()

        if not all([secret, my_email, pwd]):
            messagebox.showerror("Грешка", "Въведи имейл, парола и тайна дума!")
            return

        raw_cipher = fetch_latest_email_imap(self.imap_server, my_email, pwd)
        if raw_cipher:
            decrypted = decrypt_message(raw_cipher.strip(), secret)
            self.log(f"[ПРИЯТЕЛ]: {decrypted}")
        else:
            self.log("[СИСТЕМА]: Няма нови съобщения.")

if __name__ == "__main__":
    root = tk.Tk()
    app = FlashChatApp(root)
    root.mainloop()
