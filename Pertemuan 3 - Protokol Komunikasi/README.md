Nama: Aditia Wahyu Nugraha
NIM: H1H024014
Shift Awal: C
Shift Akhir: A

---

# Modul 3: Komunikasi Data HTTP dan MQTT pada ESP8266

Modul ini mempertemukan dua protokol yang sering dipakai di proyek IoT, yaitu HTTP dan MQTT, dengan JSON sebagai format datanya (dirakit lewat pustaka ArduinoJson). Percobaan pertama mengirim data suhu dan kelembaban dari board ke server memakai HTTP POST. Percobaan kedua mempublikasikan data serupa ke broker MQTT publik dengan pola publish-subscribe.

Modul aslinya ditulis untuk ESP32 (`WiFi.h` dan `HTTPClient.h`), sedangkan board yang saya pakai adalah ESP8266 NodeMCU, jadi pustakanya saya ganti ke `ESP8266WiFi.h` dan `ESP8266HTTPClient.h`. Endpoint httpbin.org memakai HTTPS; itu sebabnya saya menambahkan `WiFiClientSecure.h` dan memanggil `setInsecure()` supaya board tetap bisa tersambung tanpa memeriksa sertifikat SSL (cukup untuk praktikum, tidak untuk produk sungguhan). Percobaan MQTT nyaris tidak perlu perubahan, karena PubSubClient bekerja sama di kedua board.

## Alat dan Bahan

- Board ESP8266 (NodeMCU), 1 buah
- Kabel Micro-USB
- Laptop dengan Arduino IDE yang board manager ESP8266-nya sudah terpasang
- Jaringan WiFi dengan akses internet
- Aplikasi client MQTT (HiveMQ WebSocket Client) untuk memeriksa data yang masuk ke broker
- Broker publik `broker.hivemq.com` port 1883
- Endpoint uji `httpbin.org/post`

## Library

