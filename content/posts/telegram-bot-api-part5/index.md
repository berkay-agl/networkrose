---
title: "Telegram Bot API'ye Giriş - #5"
date: 2026-10-01T22:17:00+03:00
draft: false
tags: ["Telegram", "Telegram-Bot-API", "Python"]
---

Şimdi botumuzu daha akıllı hâle nasıl getiririz bunu göstereceğim. Burada **@BotFather** ile biraz daha haşır neşir olmamız gerekiyor.

Echo Bot gibi bir programda kullanıcı botumuzun davranışını değiştiremiyordu ya; işte botun davranışını değiştirmek isterse ona bu imkânı sağlamak için commands ile ilgilenmemiz gerekiyor. Hani Telegram'daki şu `/ornek` gibi...

## BotFather ile Komutları Tanımlamak

Şimdi Telegram'da **@BotFather**'ı buluyoruz, `/mybots` diyerek üzerinde çalışacağımız botumuzu seçiyoruz bu kısımda zaten BotFather botları listeler. Daha sonra Edit Bot seçeneğine, ardından Edit Commands seçeneğine tıklıyoruz.

Komutların şu şekilde bir yazım kuralı var:

```text
command1 - Açıklama
command2 - Başka Açıklama

```

Mesela benim aklıma şu şekilde iki tane komut geldi: `upper` ve `lower`. Kullanıcı `/upper` modunu seçtiğinde bundan sonra yazacağı mesajlar büyük harfe dönsün örneğin "test" yazarsa "TEST" çıktısı alsın, `/lower` modunu seçtiğinde ise yazdığı mesajlar küçük harfe dönsün "TEST" yazarsa "test" olsun.

BotFather'a şu şekilde komutlarımızı ekliyoruz:

```text
upper - Mesajları büyük harfe çevirme moduna geçer.
lower - Mesajları küçük harfe çevirme moduna geçer.

```

Bu şekilde BotFather'a gerekli komutları kaydettik; artık kullanıcı sohbette `/` işaretine bastığında bu komutlar otomatik bir menü olarak önerilecek.

Şimdi kod kısmına geçmeye başlıyorum.

## Komut ve Durum Yönetimi Kurulumu

Öncelikle `__init__` kısmına yönelik eklemeler yapmalıyız:

```python
        self.user_modes = {}

```

Bu `self.user_modes = {}` dictionary şu işe yarıyor: Kullanıcı `/upper` veya `/lower` yazdığında botun o kullanıcıya özel davranışını aklında tutması gerekiyor. Çünkü her kullanıcının modu ayrı olabilir; biri büyük harf modundayken diğeri küçük harf modunda olabilir. Bu yüzden kullanıcıların chat ID'lerini key, seçtikleri modu ise value ("upper" veya "lower") olarak bu dict içinde saklayacağız.

Bir de `CommandHandler` ekliyoruz artık. BotFather'a kaydettiğimiz komut isimleriyle birebir uyumlu olacak şekilde komutları yakalayacak handler'lar tanımlıyoruz. `MessageHandler` nasıl mesajları yakalıyorsa, `CommandHandler` da başına `/` işareti koyulan komutları yakalar:

```python
        self.application.add_handler(CommandHandler("start", self.start_command))
        self.application.add_handler(CommandHandler("upper", self.upper_command))
        self.application.add_handler(CommandHandler("lower", self.lower_command))

```

Son olarak, normal mesajları yakalayan kısımda `filters.ALL` yerine tekrar şunu kullanıyoruz:

```python
        self.application.add_handler(
            MessageHandler(filters.TEXT & ~filters.COMMAND, self.on_update_received)
        )

```

Bunu yapmamızın çok net bir sebebi var `filters.TEXT`, burada kullanıcının yazdığı metinleri büyük veya küçük harfe çevireceğimiz için gelen verinin mutlaka string olması şart. Kullanıcı fotoğraf, ses veya çıkartma atarsa Python'da `.upper()` ya da `.lower()` string fonksiyonlarını çalıştıramayız, kod hata verir.

`~filters.COMMAND` ise Başı `/` ile başlayan komutların `/upper`, `/lower`, `/start` falan fonksiyona düşmesini engeller. Komutları zaten yukarıda tanımladığımız `CommandHandler`'lar yakalayacağı için, bu filtre sayesinde normal metin mesajları ile bot komutları birbirine karışmadan temiz bir şekilde ayrılmış olur. Şimdi `start_command` fonksiyonumuzu, yani `/start` komutu geldiğinde `CommandHandler`'ın çalıştıracağı kısmı yazıyorum:

```python
    async def start_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id

        await self.send_text(
            chat_id,
            "Komutlar:\n"
            "/upper - /upper değer şeklinde kullanırsan verdiğin değerleri büyük harflerle yazarım.\n"
            "/lower - /lower DEĞER şeklinde kullanırsan verdiğin değerleri küçük harflerle yazarım."
        )

```

