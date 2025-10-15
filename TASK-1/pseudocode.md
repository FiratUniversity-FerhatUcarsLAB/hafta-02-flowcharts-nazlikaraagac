ALGORITMA ATM_PARA_CEKME

    KART_TAK()
    EGER KART_GEÇERSİZ İSE
        MESAJ("Geçersiz kart. Lütfen bankanızla iletişime geçin.")
        KART_IADESI()
        BITIR
    SON

    PIN_DENEME_HAKKI ← 3
    DOGRU_PIN ← FALSE

    TEKRAR PIN_GIR
        PIN ← KULLANICIDAN_PIN_AL()
        EGER PIN_DOGRU(PIN) ISE
            DOGRU_PIN ← TRUE
            CIK
        DEGILSE
            PIN_DENEME_HAKKI ← PIN_DENEME_HAKKI - 1
            MESAJ("Hatalı PIN. Kalan deneme hakkı: " + PIN_DENEME_HAKKI)
        SON
    PIN_DENEME_HAKKI > 0 IKEN

    EGER DOGRU_PIN = FALSE ISE
        MESAJ("Kartınız bloke oldu. Bankanızla iletişime geçin.")
        KART_IADESI()
        BITIR
    SON

    GUNLUK_LIMIT ← 5000
    BUGUN_CEKILEN ← SISTEMDEN_AL("bugün çekilen tutar")
    BAKIYE ← HESAPTAN_BAKIYE_AL()

    MESAJ("Çekmek istediğiniz tutarı giriniz (20 TL katlarında).")
    TUTAR ← KULLANICIDAN_TUTAR_AL()

    EGER TUTAR % 20 ≠ 0 ISE
        MESAJ("Tutar yalnızca 20 TL'nin katları olabilir.")
        KART_IADESI()
        BITIR
    SON

    EGER TUTAR > BAKIYE ISE
        MESAJ("Yetersiz bakiye.")
        KART_IADESI()
        BITIR
    SON

    EGER (BUGUN_CEKILEN + TUTAR) > GUNLUK_LIMIT ISE
        MESAJ("Günlük para çekme limitinizi aşıyorsunuz.")
        KART_IADESI()
        BITIR
    SON

    // Tüm kontrolleri geçtiyse
    BAKIYE ← BAKIYE - TUTAR
    BUGUN_CEKILEN ← BUGUN_CEKILEN + TUTAR
    SISTEME_YAZ("yeni bakiye", BAKIYE)
    SISTEME_YAZ("bugün çekilen tutar", BUGUN_CEKILEN)

    PARA_VER(TUTAR)
    MESAJ("İşleminiz başarıyla gerçekleşti. Yeni bakiyeniz: " + BAKIYE)

    KART_IADESI()
    BITIR

SON_ALGORITMA

