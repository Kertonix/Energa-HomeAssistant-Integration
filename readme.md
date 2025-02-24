# Home Assistant - Integracja z Energa S.A [PL]

Skrypt pozwala pobrać dane z licznika dostawcy Energa S.A oraz wysłać je na serwer MQTT. Dotyczy to zarówno licznika jednokierunkowego jak i dwukierunkowego (w taryfie G11, G12 i G12W).

![HA Screenshot](https://papuutekapt.github.io/assets/Energa-HomeAssistant-Integration/ha.png)

## Ograniczenia.

Skrypt bazuje na pozyskiwaniu danych bezpośrednio ze stron internetowych (ang. Web scrapping), dlatego przy jakichkolwiek zmianach na stronie przez Energa S.A program może ulec awarii. Postaram się na bieżąco aktualizować to repetytorium.

## Instalacja

1. Tworzenie konta na [Mój Licznik](https://mojlicznik.energa-operator.pl)
2. Aktualizacja listy dostępnych pakietów:
   - `sudo apt update`
3. Instalacja wymaganych pakietów:
   - `sudo apt install python3 python3-pip git`
   - `sudo pip3 install selenium`
   - `sudo pip3 install pyvirtualdisplay`
   - `sudo pip3 install paho-mqtt`
   - `sudo apt install xvfb`
     - Dla RPI: `sudo apt install chromium-chromedriver`
     - Dla Debiana: `sudo apt install chromium-driver`
4. Instalacja skryptu:
   - `git clone https://github.com/PapuutekAPT/Energa-HomeAssistant-Integration.git Energa-script `
   - `cd Energa-script && sudo chmod +x main.py`

## Użycie

```
usage: main.py [-h] -tm {1,2} -ms MQTT_SERVER [-mu MQTT_USERNAME]
               [-mp MQTT_PASSWORD] -mt MQTT_TOPIC -eu ENERGA_USERNAME -ep
               ENERGA_PASSWORD

required arguments:
  -tm {1,2,3,4}, --type_of_meter {1,2,3,4}
                        Typ licznika: 1 dla jednokierunkowego, 2 dla dwukierunkowego, 3 dla jednokierunkowego (taryfa G12W), 4 dla dwukierunkowego (taryfa G12W)
  -ms MQTT_SERVER, --mqtt_server MQTT_SERVER
                        Serwer MQTT
                        Haslo MQTT
  -mt MQTT_TOPIC, --mqtt_topic MQTT_TOPIC
                        Temat MQTT
  -eu ENERGA_USERNAME, --energa_username ENERGA_USERNAME
                        Login Energa S.A
  -ep ENERGA_PASSWORD, --energa_password ENERGA_PASSWORD
                        Haslo Energa S.A

optional arguments:
  -h, --help            show this help message and exit
  -mu MQTT_USERNAME, --mqtt_username MQTT_USERNAME
                        Uzytkownik MQTT
  -mp MQTT_PASSWORD, --mqtt_password MQTT_PASSWORD
                        Haslo MQTT
  -mo MQTT_PORT, --mqtt_port MQTT_PORT
                        Port MQTT
przykład: ./main.py -tm 1 -ms 192.168.19.129 -mt home/energa -eu email@gmail.com -ep haslo_energa -mu login_mqtt -mp hasło_mqtt
```

#### Wiadomość na serwerze MQTT po uruchomieniu skryptu:

![MQTT Screenshot](https://papuutekapt.github.io/assets/Energa-HomeAssistant-Integration/mqtt.png)

## Konfiguracja Home Assistant'a

W pliku configuration.yaml dodajemy (dla taryfy G12 i G12W - typ licznika 3 lub 4):

```
mqtt:
 sensor:
  - name: "Energia pobrana dzienna"
    state_topic: "home/energa" 
    unit_of_measurement: "kWh"
    device_class: energy
    state_class: total_increasing
    value_template: "{{ value_json.used_1 }}"
  - name: "Energia pobrana nocna"
    state_topic: "home/energa" 
    unit_of_measurement: "kWh"
    device_class: energy
    state_class: total_increasing
    value_template: "{{ value_json.used_2 }}"
  - name: "Energia oddana dzienna" 
    state_topic: "home/energa"
    unit_of_measurement: "kWh" 
    device_class: energy
    state_class: total_increasing
    value_template: "{{ value_json.produced_1 }}"    
  - name: "Energia oddana nocna" 
    state_topic: "home/energa"
    unit_of_measurement: "kWh" 
    device_class: energy
    state_class: total_increasing
    value_template: "{{ value_json.produced_2 }}"
```

## Automatyczne uruchanianie skryptu

Energa S.A pobiera dane z liczników w nocy. Niestety z doświadczenia wiem, że nie jest to stała godzina. Proponuję uruchaniać skrypt codziennie o 5 rano.
W tym celu dodajemy wpis do cron'a:

- `crontab -e`

```
0 5 * * * python3 /home/pi/Energa-script/main.py -tm 1 -ms 192.168.19.129 -mt home/energa -eu email@gmail.com -ep haslo_energa -mu login_mqtt -mp hasło_mqtt
```

Jeśli nie masz zainstalowanego crona, możesz to zrobić to komendą: `sudo apt install cron`

## Automatyczne uruchamianie - Docker

Istnieje możliwośc cyklicznego uruchamiania skryptu z wykorzystaniem kontenera Docker.

```bash
docker run -d --name energa-scrapper \
-e CRON="0 5 * * *" \
-e ENERGA_PASSWORD=<password> \
-e ENERGA_USERNAME=<username> \
-e TYPE=<type: 1|2|3|4> \
-e MQTT_SERVER=<server> \
-e MQTT_TOPIC=<topic> \
-e MQTT_USERNAME=<optional mqtt username> \
-e MQTT_PASSWORD=<optional mqtt password> \
-e MQTT_PORT=<optional mqtt server port> \
kertonix/energa-scrapper
```
## Automatyczne uruchamianie - Docker Compose
```
version: "3"

services:
  energa-scrapper:
    container_name: kertonix/energa-scrapper
    image: energa-scrapper:1.0
    network_mode: "host"
    environment:
        CRON: "0 */2 * * *" 
        ENERGA_PASSWORD: "<yourPassword>" 
        ENERGA_USERNAME: "<yourEmail>" 
        TYPE: 4 
        MQTT_SERVER: "<brokerIP>" 
        MQTT_TOPIC: "home/energa"
    restart: unless-stopped
```

Zmienna `CRON` przyjmuje standardowy format definiowania harmonogramu `cron`:

```
*    *    *   *    *  Command_to_execute
|    |    |    |   |
|    |    |    |    Day of the Week ( 0 - 6 ) ( Sunday = 0 )
|    |    |    |
|    |    |    Month ( 1 - 12 )
|    |    |
|    |    Day of Month ( 1 - 31 )
|    |
|    Hour ( 0 - 23 )
|
Min ( 0 - 59 )
```

Przykład:

- `-e CRON="0 5 * * *"` skrypt będzie uruchamiany każdego dnia o 5:00
- `-e CRON="10 14 * * *` skrypt bedzie uruchamiany każdego dnia o 14:10
- `-e CRON="0 */2 * * *` skrypt bedzie uruchamiany każdego dnia co dwie godziny

## Testowane na:

- Raspbian 10 Buster na Raspberry PI 3B+,
- Raspbian 10 Buster na Raspberry PI 4B,
- Debian 10.
- Powinno działać na każdej dystrybucji bazującej na debianie.
- W oparciu o obraz docker powinno działac na każdej maszynie z zainstalowanym Docker'em (również Windows)

# Home Assistant - Energa S.A Integration [EN]

Script allows to get data from polish energy distributor: Energa S.A

![HA Screenshot](https://papuutekapt.github.io/assets/Energa-HomeAssistant-Integration/ha.png)
