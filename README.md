# Ortak harita verisi

Travian sunucularının günlük harita verisi (map.sql). Yağma Listesi bu veriden eksik günleri alır.

- Veriler **veri** dalında durur: `data/<sunucu>/<gün>.sql.xz` ve `index.json`.
- Toplama Türkiye saatiyle 00:00, 06:00, 12:00 ve 18:00'de kendiliğinden çalışır.
- Özel adlı bir sunucu eklemek için `aday_sunucular.txt` dosyasına satır ekle.