Fark ettiyseniz burada chat ID bilgisini almak için daha önce kullandığımız `update.message.chat.id` yerine `update.effective_chat.id` kullandık.

Bunun sebebi Telegram'da her zaman düz bir mesaj gelmeyebilir; bazen mesaj düzenlenir, bazen bir butona basılır veya kanal iletisi gelir değinmiştim bunlara aslında ufaktan. `update.message` sadece saf mesajlarda varken, `update.effective_chat` lib'in sunduğu bir kolaylıktır. Olay ister yeni mesaj, ister düzenleme, ister buton tıklaması olsun, o anki aktif sohbeti otomatik bulur ve bize doğrudan chat nesnesini verir. Bu yüzden kodun patlamaması için `update.effective_chat.id` kullanmak en garantili yoldur. BU YÜZDEN BURAYI UNUTMAYIN BU KONUNUN SONUNDA NEDEN ÖNEMLİ OLDUĞUNU DAHA İYİ GÖRECEKSİNİZ.

Daha sonra klasik `send_text` fonksiyonumuz ile kullanıcıya komut listesini gönderiyoruz.

## upper ve lower Komut Fonksiyonları

Şimdi sırada `upper_command` yazmak var:

```python
    async def upper_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id

        self.user_modes[chat_id] = "upper"

        args = update.message.text.split(" ", 1)
        if len(args) > 1:
            await self.send_text(chat_id, args[1].upper())
        else:
            await self.send_text(chat_id, "Upper modu açıldı.")

```

Burada hem anlık çeviriyi hem de kalıcı modu aynı anda yöneten bir yapı kurduk - değişiklik yapacağım en son.

`self.user_modes[chat_id] = "upper"` ile kullanıcı bu komutu tetiklediği anda onun `chat_id` değerini dict'e key olarak yazıp değerini "upper" yapıyoruz. Böylece bot hafızasında bu kullanıcı artık upper moduna geçti bilgisini tutmuş oluyoruz.

`args = update.message.text.split(" ", 1)` ile kullanıcının yazdığı metni ilk boşluktan itibaren sadece 1 defa bölüyoruz.

Eğer sadece `split(" ")` deseydik ve kullanıcı `/upper merhaba nasılsın bugün` yazsaydı, her kelime ayrı elemana bölünecekti. Biz ise sadece komut kısmıyla yani `/upper` ile mesajın geri kalanını tek parça olarak ayırmak istiyoruz.

Örneğin kullanıcı `/upper ağlara fısıldayan adam` yazarsa, bu kod metni `["/upper", "ağlara fısıldayan adam"]` şeklinde 2 elemanlı bir liste yapar.

`if len(args) > 1` ile zaten listenin uzunluğu 1'den büyükse, demek ki kullanıcı komutun yanına bir şeyler yazmış yani `args[1]` mevcut. Bu durumda `args[1].upper()` diyerek mesajın geri kalanını büyük harfe çevirip kullanıcıya anında tek seferlik yanıt olarak gönderiyoruz.

Diğer türlü zaten eğer kullanıcı komutun yanına hiçbir şey yazmayıp sadece `/upper` gönderdiyse liste tek elemanlıdır (`["/upper"]`). Bu senaryoda yan argüman olmadığı için kullanıcıya "Upper modu açıldı." mesajı dönüyoruz. Zaten en başta `self.user_modes[chat_id] = "upper"` kaydı yapıldığı için, bundan sonra kullanıcının bota atacağı tüm düz metinler büyük harfe dönüşmeye hazır hâle gelmiş oluyor. Şimdi `lower_command` kısmını ele alayım hemen:

```python
    async def lower_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id

        self.user_modes[chat_id] = "lower"

        args = update.message.text.split(" ", 1)
        if len(args) > 1:
            await self.send_text(chat_id, args[1].lower())
        else:
            await self.send_text(chat_id, "Lower modu açıldı.")

```

Burada yine tamamen aynı mantık işliyor; sadece büyük harf yerine `.lower()` string metodunu kullanıyoruz ve kullanıcının modunu dict'e "lower" olarak güncelliyoruz.

## on_update_received Fonksiyonunda Mod Kontrolü

Ve bir diğer değişiklik ise `on_update_received` kısmında:

```python
    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id
        text = update.message.text
        mode = self.user_modes.get(chat_id)

        if mode == "upper":
            await self.send_text(chat_id, text.upper())
        elif mode == "lower":
            await self.send_text(chat_id, text.lower())
        else:
            await self.send_text(chat_id, text)

```

Buradaki mantığı biraz açalım.

`self.user_modes[chat_id]` şeklinde köşeli parantezle çağırsaydık, bot yeni başladığında veya kullanıcı henüz hiçbir komut vermeden doğrudan düz mesaj yazdığında o kullanıcı dict'e olmadığı için Python `KeyError` hatası verip çökerdi. `.get(chat_id)` kullandığımızda ise kullanıcı dict'e henüz kayıtlı değilse hata vermek yerine `None` döner. Böylece else bloğuna düşer ve mesajı bozmadan olduğu gibi geri gönderir.

