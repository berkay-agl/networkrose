---
title: "Telegram Bot API'ye Giriş - #1"
date: 2026-09-23T02:00:00+03:00
draft: false
tags: ["Telegram", "Python", "Telegram-Bot-API"]
---

Telegram Bot API konularını öğrenirken bu konuda Türkçe kaynak eksikliğini fark ettim. Ben ilk olarak Telegram'ın resmi dökümanındaki şu rehberi kullanmıştım: [core.telegram.org/bots/tutorial](https://core.telegram.org/bots/tutorial). Buradan esinlenerek ve asıl bu kaynağa bağlı kalarak Python üzerinden kendimce bir tutorial notu oluşturmak istedim.

Telegram'ın hazırladığı orijinal kaynakta Java kullanılıyordu; ben de bunu Python'a uyarlamak istedim. Zaten Java, Go, C# fark etmeksizin orijinal kaynaktaki mantık genel mimariyi anlattığı için aslında her programlama diline uygun.

## Telegram Bot API Nedir?

Telegram Bot API, geliştirdiğimiz bot uygulamalarının Telegram sunucuları ile HTTP üzerinden haberleşmesini sağlayan bir arayüzdür. Temel mantığı şöyle: biz botumuzdan Telegram Bot API'ye bir HTTP Request gönderdiğimizde, Bot API bize JSON formatında bir HTTP Response döndürür.

Ayrıca bunun için bir tane diagram oluşturttum:

![Telegram Bot API akış diyagramı](/networkrose/images/telegram-bot-api-diagram.jpeg)

Şimdi API'yi basitçe test etmeye başlayalım, hâlihazırda zaten web browser HTTP Request atabiliyor bu yüzden onu kullanabiliriz. Bunun için ihtiyacımız olan şey zaten Telegram'da **@BotFather** aratarak onu başlatmak ve şu adımları uyguluyoruz:

`/newbot` yazıp yolluyoruz, daha sonra bize *"Peki o zaman, yeni bir bot. Adını ne koyalım? Lütfen botunuz için bir isim seçin."* diyor — tabii bu normalde İngilizce, ben Türkçe olarak buraya yazıyorum.

Bot ismi belirledikten sonra, *"Güzel. Şimdi botun için bir kullanıcı adı seçelim. Adın sonu bot ile bitmelidir. Mesela şöyle: TetrisBot veya tetris_bot."* şeklinde yanıt veriyor. Örneğin botun ismine "Test" dediysen bu kısma da "TestBot" dersin.

Bu kısımdan sonra botun başarıyla oluşturulduğunu söyler ve şu kısımda *"Use this token to access the HTTP API:"* diyerek API ile botumuzun iletişime geçebilmesi için gerekli olan token'ı verir. Bu token sayesinde botumuz doğrulanacak.

Şu şekilde tarayıcıdan ziyaret ederek istek atabiliyorsun:

```
https://api.telegram.org/bot8846xxx/getMe
```

Bot API'nin temel bilgilerine göz atabiliyoruz bu şekilde.

Ama şöyle bir şey var: daha büyük projelerde bunu böyle kullanmak o kadar iyi olmaz çünkü pratik değil ve iyi şekilde ölçeklenmez.

## Framework Kullanmak

Bu yüzden zaten libraries ve framework'leri kullanacağız. Bunların sayesinde sağlam ve ölçeklenebilir şekilde API ile uğraşabiliyoruz.

Framework'ler bizim için alt seviye işlemleri hallediyor, mesela API calls kısmını, yani Telegram sunucularına istek gönderme işlemini yapıyor.

Bu yüzden bota daha iyi odaklanabiliriz, yani asıl işlevlerine odaklanırız. Ben Python için **python-telegram-bot** framework'ünü kullanıyorum. Telegram zaten alternatif bir sürü listelemiş: [core.telegram.org/bots/samples](https://core.telegram.org/bots/samples)

## Proje Klasör Yapısı

Telegram Bot API öğrenirken hâlihazırda bir klasör yapısı önemli. Ben şu şekilde oluşturdum, `BotTutorial` isminde bir klasörüm var, yapısı şöyle:

```
├── .env                 # Token bilgisini burada tutuyorum
├── requirements.txt     # Gerekli libraries listesi burada
├── bot.py               # Botun tüm mantığını burada deniyorum
└── main.py              # Burası da giriş noktası programın
```

`requirements.txt` içinde kullandığım dependencies şu şekilde:

```
python-telegram-bot==22.8
python-dotenv==1.2.3
```

Environment variable kısmında, yani `.env` içinde ise token'ı şöyle tutuyorum:

```
BOT_TOKEN=86575xxx
```

> Edit: BotFather ile detaylı bot oluşturma — [core.telegram.org/bots/features#botfather](https://core.telegram.org/bots/features#botfather)

## Diyagramı Adım Adım Açıklamak

1. **Adım:** Kullanıcı, Telegram uygulaması üzerinden mesajı yazar ve gönder butonuna basarak süreci başlatır.
2. **Adım:** Telegram sunucuları kullanıcıdan gelen bu mesajı karşılar ve işlenmesi için doğrudan Telegram Bot API tarafına iletir.
3. **Adım:** Bot API, gelen mesajı **Update Object** adı verilen ve mesaj ID'si, gönderen bilgisi, metin içeriği gibi tüm detayları barındıran bir JSON formatına dönüştürür. Ardından bu veriyi bizim Python kodumuza ulaştırır. Burada veri aktarımı iki farklı yöntemle yapılıyor:
   - **Long Polling:** Python kodumuzun belirli aralıklarla Telegram API'ye gidip yeni mesaj var mı diye sorması yöntemidir.
   - **Webhook:** Telegram API'nin yeni mesaj geldiği anda bunu bizim sunucumuza anlık HTTP POST isteği olarak bildirmesi yöntemidir.
4. **Adım:** Python kodumuz gelen JSON veri paketini parse ediyor, gerekli mantıksal işlemleri çalıştırır ve vereceği cevabı hazırlar. Cevabı iletmek için Telegram Bot API'sine `sendMessage` metodunu kullanarak bir HTTP Request atar.
5. **Adım:** Bot API, gönderdiğimiz mesaj isteğini alır ve işlemin başarılı olduğunu doğrulayan, içinde mesajın iletildiği bilgisini barındıran bir JSON Response ile yanıt vererek Python kodumuza onay döner.
6. **Adım:** İşlemi doğrulayan Bot API, hazırlanan yanıt içeriğini kullanıcılara dağıtım yapacak olan main Telegram sunucularına iletir.
7. **Adım:** Telegram sunucuları gelen bu cevabı anlık olarak kullanıcının Telegram uygulamasına basar ve kullanıcı sohbet ekranında botun yanıtını görmüş olur.

Ben anlamaya çok takıntılı biriyim... Ayrıca yetmezse diye yapay zekadan bunun bir betimlemesini istedim, gayet faydalı bence:

- **Kullanıcı siparişi verir (Adım 1):** Müşteri menüden yemeği seçer (bunu Telegram botunun menüsü gibi düşünüyorum) ve masadaki çağrı butonuna basar. (Kullanıcı mesajı yazar ve gönderir.)
- **Garson siparişi alır (Adım 2-3):** Müşteri doğrudan mutfağa bağırıp sipariş veremez. Garson (Telegram Sunucuları ve API) masaya gelir, siparişi bir fişe yazar (Update Object / JSON). Bu fişi alır ve mutfağa, yani bizim önümüze getirir.
- **Aşçı yemeği pişirir (Adım 4):** Fiş bizim önümüze (Python koduna) en son gelir çünkü siparişi hazırlayacak olan biziz. Fişi okuruz, yemeği pişiririz ve garsona "Sipariş hazır, masaya götür" deriz (`sendMessage` isteği).
- **Garson yemeği teslim eder (Adım 5-6-7):** Garson yemeği bizden alır, mutfaktan çıkar ve masadaki müşteriye sunar.

Bu betimlemeli örnek ile daha iyi oldu bence. Python kodumuz diyagramın en sonunda çünkü mesajın yolculuğu kullanıcıda başlayıp Telegram'ın ağından geçerek en son bizim kodumuza ulaşmak zorunda ki cevabı işleyip geri gönderebilelim.
