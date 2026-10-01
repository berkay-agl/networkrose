---
title: "Telegram Bot API'ye Giriş - #3"
date: 2026-10-01T20:45:00+03:00
draft: false
tags: ["Telegram", "Telegram-Bot-API", "Python"]
---


Şimdi değineceğim şey kullanıcılarından gelen mesajların işlenmesi aslında. Hani Telegram Bot API'den Update object geliyordu ya, bizim bunu tuttuğumuz daha doğrusu bu çağrıyı aldığımız fonksiyon `on_update_received`'di. Bu Update object üzerinden birçok bilgiye ulaşabiliyoruz, bir listesini aslında görmüş olduk önceki yazılarda; mesela `first_name`, `text` vesaire vesaire.

Daha iyi akılda kalması için kullanıcının gönderdiği mesajı mesela görmeyi deneyelim. Bu arada Update object'in bilgileri tam olarak listelenmiş, göz atabilirsiniz: [core.telegram.org/bots/api#update](https://core.telegram.org/bots/api#update)

## Kullanıcı Mesajını Yakalamak

Şimdi biz kullanıcının mesajını almak istiyorsak iki değeri bilmeliyiz:

* **Mesajı kim gönderdi?** Bu, Update object içinde `update.message.from_user` kısmında tutuluyor.
* **Ne gönderildi?** Yani kullanıcı bize ne yolladı o önemli, onu da `update.message.text` kısmında görebiliyoruz.

Şimdi daha önce `print(update)` diyerek o object'in o anki hâlini tamamen çekmiştik; yani bir kullanıcı text yolladığı zaman Update object'in nasıl gözüktüğünü gördük.

Şimdi fonksiyonumuzu güncelleyelim:

```python
    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        msg = update.message
        user = msg.from_user

        print(f"{user.first_name} isimli kullanıcının mesajı: {msg.text}")

```

Bu şekilde işte istediğimiz o mesaja erişebileceğiz. `python main.py` dediğimiz zaman polling mekanizması çalışıyor olacak, Telegram sunucusundan bir şeyler bekleyecek. Sonra oluşturduğumuz bota bir şeyler yazdığımız zaman terminalimizde kimin ne yazdığını görüyoruz:

```bash
‎ b isimli kullanıcının mesajı: Ağlara Fısıldayan Adam

```

Bu şekilde gördüğünüz gibi aslında ayıkladık resmen Update object'i.

## Bot ile Mesaj Göndermek

Şimdi bir de bot mesaj göndersin istiyorsak onun adımlarını göstereceğim.

Öncelikle ihtiyacımız olan burada 3 kısım var:

1. **Kullanıcının Etkileşimi:** Telegram kullanıcısı kesinlikle botunuza önceden `/start` diyerek bir mesaj atmış olmak zorunda. Botlar kullanıcılara durduk yere ilk mesajı atamaz. Bunun tek istisnası; botun admin olduğu bir kanala veya gruba katılım isteği gönderilmesi veya botun zaten mesaj gönderme yetkisiyle bir gruba, kanala eklenmiş olmasıdır.
2. **Chat/User ID:** Kullanıcının benzersiz `user_id` ya da `chat_id` değerini bilmemiz gerekiyor. Bunu Update object içinde `update.message.chat.id` veya `from_user.id` üzerinden yakalıyoruz.
3. **Mesaj Metni:** 1 ile 4096 karakter arasında metin içeren bir String dizisi gerekiyor Telegram tek mesajda en fazla 4096 karaktere izin veriyor.

Şimdi bot class'ımız içinde yeni bir fonksiyon açmamız lazım:

```python
    async def send_text(self, who, what):
        try:
            await self.application.bot.send_message(chat_id=who, text=what)
        except Exception as e:
            print(f"{who}'a mesaj gönderilemedi... Hata: {e}")

```

Burada `who` parametresi mesajın kime gideceğini belirten `chat_id` değerini tutar; `what` ise botun kullanıcıya göndereceği metni yani "text"i taşır.

`self.application.bot.send_message` kısmı ise python-telegram-bot lib'in Telegram Bot API'deki `sendMessage` metoduna HTTP POST isteği göndererek mesajı ilettiği asıl fonksiyondur.

Bir de fark ettiyseniz bu fonksiyon `async`. Çünkü ağ üzerinden Telegram sunucularına bir istek gönderilirken botun donup kilitlenmemesi, aynı anda birden fazla kullanıcının isteklerine cevap verebilmesi için bu işlem arka planda asenkron çalışır.

## Statik Mesaj Gönderimi 

Ve bu asenkron yapı gereği bu mesaj gönderme fonksiyonunu doğrudan senkron çalışan `main` içinde çağıramıyoruz. Bunun yerine kullanılan yöntemlerden biri; bot henüz polling mekanizmasına girip sürekli dinlemeye başlamadan hemen önce, yani ayağa kalktığı `__init__` aşamada bu mesajı tetiklemek. Bunun için `post_init` diye bir fonksiyon oluşturuyorum:

```python
    async def post_init(self, application):
        await self.send_text(765745xxx, "Ağlara Fısıldayan Adam")

```

Bu kısımda işte `who`, `what` değerlerini veriyoruz. En önemlisi `__init__` kısmında bot yapılandırılırken bu `post_init`'i tanıtmak:

```python
    def __init__(self):
        self.application = (
            Application.builder()
            .token(self.get_bot_token())
            .post_init(self.post_init)
            .build()
        )

```

Ayrıca sadece text mesaj değil; fotoğraf, ses, video gibi bir sürü metot var: [core.telegram.org/bots/api#available-methods](https://core.telegram.org/bots/api#available-methods)

Bu `post_init` yöntemiyle `python main.py` çalıştırdığımız her seferde bot ayağa kalkarken verdiğimiz statik user ID'ye bu mesajı otomatik olarak iletir.

## Dinamik Yanıt Mekanizması

Yalnız şöyle bir durum var; eğer biz böyle elle girdiğimiz statik bir ID'ye değil de, bota kim yazdıysa anında dinamik olarak ona cevap vermek istiyorsak, `post_init` kısmındaki mesajı kaldırabiliriz yada orayı boş bırakabiliriz. Asıl dinamik yanıt mekanizmasını `on_update_received` içinde kurgularız:

```python
    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        if update.message:
            user_id = update.message.chat.id
            await self.send_text(user_id, "Ağlara Fısıldayan Adam")

```

Burada `if update.message:` ile gelen güncellemede bir mesaj nesnesi var mı diye kontrol ediyoruz. Varsa `update.message.chat.id` üzerinden mesajı atan kişinin ID'sini alıp `self.send_text` fonksiyonumuzu çağırıyoruz.

Başına koyduğumuz `await` anahtar kelimesi ise Python'a bu mesaj Telegram sunucularına iletilene kadar burada bekle, ama bu sırada arkadan gelen diğer işlemleri engelleme, programı dondurma talimatı veriyor. Asenkron tanımlanmış tüm fonksiyonlar çağrılırken mutlaka başına `await` konulmalıdır bu önemli ya.

> Edit: Böylece bu şekilde hem statik (bot başlarken tek seferlik) hem de dinamik (gelen mesaja anında yanıt veren) mesaj gönderimini göstermiş oldum.
