---
title: "Telegram Bot API'ye Giriş - #2"
date: 2026-10-01T20:40:00+03:00
draft: false
tags: ["Telegram", "Telegram-Bot-API", "Python"]
---

Gerekli şeylere değindikten sonra artık ilk Python ile Telegram Botumuzu kurgulamaya başlayalım.

Öncelikle bot oluşturmak için bir bot class'ına ihtiyacımız var. Bu class dediğimiz şey aslında Object-Oriented Programming (OOP) dediğimiz; yani veri ve fonksiyonları bir arada tutan class yapıları üzerine kurulu bir yaklaşım. Class'ın içine belirli bir mantığı yazdığımız, veri ve davranışları bir arada tutan bir şablon gibi.

## Bot Sınıfını Kurgulamak

Şimdi `bot.py` dosyasını şu şekilde kurguluyorum; bu arada `python-telegram-bot` librariesi, Telegram BOT API ile iletişimi `Application` class'ı üzerinden hallediyor. Bizim yazacağımız class'ın içinde bir `Application` örneği oluşturarak bağlantıyı kuracağız.

```python
import os
from dotenv import load_dotenv
from telegram import Update
from telegram.ext import Application, MessageHandler, filters, ContextTypes

load_dotenv()

class Bot:
    def __init__(self):
        self.application = Application.builder().token(self.get_bot_token()).build()
        self.application.add_handler(
            MessageHandler(filters.TEXT & ~filters.COMMAND, self.on_update_received)
        )

    def get_bot_username(self):
        return "TestBot"

    def get_bot_token(self):
        return os.getenv("BOT_TOKEN", "")

    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        pass

    def run(self):
        self.application.run_polling()

```

Class'ımız temel hâliyle bu şekilde. En temel 3 çok önemli fonksiyon var burada:

`get_bot_username` kısmını Java tutorial'ında zorunlu olduğu için buraya da ekledim, botumuzun username bilgisini tutuyor. `get_bot_token` ise framework'e botumuzun gerekli token bilgisini veriyor; bu bilgiyi framework `.env` sayesinde alıp işliyor.

Artı olarak bir diğer önemli kısım `on_update_received`. Bu aslında bu class'ın en önemli kısmı. Telegram sunucularından her yeni `Update` object geldiği zaman (işte örneğin kullanıcının mesaj girmesi, kullanıcının bir butona tıklaması gibi gibi; yani botumuza ulaşan her türlü güncelleme verisi neyse) bu fonksiyon sayesinde framework otomatik olarak bu `Update` object'ini çağırıyor.

## İlk Test ve Update Nesnesini İncelemek

Telegram sunucularından `Update` olarak ne gelmiş diye kontrol edelim şimdiden:

```python
    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        print(update)

```

`main.py` dosyasına da şu kodları girelim:

```python
from bot import Bot

if __name__ == '__main__':
    bot = Bot()
    bot.run()

```

Ve `python main.py` dediğimiz zaman eğer herhangi bir libraries, dependencies sorunu yoksa çalışmaya başlayacak.

Test etmek için oluşturduğunuz Telegram botuna bir mesaj yollayın.

İşte şu şekilde anında `Update` object çıktısını görüyorum:

```python
Update(message=Message(channel_chat_created=False, chat=Chat(first_name='\u200e b', id=85179xxx, type=<ChatType.PRIVATE>), date=datetime.datetime(2026, 8, 30, 16, 31, 8, tzinfo=datetime.timezone.utc), delete_chat_photo=False, from_user=User(first_name='\u200e b', id=85179xxx, is_bot=False, language_code='tr'), group_chat_created=False, message_id=318, supergroup_chat_created=False, text='Ağlara Fısıldayan Adam'), update_id=549900230)

```

Bakın işte Telegram'dan gelen JSON verisinin library tarafından Python nesnesine dönüştürülmüş hâlini görebilirsiniz. Daha önce dedim mi hatırlamıyorum; içinde `first_name`, `message_id`, `text` gibi gibi bilgiler var. Şimdi kodların ve bu yapının üstüne daha iyi konuşabiliriz:

## Polling Mekanizması

```python
    def run(self):
        self.application.run_polling()

```

Dediğimiz kısım hani bahsetmiştim ya Telegram BOT API gelen mesajı iletiyor diye; bunu Python ile yazdığımız bota alırken Polling dediğimiz veri aktarım yöntemini kullanıyor. Burada bu polling zaten belli aralıklarla botumuzun API'ye yeni `Update` object var mı diye sorması yöntemidir.

## Handler ve Filtreleme Mantığı

Şu kısım ise:

```python
    def __init__(self):
        self.application = Application.builder().token(self.get_bot_token()).build()
        self.application.add_handler(
            MessageHandler(filters.TEXT & ~filters.COMMAND, self.on_update_received)
        )

```

Öncelikle bu `__init__` kısmı class'ların başlatıcı fonksiyonu. Burada ben mesela ilk botun kurulumu, gerekli token'ı almak gibi işlemleri yaptım. Bu arada `self` ifadesini de çok görürsünüz, bu da zaten class içindeki variable ve fonksiyonlara dışarıdan değil içeriden erişmek istediğimizi belirtmek için buna yarıyor yani.

`Application.builder().token(self.get_bot_token()).build()` burası botumuzu Telegram API'ye tanıtan kısım. `Application.builder()` gerekli yapıyı kurar yani. `.token()` da bizim yazdığımız `get_bot_token` üzerinden çekiyor bunu.

`self.application.add_handler` kısmı ise botumuzun aslında kullanıcı tarafıyla alakalı bir işlevi; yani kullanıcıların atacağı mesaj, ses, fotoğraf vesaire bunları yakalar. Ben `MessageHandler` diyerek mesajların yakalanmasını söyledim. Mesela `filters.TEXT` diyerek kullanıcıdan gelen mesajın text formatında olmasını söyler, artı `~filters.COMMAND` ise `/test` gibi komutları yakalamamasını söyler. Yani yazı olsun ama başında `/` işareti olan bir komut olmasın, böylece dümdüz text mesajlarını yakalamış oluyoruz. Bunu test edin yani bota mesela `/start selam` yazdığın zaman `on_update_received` içine düşmez. En sonda `self.on_update_received` işte burayı çalıştırır, tabii `MessageHandler` ile bizim istediğimiz formata uyuyorsa gelen mesaj.