- `ESP8266WiFi.h`: bawaan board manager ESP8266, menyambungkan board ke WiFi sebagai klien.
- `WiFiClientSecure.h`: membangun koneksi HTTPS ke httpbin.org.
- `ESP8266HTTPClient.h`: mengirim request HTTP POST di percobaan 3A.
- PubSubClient (Nick O'Leary), dipasang lewat Library Manager: dasar komunikasi MQTT di percobaan 3B.
- ArduinoJson (Benoit Blanchon), juga lewat Library Manager: menyusun data sensor menjadi JSON di kedua percobaan.

---

# Percobaan 3A: Komunikasi Data Menggunakan HTTP

## Tujuan

Mengirim data suhu dan kelembaban dari ESP8266 ke server lewat HTTP POST dalam format JSON, lalu menambahkan data waktu dari `millis()` ke dalam JSON tersebut.

## Rangkaian

Tidak ada rangkaian tambahan. Nilai suhu dan kelembaban masih berupa angka contoh, bukan hasil baca sensor, jadi board cukup disambungkan ke laptop lewat USB untuk mengunggah program dan memantau Serial Monitor.

## Kode Program

```cpp
#include <ESP8266WiFi.h>
#include <WiFiClientSecure.h>
#include <ESP8266HTTPClient.h>
#include <ArduinoJson.h>

const char* ssid = "vivo Y17s";
const char* password = "Aditiaaaa";
const char* serverUrl = "https://httpbin.org/post";

// Client untuk koneksi HTTPS
WiFiClientSecure clientInsecure;

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);

  Serial.print("Menghubungkan ke WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi berhasil terhubung!");

  // Tidak melakukan verifikasi sertifikat SSL
  clientInsecure.setInsecure();
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {

    HTTPClient http;

    // Koneksi HTTPS menggunakan WiFiClientSecure
    http.begin(clientInsecure, serverUrl);
    http.addHeader("Content-Type", "application/json");

    // Membuat objek data sensor dalam format JSON
    JsonDocument doc;
    doc["suhu"] = 28.5;
    doc["kelembaban"] = 65.0;

    // Menambahkan data waktu sejak ESP8266 dinyalakan
    doc["waktu"] = millis();

    String requestBody;
    serializeJson(doc, requestBody);

    Serial.print("Mengirim data: ");
    Serial.println(requestBody);

    // Mengirim data melalui HTTP POST
    int httpResponseCode = http.POST(requestBody);

    if (httpResponseCode > 0) {
      Serial.print("Kode Response HTTP: ");
      Serial.println(httpResponseCode);
      Serial.println("Isi Response:");
      Serial.println(http.getString());
    } else {
      Serial.print("Pengiriman gagal, kode error: ");
      Serial.println(httpResponseCode);
    }

    http.end();
  }

  delay(10000); // kirim data setiap 10 detik
}
```

## Penjelasan Kode

Empat baris `#include` di bagian atas memanggil pustaka untuk WiFi, HTTPS, HTTP client, dan JSON. `ssid` dan `password` menyimpan identitas jaringan WiFi, sementara `serverUrl` berisi alamat tujuan kiriman, yaitu endpoint uji `httpbin.org/post`.

Objek `clientInsecure` (tipe `WiFiClientSecure`) sengaja saya letakkan di luar semua fungsi. Dengan begitu objek yang sama bisa dipakai lagi di tiap putaran `loop()` tanpa dibuat ulang, dan pengaturan `setInsecure()` yang cuma dipanggil sekali di `setup()` tetap berlaku.

### Percabangan dan perulangan

`while (WiFi.status() != WL_CONNECTED)` di `setup()` berjalan selama WiFi belum tersambung; tiap putaran menunggu 500 milidetik lalu mencetak satu titik, dan berhenti begitu statusnya berubah menjadi `WL_CONNECTED`.

Di `loop()`, `if (WiFi.status() == WL_CONNECTED)` menjaga agar pengiriman hanya jalan saat board masih online. Kalau WiFi putus, seluruh blok itu dilompati pada putaran tersebut.

Di dalam blok pengiriman ada `if (httpResponseCode > 0)`. Kondisi benar berarti request terkirim dan server membalas, sehingga program mencetak kode response beserta isinya. Kondisi salah menandakan pengiriman gagal (misalnya timeout), dan cabang `else` mencetak pesan gagal bersama kode errornya.

## Hasil Serial Monitor Percobaan 3A

Log di bawah ini dari run saya sendiri, diambil sebelum baris `waktu` saya tambahkan. Alamat IP pada `origin` saya sensor.

```text
13:54:22.693 -> .............................................................................................
13:55:45.325 -> WiFi berhasil terhubung!
13:55:45.361 -> Mengirim data: {"suhu":28.5,"kelembaban":65}
13:55:48.547 -> Kode Response HTTP: 200
13:55:48.547 -> Isi Response:
13:55:48.547 -> {
13:55:48.547 ->   "args": {},
13:55:48.579 ->   "data": "{\"suhu\":28.5,\"kelembaban\":65}",
13:55:48.579 ->   "files": {},
13:55:48.579 ->   "form": {},
13:55:48.579 ->   "headers": {
13:55:48.579 ->     "Accept-Encoding": "identity;q=1,chunked;q=0.1,*;q=0",
13:55:48.579 ->     "Content-Length": "29",
13:55:48.579 ->     "Content-Type": "application/json",
13:55:48.579 ->     "Host": "httpbin.org",
13:55:48.579 ->     "User-Agent": "ESP8266HTTPClient",
13:55:48.579 ->     "X-Amzn-Trace-Id": "Root=1-6aa8ebf7-26a486f412bb03ae67df6329"
13:55:48.579 ->   },
13:55:48.579 ->   "json": {
13:55:48.623 ->     "kelembaban": 65,
13:55:48.623 ->     "suhu": 28.5
13:55:48.623 ->   },
13:55:48.623 ->   "origin": "xxx.xxx.xxx.xxx",
13:55:48.623 ->   "url": "https://httpbin.org/post"
13:55:48.623 -> }
13:55:58.584 -> Mengirim data: {"suhu":28.5,"kelemb...
```

Yang bisa dibaca dari log ini: koneksi WiFi butuh sekitar 83 detik (titik pertama 13:54:22, tersambung 13:55:45); satu siklus kirim sampai response makan sekitar 3,2 detik; kiriman berikutnya mulai 13:55:58, sesuai `delay(10000)`; dan `65.0` di kode tercetak sebagai `65` karena ArduinoJson membuang nol di belakang koma.

> Tempel tangkapan layar Serial Monitor setelah `waktu` ditambahkan di sini.

<img width="1120" height="700" alt="percobaan3A" src="https://github.com/user-attachments/assets/65810d6b-5a79-47e4-aa22-81bc9eb96163" />


# Percobaan 3B: Komunikasi Data Menggunakan MQTT

## Tujuan

Mempublikasikan data JSON dari ESP8266 ke broker MQTT publik (`broker.hivemq.com`) dengan pola publish-subscribe, lalu memastikan datanya sampai lewat client MQTT yang subscribe ke topic yang sama.

## Rangkaian

Sama seperti 3A: hanya board ESP8266 yang tersambung ke laptop lewat USB.

## Kode Program

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <WiFiClientSecure.h> // Tambahan wajib untuk HTTPS di ESP8266
#include <ArduinoJson.h>

const char* ssid = "vivo Y17s";
const char* password = "Aditiaaaa";
const char* serverUrl = "https://httpbin.org/post"; // endpoint uji HTTP POST

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  Serial.print("Menghubungkan ke WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.println("WiFi berhasil terhubung!");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    WiFiClientSecure client; 
    client.setInsecure(); // Mengabaikan verifikasi sertifikat SSL agar praktis

    HTTPClient http;
    // ESP8266 butuh argumen 'client' disisipkan sebelum serverUrl
    http.begin(client, serverUrl); 
    http.addHeader("Content-Type", "application/json");
    
    // Membuat objek data sensor dalam format JSON
    JsonDocument doc;
    doc["suhu"] = 28.5;       // contoh data suhu (°C)
    doc["kelembaban"] = 65.0; // contoh data kelembaban (%)
    String requestBody;
    serializeJson(doc, requestBody);
    
    Serial.print("Mengirim data: ");
    Serial.println(requestBody);
    
    // Mengirim data melalui HTTP POST
    int httpResponseCode = http.POST(requestBody);
    if (httpResponseCode > 0) {
      Serial.print("Kode Response HTTP: ");
      Serial.println(httpResponseCode);
      Serial.println("Isi Response:");
      Serial.println(http.getString());
    } else {
      Serial.print("Pengiriman gagal, kode error: ");
      Serial.println(httpResponseCode);
    }
    http.end();
  }
  delay(10000); // kirim data setiap 10 detik
}
```

## Penjelasan Kode

Include `ESP8266WiFi.h`, `WiFiClient.h`, `PubSubClient.h`, dan `ArduinoJson.h` menyediakan WiFi, koneksi TCP biasa, client MQTT, dan JSON. `mqttServer` serta `mqttPort` menyimpan alamat broker dan port bawaan MQTT (1883). `mqttTopic` adalah topic tempat data dititipkan; saya beri nama unik dengan menyertakan nama kelompok supaya tidak bentrok dengan kelompok lain di broker publik yang sama.

Objek `espClient` (tipe `WiFiClient`) menyediakan jalur TCP dasar, dan `client` (tipe `PubSubClient`) dibangun di atasnya untuk menjalankan protokol MQTT.

### Fungsi yang dipakai

| Fungsi | Peran |
|---|---|
| `hubungkanWiFi()` | Fungsi buatan sendiri: memanggil `WiFi.begin` lalu menunggu sampai `WL_CONNECTED`, sama seperti di 3A. |
| `hubungkanMQTT()` | Fungsi buatan sendiri untuk menyambung ke broker. `clientId` disusun dari awalan `ESP32Client-` plus `random(0xffff)`, supaya tiap perangkat di broker publik punya identitas berbeda, lalu `client.connect(clientId.c_str())` dipanggil. |
| `client.state()` | Mengembalikan kode alasan (rc) saat `client.connect` gagal; berguna untuk mencari penyebabnya. |
| `client.setServer(mqttServer, mqttPort)` | Menentukan alamat dan port broker; dipanggil sekali di `setup()` setelah WiFi tersambung. |
| `client.connected()` | Bernilai `true` selama client masih terhubung ke broker. |
| `client.loop()` | Dipanggil di setiap putaran `loop()` agar pesan masuk-keluar diproses dan koneksi tetap hidup lewat keep-alive. |
| `serializeJson(doc, buffer)` | Mengubah `JsonDocument` menjadi teks JSON di dalam `buffer` (char array). |
| `client.publish(mqttTopic, buffer)` | Mengirim isi `buffer` ke topic yang sudah ditentukan. |
| `delay(5000)` | Jeda 5 detik antar publish. |

### Percabangan dan perulangan

`while (WiFi.status() != WL_CONNECTED)` di dalam `hubungkanWiFi()` bekerja sama persis dengan versi 3A: menunggu sampai WiFi tersambung.

`while (!client.connected())` di dalam `hubungkanMQTT()` terus mencoba menyambung ke broker sampai berhasil. Percobaannya diperiksa dengan `if (client.connect(clientId.c_str()))`: kalau berhasil, program mencetak pesan sukses dan keluar dari perulangan; kalau gagal, program mencetak kode error (rc) dan pesan bahwa ia akan mencoba lagi, lalu menunggu 2 detik.

Di `loop()`, `if (!client.connected())` memeriksa apakah koneksi ke broker masih hidup. Kalau ternyata putus, `hubungkanMQTT()` dipanggil lagi untuk menyambung ulang sebelum proses publish dilanjutkan.

> Tempel tangkapan layar Serial Monitor dan aplikasi client MQTT untuk Percobaan 3B di sini.

<img width="1120" height="700" alt="percobaan3B" src="https://github.com/user-attachments/assets/253a5874-16cc-4ae6-9117-8f98472fa551" />


# Pertanyaan Praktikum - Percobaan 3A

## 1. Gambarkan diagram alur (flowchart) proses pengiriman data melalui HTTP POST pada program di atas!
 

Flowchart ini mengikuti program final: ESP8266 menyambung ke WiFi dulu, mengatur client HTTPS, lalu masuk ke `loop()` untuk merakit JSON (termasuk `waktu`) dan mengirimnya ke httpbin.org, kemudian diam 10 detik sebelum mengulang.


2.	Apa fungsi dari perintah http.addHeader("Content-Type", "application/json") pada program tersebut?
Jawab: Perintah tersebut digunakan untuk menambahkan header HTTP yang memberi tahu server bahwa data (body) yang dikirimkan berformat JSON. Header Content-Type ini penting agar server dapat menginterpretasikan dan memproses isi request dengan benar sebagai data JSON, bukan sebagai teks biasa atau format lain seperti form-urlencoded.]
3.	Jelaskan arti dari kode response HTTP 200 dan sebutkan salah satu contoh kode response HTTP lain beserta artinya!
Jawab: Kode response HTTP 200 (OK) menandakan bahwa permintaan (request) yang dikirimkan oleh klien telah berhasil diproses oleh server tanpa terjadi kesalahan. Contoh kode response lain adalah HTTP 404 (Not Found), yang berarti server tidak dapat menemukan resource atau endpoint yang diminta oleh klien, misalnya karena URL yang salah atau endpoint tersebut sudah tidak tersedia.]
4.	Modifikasi program agar ESP32 dapat mengirimkan data tambahan berupa waktu (dalam milidetik sejak dinyalakan menggunakan millis()) ke dalam JSON yang dikirim, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md!
Jawab: Lampirkan source code hasil modifikasi (menambahkan baris doc["waktu"] = millis(); sebelum proses serializeJson()) beserta file README.md penjelasannya pada repository GitHub sesuai format yang diminta modul.]
B. Pertanyaan Praktikum Percobaan 3B (MQTT)
5.	Apa fungsi dari topic pada protokol MQTT, dan mengapa topic yang digunakan perlu dibuat unik?
Jawab: Topic pada protokol MQTT berfungsi sebagai alamat atau saluran logis yang digunakan untuk mengelompokkan dan merutekan pesan antara publisher dan subscriber, sehingga perangkat hanya menerima data dari topic yang relevan baginya. Topic perlu dibuat unik (misalnya menyertakan nama kelompok) karena broker MQTT publik seperti broker.hivemq.com dapat diakses oleh siapa saja, sehingga tanpa penamaan yang unik, data dari berbagai kelompok atau pengguna lain dapat tercampur atau saling menimpa pada topic yang sama.]
6.	Jelaskan fungsi dari perintah client.loop() yang dipanggil pada setiap iterasi loop()!
Jawab: Perintah client.loop() digunakan untuk menjaga agar koneksi MQTT tetap berjalan (maintain), termasuk memproses pesan masuk (apabila ESP32 juga bertindak sebagai subscriber), mengirimkan sinyal keep-alive ke broker agar koneksi tidak dianggap terputus, serta menangani proses internal lain dari pustaka PubSubClient. Fungsi ini perlu dipanggil secara berkala pada setiap iterasi loop() agar komunikasi dengan broker tetap stabil dan responsif.
7.	Apa yang akan terjadi apabila koneksi ke broker MQTT terputus di tengah program berjalan?
Jawab: Apabila koneksi ke broker MQTT terputus, fungsi client.connected() akan mengembalikan nilai false, sehingga pada iterasi loop() berikutnya program akan mendeteksi kondisi tersebut dan memanggil kembali fungsi hubungkanMQTT() untuk mencoba menyambungkan ulang (reconnect) ke broker secara otomatis. Selama proses reconnect berlangsung, data sensor tidak dapat dipublikasikan hingga koneksi berhasil dipulihkan kembali.
C. Pertanyaan Analisis
8.	Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!
Jawab: Rangkum hasil Percobaan 3A (keberhasilan ESP32 mengirimkan data JSON ke server melalui HTTP POST dan menerima response dari httpbin.org) dan Percobaan 3B (keberhasilan ESP32 mempublikasikan data JSON ke broker MQTT dan terverifikasi melalui aplikasi client MQTT) berdasarkan data pengamatan pada Tabel 1 dan Tabel 2.
9.	Bandingkan besar overhead data dan pola komunikasi antara protokol HTTP dan MQTT berdasarkan hasil percobaan yang telah dilakukan!
Jawab: Berdasarkan hasil percobaan, setiap request HTTP POST memuat header tambahan (seperti Content-Type, Host, User-Agent, dan lainnya) yang menyebabkan ukuran data yang dikirimkan menjadi lebih besar dibandingkan payload JSON itu sendiri, serta setiap request memerlukan pembentukan koneksi baru (request-response). Sebaliknya, MQTT hanya memerlukan satu kali proses koneksi (handshake) ke broker yang kemudian tetap terjaga (persistent connection), sehingga setiap pengiriman data berikutnya hanya berupa payload singkat tanpa overhead header HTTP, menjadikan MQTT lebih ringan dan efisien untuk pengiriman data berulang.
10.	Untuk skenario pengiriman data sensor secara terus-menerus setiap beberapa detik dalam jangka waktu lama, protokol manakah (HTTP atau MQTT) yang lebih sesuai digunakan? Jelaskan alasannya!
Jawab: Untuk skenario tersebut, protokol MQTT lebih sesuai digunakan karena koneksinya bersifat persisten sehingga tidak perlu membangun ulang koneksi pada setiap pengiriman data, menghasilkan overhead komunikasi yang jauh lebih kecil dan konsumsi daya yang lebih hemat dibandingkan HTTP yang harus membentuk koneksi baru pada setiap request. Karakteristik ini menjadikan MQTT lebih cocok untuk aplikasi IoT dengan pengiriman data kontinu dalam jangka waktu lama, seperti pemantauan sensor secara real-time.
11.	Bagaimana peran format JSON dalam mendukung interoperabilitas data antara perangkat IoT dan berbagai platform/aplikasi yang berbeda?
Jawab: JSON berperan sebagai format pertukaran data standar yang bersifat ringan, terstruktur, dan mudah dibaca baik oleh manusia maupun mesin, serta didukung secara luas oleh hampir seluruh bahasa pemrograman dan platform, termasuk web, mobile, maupun cloud. Dengan menggunakan JSON, data yang dikirimkan oleh perangkat IoT seperti ESP32 dapat dengan mudah diproses, disimpan, atau ditampilkan oleh berbagai aplikasi dan layanan pihak ketiga tanpa memerlukan format khusus atau konversi tambahan, sehingga meningkatkan interoperabilitas antar sistem yang berbeda.
3.5. Kesimpulan
•	ESP32 berhasil mengimplementasikan komunikasi data menggunakan protokol HTTP dengan metode POST, ditandai dengan diterimanya kode response dan isi balasan (echo) dari server httpbin.org.
•	ESP32 berhasil mengimplementasikan komunikasi data menggunakan protokol MQTT dengan pola publish-subscribe, dengan data yang dipublikasikan berhasil diverifikasi melalui aplikasi client MQTT.
•	Format JSON memudahkan proses pembuatan, pengiriman, dan interpretasi data sensor pada kedua protokol komunikasi yang digunakan.
•	Protokol MQTT memiliki overhead komunikasi yang lebih kecil dibandingkan HTTP, sehingga lebih efisien untuk pengiriman data secara kontinu, sedangkan HTTP lebih sesuai untuk komunikasi yang bersifat periodik dan tidak memerlukan koneksi yang selalu terbuka.

