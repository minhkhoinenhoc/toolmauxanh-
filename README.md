import paho.mqtt.client as mqtt
import json
import time
import threading
import uuid
import ssl
import os
import requests
from termcolor import colored
import warnings

warnings.filterwarnings("ignore", category=DeprecationWarning)

running = True
cookies = []
delays = []
idbox = []
message = ""

live_cookies = 0
die_cookies = 0

live_list = []
die_list = []


def clear():
    os.system("cls" if os.name == "nt" else "clear")


def banner():

    clear()

    logo = [
"██████╗ ███╗   ██╗██████╗ ██╗  ██╗",
"██╔══██╗████╗  ██║██╔══██╗██║ ██╔╝",
"██████╔╝██╔██╗ ██║██║  ██║█████╔╝ ",
"██╔═══╝ ██║╚██╗██║██║  ██║██╔═██╗ ",
"██║     ██║ ╚████║██████╔╝██║  ██╗",
"╚═╝     ╚═╝  ╚═══╝╚═════╝ ╚═╝  ╚═╝"
]

    width = 70
    colors = ["white", "red"]

    print(colored("╔" + "═"*width + "╗", "red"))

    color_index = 0

    for line in logo:
        print(" ", end="")
        for char in line:
            color = colors[color_index % len(colors)]
            print(colored(char, color), end="", flush=True)
            color_index += 1
            time.sleep(0.0008)
        print()

    print(colored("╠" + "═"*width + "╣", "red"))

    dev = " Developer : Nguyễn Minh Khôi "
    app = " Tool treo messenger 2026 "

    for char in dev.center(width):
        print(colored(char, "white"), end="", flush=True)
        time.sleep(0.001)
    print()

    for char in app.center(width):
        print(colored(char, "red"), end="", flush=True)
        time.sleep(0.001)
    print()

    print(colored("╚" + "═"*width + "╝", "red"))

    print(colored("\n[ LOADING TOOL ]", "red"))

    for i in range(30):
        color = colors[i % len(colors)]
        print(colored("█", color), end="", flush=True)
        time.sleep(0.03)

    print("\n")
    time.sleep(0.5)


def log(text, color="green"):
    print(colored(f"[+] {text}", color))


def load_cookies(file_path):

    data = []

    try:
        with open(file_path, "r", encoding="utf-8") as f:
            for line in f:
                c = line.strip()
                if c:
                    data.append(c)

        log(f"Đã tải {len(data)} cookie")

    except:
        log("File cookie này sai rồi cha hay cha chưa tạo", "red")

    return data


def get_uid(cookie):

    for part in cookie.split(";"):
        if "c_user=" in part:
            return part.split("=")[1]

    return "Unknown"


def get_name(cookie):

    uid = get_uid(cookie)

    try:

        headers = {
            "cookie": cookie,
            "user-agent": "Mozilla/5.0"
        }

        url = f"https://graph.facebook.com/{uid}?fields=name"

        r = requests.get(url, headers=headers, timeout=10)

        data = r.json()

        if "name" in data:
            return data["name"]

    except:
        pass

    return "Unknown"


def cookie_live(cookie):

    try:

        headers = {
            "cookie": cookie,
            "user-agent": "Mozilla/5.0"
        }

        r = requests.get(
            "https://www.facebook.com/me",
            headers=headers,
            allow_redirects=False,
            timeout=10
        )

        if r.status_code == 302:
            return True

    except:
        pass

    return False


def get_token(cookie):

    parts = cookie.split(";")
    c_user = None
    xs = None

    for part in parts:

        part = part.strip()

        if part.startswith("c_user="):
            c_user = part.split("=")[1]

        if part.startswith("xs="):
            xs = part.split("=")[1]

    if c_user and xs:
        return f"{c_user}|{xs}"

    return cookie


