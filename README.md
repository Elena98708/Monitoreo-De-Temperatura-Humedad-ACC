# Monitoreo-De-Temperatura-Humedad-ACC
from machine import Pin, PWM, I2C
from imu import MPU6050

import network
import time
import _thread
import socket
import dht
import urequests

SSID = "JPRU"
PASSWORD = "123456789"

# =========================================
# TELEGRAM
# =========================================

TOKEN = "7955960258:AAGdYJm7sfahsUFc8uhy5tqLMVFqBmPOWPY"
CHAT_ID = "8676291034"


# =========================================
# PINES
# =========================================

PIN_DHT = 18
PIN_BUZZER = 15
PIN_BOTON = 4

# MPU6050
PIN_SDA = 21
PIN_SCL = 22

# =========================================
# SENSOR DHT11
# =========================================

dht_sensor = dht.DHT11(Pin(PIN_DHT))

# =========================================
# BUZZER
# =========================================

buzzer = PWM(Pin(PIN_BUZZER))

buzzer.freq(1000)
buzzer.duty(0)

# =========================================
# BOTON PANICO
# =========================================

boton_panico = Pin(PIN_BOTON, Pin.IN, Pin.PULL_UP)

# =========================================
# MPU6050
# =========================================

i2c = I2C(0, scl=Pin(PIN_SCL), sda=Pin(PIN_SDA))

mpu = MPU6050(i2c)

# =========================================
# VARIABLES
# =========================================

temperatura_actual = 0
humedad_actual = 0

estado_movimiento = "REPOSO"

# =========================================
# UMBRALES
# =========================================

TEMP_MAX = 50
TEMP_MIN = 15

HUM_MAX = 70
HUM_MIN = 20

MOVIMIENTO_NORMAL = 1.2
MOVIMIENTO_BRUSCO = 1.8

# =========================================
# ALARMAS
# =========================================

alarma_temp_alta = False
alarma_temp_baja = False

alarma_hum_alta = False
alarma_hum_baja = False

alarma_movimiento = False
alarma_movimiento_brusco = False

alarma_panico = False

ip_address = "No conectado"

# =========================================
# BANDERAS
# =========================================

alarma_activa = False

# =========================================
# TELEGRAM
# =========================================

def send_telegram_alert(message):

    try:

        message = message.replace(" ", "%20")

        url = (
            f"https://api.telegram.org/bot{TOKEN}"
            f"/sendMessage?chat_id={CHAT_ID}&text={message}"
        )

        response = urequests.get(url)

        response.close()

        print("[Telegram] Enviado")

    except Exception as e:

        print("[Telegram] Error:", e)

# =========================================
# CONSULTAR TELEGRAM
# =========================================

def revisar_telegram():

    global temperatura_actual
    global humedad_actual
    global estado_movimiento

    try:

        url = f"https://api.telegram.org/bot{TOKEN}/getUpdates"

        response = urequests.get(url)

        data = response.json()

        response.close()

        if data["result"]:

            ultimo = data["result"][-1]

            mensaje = ultimo["message"]["text"]

            chat_id = ultimo["message"]["chat"]["id"]

            if str(chat_id) == str(CHAT_ID):

                if mensaje == "/temp":

                    send_telegram_alert(
                        "Temperatura: "
                        + str(temperatura_actual)
                        + " C"
                    )

                elif mensaje == "/hum":

                    send_telegram_alert(
                        "Humedad: "
                        + str(humedad_actual)
                        + " %"
                    )

                elif mensaje == "/mov":

                    send_telegram_alert(
                        "Movimiento: "
                        + estado_movimiento
                    )

                elif mensaje == "/estado":

                    texto = (
                        "Temp: "
                        + str(temperatura_actual)
                        + " C | Hum: "
                        + str(humedad_actual)
                        + " % | Mov: "
                        + estado_movimiento
                    )

                    send_telegram_alert(texto)

    except Exception as e:

        print("Error Telegram:", e)

# =========================================
# WIFI
# =========================================

def wifi_connect():

    global ip_address

    wlan = network.WLAN(network.STA_IF)

    wlan.active(False)
    time.sleep(2)

    wlan.active(True)
    time.sleep(3)

    if not wlan.isconnected():

        print("Conectando WiFi...")

        wlan.connect(SSID, PASSWORD)

        timeout = 20

        while not wlan.isconnected() and timeout > 0:

            print("Esperando...")
            time.sleep(1)

            timeout -= 1

    if wlan.isconnected():

        ip_address = wlan.ifconfig()[0]

        print("WiFi conectado")
        print("IP:", ip_address)

        send_telegram_alert(
            "Sistema conectado IP: " + ip_address
        )

    else:

        print("No se pudo conectar")

# =========================================
# SERVIDOR WEB
# =========================================