Hem komutlarda hem de burada `.upper()` / `.lower()` kullanıyoruz fark ettiyseniz.

Çünkü iki farklı olay var burada. Birincisi anlık işlem: Kullanıcı doğrudan `/upper merhaba` yazdığında bu mesaj `upper_command` fonksiyonuna gider. Oradaki `.upper()` sadece o an komutun yanına yazılan "merhaba" kelimesini tek seferlik büyütür. Diğeri ise sürekli işlem: Kullanıcı sadece `/upper` yazıp bıraktığında veya daha önceden modu açtığında, bundan sonra attığı her düz mesaj "selam", "nasılsın" gibi `on_update_received` fonksiyonuna düşer. İşte burada dict'en kullanıcının o anki moduna bakıp, gelen düz metinleri hafızadaki moda göre büyütüp küçültüyoruz.

Bu arada çalıştıktan sonra şu `user_modes` kısmının arka planda nasıl dolduğunu ve değiştiğini merak ederseniz `on_update_received` fonksiyonuna şunu ekleyebilirsiniz:

```python
        user = update.message.from_user.full_name
        print(f"{user} kullanıcısının modu: {mode}")

```

Böylece terminalden gelen her mesajda kullanıcının o an hangi modda olduğunu canlı canlı takip edebilirsiniz.

Örnek olarak; ben bota önce `/upper` yazıp yolladım, sonra bir şeyler yazdım ve terminalde kontrol ettim:

```bash
‎ b kullanıcısının modu: upper

```

Sonra `/lower` dedim ve iki tane düz mesaj attım, terminal çıktısı şu şekilde aktı:

```bash
‎ b kullanıcısının modu: upper
‎ b kullanıcısının modu: lower
‎ b kullanıcısının modu: lower

```

Gördüğünüz gibi dict sayesinde bot her kullanıcının durumunu hafızasında tutarak ona göre dinamik tepki verebiliyor. Şimdi bahsetmek istediğim o çok önemli kısma gelelim. Daha önce `effective_chat.id` kullanmanın öneminden bahsetmiştik; çünkü mesaj düzenlenebilir, buton tıklaması gelebilir vesaire.

## Düzenlenen Mesajlar ve effective_message Önemi

Üstte hazırladığımız kod normalde tıkır tıkır çalışır. Ama gidip Telegram'da daha önce bota attığınız bir mesajı düzenlemeyi denerseniz terminalde hemen şu hatayı alırsınız:

```bash
args = update.message.text.split(" ", 1)
       ^^^^^^^^^^^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'text'

```

Telegram Bot API mimarisinde yeni atılan bir mesaj ile sonradan düzenlenen bir mesaj farklı Update alanlarına düşer.

Kullanıcı yeni bir mesaj attığında veri `update.message` nesnesi içine gelir.

Kullanıcı eski bir mesajını düzenlediğinde ise `update.message` tamamen `None` olur! Telegram bu veriyi `update.edited_message` alanına yazar.

Biz kodlarımızda sadece `update.message.text` yazdığımız için, kullanıcı bir mesajı düzenlediğinde `update.message` `None` geliyor ve Python doğal olarak "NoneType nesnesinin text özelliği olamaz" diyerek `AttributeError` veriyor. Biz daha önce sadece chat ID'yi sağlama almak için `update.effective_chat.id` yapmıştık ama mesaj metnini hâlâ saf `update.message` üzerinden çekiyorduk.

İşte bu yüzden `python-telegram-bot` kütüphanesinin sunduğu `update.effective_message` kullanmak lazım.

`effective_message` arka planda bu yeni bir mesaj mı, yoksa düzenlenen bir mesaj mı diye bakıyor, yoksa kanaldan gelen bir ileti mi? gibi sorguluyor aslında. Hangisi doluysa onu otomatik seçer ve bize doğrudan o mesaj nesnesini verir.

Kodlarımızda `update.message.text` gördüğümüz bütün kısımları `update.effective_message.text` ile değiştiriyoruz:

```python
args = update.effective_message.text.split(" ", 1)

```

```python
text = update.effective_message.text

```

Ayrıca kullanıcı adını aldığımız yeri de `update.effective_user.full_name` yaparak tamamen garantiye alabilirsiniz.

Bunu yaptıktan sonra test edin:

Önce bota `/upper` komutunu gönderin, ardından "merhaba" yazın. Bot "MERHABA" diye yanıt verecek. Şimdi gidip Telegram'da o önceden yazdığınız "merhaba" mesajını düzenleyip "selamlar test test" yapın. Bot hata vermeden, `effective_message` sayesinde düzenlenen metni anında yakalayacak ve yeni metne göre tekrar büyük harflerle yanıt verecektir.
