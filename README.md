# Postfix SPF Check

Merhaba, bu yazımızda postfix tarafından sunucunuza gelen maillerin SPF, DKIM gibi kayıtların nasıl kontrolünü yapacağını anlatacağım.


İlk önce gerekli paketi yükleyelim

```
apt-get install postfix-policyd-spf-python postfix-pcre
```

`/etc/postfix-policyd-spf-python/policyd-spf.conf` dosya yoluna gidip gerekli ayarları yapıyoruz.

Bize sunulan ayarlar aşağıdaki gibidir:

```
- **SPF_Not_Pass (default)** - Sonuç Geçerli/Hiçbiri/Geçici Hata değilse reddet.
- **Softfail** - Sonuç Softfail ve Fail ise reddet.
- **Fail** - HELO Fail ise reddet.
- **Null** - Sadece Null gönderen için HELO Fail durumunda reddet (SPF Classic).
- **False** - HELO'da asla reddetme/erteleme, sadece başlık ekle.
- **No_Check** - HELO'yu asla kontrol etme.
```

Kendi ihtiyacınıza göre ayarlardan bir tanesini seçebilirsiniz.

Ben config dosyamı aşağıdaki gibi bırakıyorum:

```
 debugLevel = 1
 defaultSeedOnly = 1

 HELO_reject = SPF_Not_Pass
 Mail_From_reject = SPF_Not_Pass

 PermError_reject = False
 TempError_Defer = False

 skip_addresses = 127.0.0.0/8,::ffff:127.0.0.0/104,::1
```

Şimdi /etc/postfix dizinindeki master.cf dosyamıza geçiyoruz ve sonuna şunu ekliyoruz:

```
policyd-spf  unix  -       n       n       -       0       spawn
     user=policyd-spf argv=/usr/bin/policyd-spf
```

Şimdi main.cf'ye geçiyoruz ve SPF kaydının kontrol edilmesinin zaman aşımını uzatmak için bir satır ekliyoruz:

```
policyd-spf_time_limit = 3600
```

Ve son olarak smtpd_recipient_restrictions ayarlarımızı yeni SPF kontrolünü hesaba katacak şekilde ayarlayın

```
smtpd_recipient_restrictions = reject_unauth_destination, check_policy_service unix:private/policyd-spf
```

Postfix servisini yeniden başlatın.

```
systemctl restart postfix
```

Mail testi yaparak kontrol sağlayabilirsiniz.

```
Jun 11 17:11:44 localhost postfix/smtpd[1406106]: connect from mail-qk1-f170.google.com[209.85.222.170]
Jun 11 17:11:45 localhost postfix/smtpd[1406106]: TLS SNI webmail.ugurcomptech.com.tr from mail-qk1-f170.google.com[209.85.222.170] not matched, using default chain
Jun 11 17:11:46 localhost policyd-spf[1406112]: WARNING: Deprecated Config Option defaultSeedOnly in use in: /etc/postfix-policyd-spf-python/policyd-spf.conf
Jun 11 17:11:46 localhost policyd-spf[1406112]: prepend Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=209.85.222.170; helo=mail-qk1-f170.google.com; envelope-from=ugurcomptech@gmail.com; receiver=<UNKNOWN> 
Jun 11 17:11:47 localhost postfix/smtpd[1406106]: F387D180617: client=mail-qk1-f170.google.com[209.85.222.170]
Jun 11 17:11:47 localhost postfix/cleanup[1406113]: F387D180617: message-id=<CAO3ShMFFh3Tt620qioRt_=3ONRC++U2zr8evNU9_EUuHcKRLew@mail.gmail.com>
Jun 11 17:11:47 localhost opendkim[619499]: F387D180617: DKIM verification successful
Jun 11 17:11:47 localhost opendkim[619499]: F387D180617: s=20230601 d=gmail.com a=rsa-sha256 SSL 
Jun 11 17:11:47 localhost postfix/qmgr[1404912]: F387D180617: from=<ugurcomptech@gmail.com>, size=3066, nrcpt=1 (queue active)
Jun 11 17:11:47 localhost postfix/smtpd[1406106]: disconnect from mail-qk1-f170.google.com[209.85.222.170] ehlo=2 starttls=1 mail=1 rcpt=1 bdat=1 quit=1 commands=7
Jun 11 17:11:47 localhost postfix/pipe[1406115]: F387D180617: to=<ugur@ugurcomptech.com.tr>, relay=dovecot, delay=1.9, delays=1.6/0.05/0/0.27, dsn=2.0.0, status=sent (delivered via dovecot service)
Jun 11 17:11:47 localhost postfix/qmgr[1404912]: F387D180617: removed
```