def iniciar_servidor_web():

    global alarma_activa

    addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]

    s = socket.socket()

    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    s.bind(addr)

    s.listen(5)

    print("Servidor web iniciado")

    while True:

        try:

            print("Esperando cliente...")

            conn, addr = s.accept()

            print("Cliente conectado:", addr)

            request = conn.recv(1024)

            request = str(request)

            # =====================================
            # CONTROL BUZZER WEB
            # =====================================

            if '/silenciar' in request:

                alarma_activa = False

                buzzer.duty(0)

                print("Alarma silenciada desde WEB")

            # =====================================
            # PAGINA HTML
            # =====================================

            html = f"""
            <!DOCTYPE html>

            <html>

            <head>

                <meta charset="UTF-8">

                <title>BioStep Monitor</title>

                <meta name="viewport"
                content="width=device-width, initial-scale=1">

                <meta http-equiv="refresh" content="3">

                <style>

                    body {{

                        font-family: Arial;
                        background-color: #f0f2f5;
                        text-align: center;
                        padding: 20px;

                    }}

                    .card {{

                        max-width: 500px;
                        margin: auto;
                        background: white;
                        padding: 20px;
                        border-radius: 10px;

                    }}

                    .box {{

                        background: #f8f9fa;
                        padding: 15px;
                        margin-top: 15px;
                        border-radius: 5px;

                    }}

                    .btn {{

                        display: inline-block;
                        padding: 12px 20px;
                        margin: 10px;
                        color: white;
                        text-decoration: none;
                        border-radius: 5px;
                        background-color: red;

                    }}

                </style>

            </head>

            <body>

                <div class="card">

                    <h1>Monitor IoT BioStep</h1>

                    <p><b>IP:</b> {ip_address}</p>

                    <div class="box">

                        <p><b>Temperatura:</b> {temperatura_actual} °C</p>

                        <p><b>Humedad:</b> {humedad_actual} %</p>

                        <p><b>Movimiento:</b> {estado_movimiento}</p>

                    </div>

                    <div class="box">

                        <p><b>Botón de pánico:</b>
                        {"ACTIVADO" if alarma_panico else "OK"}</p>

                        <p><b>Estado alarma:</b>
                        {"ACTIVA" if alarma_activa else "NORMAL"}</p>

                    </div>

                    <a href="/silenciar" class="btn">
                        SILENCIAR ALARMA
                    </a>

                </div>

            </body>

            </html>
            """

            conn.send('HTTP/1.1 200 OK\r\n')
            conn.send('Content-Type: text/html\r\n')
            conn.send('Connection: close\r\n\r\n')

            conn.sendall(html)

            conn.close()

        except Exception as e:

            print("Error servidor:", e)

# =========================================
# GESTIONAR BUZZER
# =========================================

def gestionar_buzzer():

    if alarma_activa:

        if alarma_panico:

            buzzer.freq(2000)

        elif alarma_movimiento_brusco:

            buzzer.freq(1700)

        elif alarma_movimiento:

            buzzer.freq(1300)

        elif (
            alarma_temp_alta
            or alarma_temp_baja
        ):

            buzzer.freq(800)

        elif (
            alarma_hum_alta
            or alarma_hum_baja
        ):

            buzzer.freq(500)

        buzzer.duty(512)

    else:

        buzzer.duty(0)

# =========================================
# INICIO
# =========================================

wifi_connect()

_thread.start_new_thread(iniciar_servidor_web, ())

print("Sistema iniciado")

last_sensor_check = 0

# =========================================
# LOOP PRINCIPAL
# =========================================

while True:

    try:

        now = time.ticks_ms()

        revisar_telegram()

        # =====================================
        # LEER SENSOR CADA 2 SEGUNDOS
        # =====================================

        if time.ticks_diff(now, last_sensor_check) > 2000:

            last_sensor_check = now

            # =====================================
            # DHT11
            # =====================================

            dht_sensor.measure()

            temperatura_actual = dht_sensor.temperature()

            humedad_actual = dht_sensor.humidity()

            print("--------------------")

            print("Temperatura:", temperatura_actual)

            print("Humedad:", humedad_actual)

            # =====================================
            # MPU6050
            # =====================================

            ax = abs(mpu.accel.x)
            ay = abs(mpu.accel.y)

            movimiento_total = ax + ay

            print("Movimiento:", movimiento_total)

            if movimiento_total < MOVIMIENTO_NORMAL:

                estado_movimiento = "REPOSO"

                alarma_movimiento = False
                alarma_movimiento_brusco = False

            elif movimiento_total < MOVIMIENTO_BRUSCO:

                estado_movimiento = "MOVIMIENTO"

                alarma_movimiento = True
                alarma_movimiento_brusco = False

            else:

                estado_movimiento = "MOVIMIENTO BRUSCO"

                alarma_movimiento = False
                alarma_movimiento_brusco = True

                send_telegram_alert(
                    "ALERTA MOVIMIENTO BRUSCO"
                )

            # =====================================
            # VERIFICAR LIMITES
            # =====================================

            alarma_temp_alta = temperatura_actual > TEMP_MAX
            alarma_temp_baja = temperatura_actual < TEMP_MIN

            alarma_hum_alta = humedad_actual > HUM_MAX
            alarma_hum_baja = humedad_actual < HUM_MIN

            # =====================================
            # ACTIVAR ALARMA AUTOMATICA
            # =====================================

            if (
                alarma_temp_alta
                or alarma_temp_baja
                or alarma_hum_alta
                or alarma_hum_baja
                or alarma_movimiento
                or alarma_movimiento_brusco
            ):

                alarma_activa = True

                print("ALERTA ACTIVADA")

                send_telegram_alert(
                    "ALERTA VARIABLES FUERA DE RANGO"
                )

        # =====================================
        # BOTON PANICO
        # =====================================

        if boton_panico.value() == 0:

            alarma_panico = True

            alarma_activa = True

            print("BOTON DE PANICO ACTIVADO")

            send_telegram_alert(
                "BOTON DE PANICO ACTIVADO"
            )

        else:

            alarma_panico = False

        # =====================================
        # CONTROL BUZZER
        # =====================================

        gestionar_buzzer()

        time.sleep_ms(100)

    except Exception as e:

        print("Error:", e)

        time.sleep(2)
