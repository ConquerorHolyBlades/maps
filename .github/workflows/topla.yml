"""
Ortak harita verisi toplayıcı (GitHub Actions'ta çalışır).

- Açık olan bütün Travian sunucularını kendiliğinden bulur (günde bir kez tarar),
  kapanan sunucuları takipten çıkarır.
- Her sunucunun map.sql dosyasını günde bir kez kaydeder:
      veri/data/<sunucu>/<YYYY-MM-DD>.sql.xz      (her sunucu kendi klasöründe)
- Gün, Türkiye saatine (UTC+3) göre belirlenir. Sunucu o günün verisini henüz
  yayınlamadıysa (dosya dünküyle aynıysa) ya da dosya boşsa kaydetmez.
- Son SAKLA_GUN gün tutulur; veri dalı her çalışmada tek commit olarak yeniden
  yazıldığı için depo büyümez.
"""
import concurrent.futures as cf
import datetime as dt
import gzip
import hashlib
import json
import lzma
import os
import re
import sys
import urllib.request

SAKLA_GUN = 8                      # 7 gün inaktiflik araması için 8 günlük kayıt yeter
TR = dt.timezone(dt.timedelta(hours=3))
KOK = os.path.dirname(os.path.abspath(__file__))
VERI = sys.argv[1] if len(sys.argv) > 1 else os.path.join(KOK, "veri")
BOLGELER = ["international", "europe", "america", "asia", "arabics"]
DUSME_SINIRI = 8                   # art arda bu kadar çalışmada açılmayan sunucu takipten çıkar


def aday_sunucular():
    """Travian'ın adlandırma düzenine göre olası sunucular + elle eklenenler."""
    adaylar = set()
    hiz_numara = {1: range(1, 13), 2: range(19, 24), 3: range(29, 36), 5: range(49, 53), 10: range(99, 104)}
    for hiz, numaralar in hiz_numara.items():
        for b in BOLGELER:
            adaylar.add(f"rog.x{hiz}.{b}.travian.com")
            for n in numaralar:
                adaylar.add(f"ts{n}.x{hiz}.{b}.travian.com")
    yol = os.path.join(KOK, "aday_sunucular.txt")
    if os.path.exists(yol):
        for satir in open(yol, encoding="utf-8"):
            s = satir.split("#")[0].strip().lower()
            if re.fullmatch(r"[a-z0-9.-]+\.travian\.com", s):
                adaylar.add(s)
    return adaylar


def indir(sunucu, zaman_asimi=60):
    istek = urllib.request.Request(f"https://{sunucu}/map.sql", headers={
        "User-Agent": "Mozilla/5.0 (ortak harita verisi)", "Accept-Encoding": "gzip"})
    with urllib.request.urlopen(istek, timeout=zaman_asimi) as y:
        veri = y.read()
    if veri[:2] == b"\x1f\x8b":
        veri = gzip.decompress(veri)
    return veri if b"INSERT INTO" in veri[:4000] else None


def dene(sunucu):
    try:
        return sunucu, indir(sunucu, 30)
    except Exception:
        return sunucu, None


def main():
    simdi = dt.datetime.now(TR)
    bugun = simdi.date().isoformat()
    os.makedirs(os.path.join(VERI, "data"), exist_ok=True)
    durum_yolu = os.path.join(VERI, "sunucular.json")
    try:
        durum = json.load(open(durum_yolu, encoding="utf-8"))
    except Exception:
        durum = {"sunucular": {}, "son_tarama": None}
    takip = durum["sunucular"]           # sunucu -> {"ilk": gün, "son": gün, "hata": sayı}

    # Günde bir kez (ya da liste boşsa) tüm olası sunucuları tara
    taranacak = set(takip)
    if durum.get("son_tarama") != bugun or not takip:
        taranacak |= aday_sunucular()
        durum["son_tarama"] = bugun
    print(f"{len(taranacak)} sunucu kontrol ediliyor...")

    with cf.ThreadPoolExecutor(max_workers=16) as havuz:
        sonuclar = list(havuz.map(dene, sorted(taranacak)))

    for sunucu, veri in sonuclar:
        k = takip.get(sunucu)
        if veri is None:
            if k:
                k["hata"] = k.get("hata", 0) + 1
                if k["hata"] >= DUSME_SINIRI:
                    print(f"{sunucu}: kapanmış görünüyor, takipten çıkarıldı")
                    del takip[sunucu]
            continue
        if not k:
            print(f"{sunucu}: yeni sunucu bulundu")
            k = takip[sunucu] = {"ilk": bugun}
        k.update(son=bugun, hata=0)
        klasor = os.path.join(VERI, "data", sunucu)
        os.makedirs(klasor, exist_ok=True)
        hedef = os.path.join(klasor, f"{bugun}.sql.xz")
        if os.path.exists(hedef):
            continue
        onceki = sorted(f for f in os.listdir(klasor) if f.endswith(".sql.xz"))
        if onceki:
            with open(os.path.join(klasor, onceki[-1]), "rb") as f:
                if hashlib.sha256(lzma.decompress(f.read())).digest() == hashlib.sha256(veri).digest():
                    print(f"{sunucu}: bugünün verisi henüz yayınlanmadı")
                    continue
        with open(hedef, "wb") as f:
            f.write(lzma.compress(veri, preset=6))
        print(f"{sunucu}: {bugun} kaydedildi")

    # eski günleri ve takipten çıkan sunucuları sil, listeyi yaz
    sinir = (simdi.date() - dt.timedelta(days=SAKLA_GUN - 1)).isoformat()
    indeks = {}
    data = os.path.join(VERI, "data")
    for sunucu in sorted(os.listdir(data)):
        klasor = os.path.join(data, sunucu)
        if sunucu not in takip:
            for f in os.listdir(klasor):
                os.remove(os.path.join(klasor, f))
            os.rmdir(klasor)
            continue
        gunler = []
        for f in sorted(os.listdir(klasor)):
            gun = f[:10]
            if gun < sinir:
                os.remove(os.path.join(klasor, f))
            else:
                gunler.append(gun)
        if gunler:
            indeks[sunucu] = gunler
    json.dump(durum, open(durum_yolu, "w", encoding="utf-8"), indent=1, sort_keys=True)
    json.dump({"guncelleme": simdi.strftime("%Y-%m-%d %H:%M"), "sunucular": indeks},
              open(os.path.join(VERI, "index.json"), "w", encoding="utf-8"), indent=1)
    print(f"Toplam {len(indeks)} sunucu, {sum(len(v) for v in indeks.values())} günlük kayıt.")


if __name__ == "__main__":
    main()