def create_mqtt(cookie):

    try:

        token = get_token(cookie)

        client_id = f"mqttwsclient_{uuid.uuid4().hex[:8]}"

        client = mqtt.Client(
            client_id=client_id,
            transport="websockets",
            protocol=mqtt.MQTTv31
        )

        client.username_pw_set(
            username=json.dumps({
                "u": token.split("|")[0],
                "s": 1,
                "chat_on": True,
                "fg": True,
                "d": str(uuid.uuid4()),
                "ct": "websocket",
                "aid": 219994525426954
            }),
            password=""
        )

        client.tls_set(cert_reqs=ssl.CERT_NONE)
        client.tls_insecure_set(True)

        client.ws_set_options(
            path="/chat",
            headers={
                "Cookie": cookie,
                "Origin": "https://www.facebook.com",
                "User-Agent": "Mozilla/5.0"
            }
        )

        client.connect("edge-chat.facebook.com", 443, 60)

        client.loop_start()

        time.sleep(2)

        return client, token

    except:
        return None, None


def check_cookies():

    global live_cookies, die_cookies

    print()
    log("Đang check cookie...")

    for cookie in cookies:

        uid = get_uid(cookie)
        name = get_name(cookie)

        if cookie_live(cookie):

            live_cookies += 1
            info = f"{name} | {uid}"

            live_list.append(info)

            log(f"{name} | {uid} LIVE", "green")

        else:

            die_cookies += 1
            info = f"{name} | {uid}"

            die_list.append(info)

            log(f"{name} | {uid} DIE", "red")

    print()

    log(f"COOKIE LIVE : {live_cookies}", "green")
    log(f"COOKIE DIE  : {die_cookies}", "red")

    print()

    if live_list:

        log("LIVE LIST:", "green")

        for x in live_list:
            print(colored(x, "green"))

    print()


def send_message(client, token, thread_id, message, cookie_index):

    try:

        msg_id = str(int(time.time() * 1000))

        payload = {
            "body": message,
            "msgid": msg_id,
            "sender_fbid": token.split("|")[0],
            "to": thread_id,
            "offline_threading_id": msg_id
        }

        topic = "/send_message2"

        result = client.publish(topic, json.dumps(payload), qos=1)

        if result.rc == mqtt.MQTT_ERR_SUCCESS:
            log(f"Cookie {cookie_index+1} gửi tới {thread_id}", "green")

        else:
            log(f"Cookie {cookie_index+1} lỗi gửi", "red")

    except Exception as e:
        log(f"Lỗi: {e}", "red")


def worker(cookie_index, cookie, delay):

    global running

    client, token = create_mqtt(cookie)

    if not client:
        log(f"Cookie {cookie_index+1} DIE - bỏ qua", "red")
        return

    while running:

        if not cookie_live(cookie):
            log(f"Cookie {cookie_index+1} đã DIE trong lúc chạy", "red")
            break

        for thread_id in idbox:
            send_message(client, token, thread_id, message, cookie_index)

        time.sleep(delay)

    client.loop_stop()
    client.disconnect()


def main():

    global cookies, idbox, message, delays

    banner()

    file_cookie = input("Nhập file cookie: ")

    cookies = load_cookies(file_cookie)

    if not cookies:
        return

    check_cookies()

    ids = input("Nhập ID box (cách nhau ,): ")

    idbox = [x.strip() for x in ids.split(",") if x.strip()]

    file_txt = input("Nhập file nội dung: ")

    try:

        with open(file_txt, "r", encoding="utf-8") as f:
            message = f.read().strip()

    except:

        log("Không đọc được file nội dung", "red")
        return

    for i in range(len(cookies)):

        try:
            d = float(input(f"Delay cookie {i+1}: "))
        except:
            d = 5

        delays.append(d)

    log("Bắt đầu chạy tool")

    threads = []

    for i, cookie in enumerate(cookies):

        t = threading.Thread(target=worker, args=(i, cookie, delays[i]))
        t.daemon = True
        t.start()

        threads.append(t)

        time.sleep(1)

    try:

        while True:
            time.sleep(1)

    except KeyboardInterrupt:

        log("Đã dừng tool", "yellow")


if __name__ == "__main__":
    main()
