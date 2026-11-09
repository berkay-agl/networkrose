---
title: "Telegram Bot API'ye Giriş - #6"
date: 2026-11-04T14:21:00+03:00
draft: false
tags: ["Telegram", "Telegram-Bot-API", "Python"]
---

Bu kısımda artık değinmek istediğim konu Butonlar ve Klavyeler. Metin tabanlı diyalogları daha pratik ve işlevsel olan butonlarla değiştirebiliriz.

Telegram'da temelde iki adet buton tipi var, önce ona değinmek istiyorum:

1. **Reply Butonları:** Kullanıcıya önceden tanımlı metin seçenekleri sunmaya yarar. Ekranın alt kısmında, normal telefon klavyesinin yerine açılır ve kullanıcı hızlıca dokunup yanıt verebilir.
2. **Inline Butonlar:** Mesajın hemen altında, sohbetin içinde görünürler. Hızlı navigasyonlar, kısayollar, URL açma, oyunlar ve daha fazlası içindir.

Ben de bu yazımda Inline Butonlarına sadık kalacağım ve onun üzerinden anlatacağım.

## Gerekli Nesneleri Import Etmek

Öncelikle gerekli dependencies kısımlarını import edelim:

```python
import os
from dotenv import load_dotenv
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    MessageHandler,
    CommandHandler,
    filters,
    ContextTypes,
)

load_dotenv()

```

Burada `InlineKeyboardButton` ve `InlineKeyboardMarkup` asıl başroller diyebilirim. Çünkü bunlar butonları oluşturmak ve buton düzenlerini mesaja yapıştırmak için kullanılan düzenleyici object'ler.

## Buton Oluşturma Mantığı

Butonları oluşturmak için şu şekilde kodlara ihtiyacımız var. `__init__` metodunda Application builder'dan önce tanımlamak lazım:

```python
def __init__(self):
    
    next_button = InlineKeyboardButton(text="İleri", callback_data="ileri")
    back_button = InlineKeyboardButton(text="Geri", callback_data="geri")
    
    url_button = InlineKeyboardButton(
        text="NetworkRose",
        url="https://networkrose.is-a.dev"
    )

```

Şimdi burada belirttiğim alanlar şu şekilde:

* `text=""` kısmı kullanıcının göreceği metin; butonun üstünde bu text yazacak.
* `callback_data=""` kısmı: Butona tıklanıldığı zaman Telegram sunucuları botumuza yeni bir Update içinde bu veriyi geri döndürür. Bunun sayesinde hangi butona basıldığını kod tarafında hızlıca tanımlama imkânımız olacak.
* `url_button` kısmındaki `url=""` ise `callback_data` gerektirmez. Çünkü davranışı zaten tanımlıdır; kullanıcı butona tıkladığında doğrudan belirtilen URL'ye yönlendirilir.

## Klavye Düzenlerini Oluşturmak

Bu adımdan sonra oluşturduğumuz butonları klavyelere, yani buton düzenlerine yerleştirmek kaldı. Bu klavyeler iki menü arasında ileri-geri gezinmek için kullanılacak.

Önce klavyeleri saklamak için yine class seviyesinde iki alan eklememiz lazım:

```python
self.keyboard_m1 = InlineKeyboardMarkup([[next_button]])
self.keyboard_m2 = InlineKeyboardMarkup([[back_button], [url_button]])

```

Burada dikkat edilmesi gereken nokta: `InlineKeyboardMarkup` butonları iki boyutlu bir liste olarak bekler. Her iç liste, klavyedeki bir satıra karşılık gelir.

* `keyboard_m1` içinde sadece `[[next_button]]` var; yani ilk satırda sadece İleri butonu duruyor.
* `keyboard_m2` içinde ise `[[back_button], [url_button]]` var. İki ayrı iç liste olduğu için Geri butonu üst satırda, URL butonu ise onun altındaki satırda yer alacak.

Eğer bir satırda birden fazla butonu yan yana koymak isteseydik, o satırın listesine art arda buton eklememiz yeterliydi: `[[button1, button2]]` gibi.

Bu kodları istediğimiz yere yerleştirebiliriz ama önemli olan kısım; klavye değişkenlerinin, menüyü gönderecek metodun erişebileceği bir yerde (örneğin `self` üzerinde) tanımlanmış olmasıdır.

## Menü Gönderme Fonksiyonu ve HTML Formatı

Şimdi klavyeleri kullanıcıya iletmek kaldı. Gönderdiğimiz mesaja `reply_markup` parametresi eklemek gerekiyor bunun için. Ayrıca metinde formatlama desteği kazandırmak için de `parse_mode="HTML"` parametresini eklememiz lazım.

```python
async def send_menu(self, who, txt, kb):
    try:
        await self.application.bot.send_message(
            chat_id=who,
            text=txt,
            reply_markup=kb,
            parse_mode="HTML"
        )
    except Exception as e:
        print(f"Menü gönderilemedi. Hata: {e}")

```

Belirtilen klavyeyi içeren menü mesajı için bu fonksiyona ihtiyacımız var:

* `who`: Mesajın gönderileceği kullanıcının `chat_id` bilgisini taşır.
* `txt`: HTML tag destekli mesaj metni (örneğin `<i>Menu 1</i>` gibi italik yazılar için).
* `kb`: Oluşturduğumuz `InlineKeyboardMarkup` nesnesi.

`reply_markup=kb` diyerek oluşturduğumuz buton düzenini doğrudan mesaja yapıştırmış oluyoruz.

## Komut Ekleme ve Menüyü Tetikleme

Şimdi menüyü tetiklemek için gerekli işlemleri göstereceğim.

Öncelikle botunuza **@BotFather** ile `/menu` komutunu veya istediğiniz bir isimde menü komutunu tanımlayın.

Sonrasında `__init__` içinde bu komutu yakalamak için bir handler oluşturalım:

```python
self.application.add_handler(CommandHandler("menu", self.menu_command))

```

Ardından `menu_command` metodunu yazıyorum:

```python
async def menu_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_message.from_user.id
    await self.send_menu(
        user_id,
        "<i> Menu 1</i>",
        self.keyboard_m1
    )

```

Bunların ardından bir eksik yoksa Telegram botunda `/menu` yazınca "Menu 1" yazan italik bir metin ve altında İleri yazan butonlu bir menü gelir.

Tam koda ve tüm tutorial boyunca eklediğim örneklere şuradan bakabilirsiniz:

[GitHub - 10_En_Temel.py](https://github.com/berkay-agl/Telegram-Bot-API-Example/blob/main/10_En_Temel.py)

Bu arada bu kodlara kadar İleri ve Geri butonları `callback_data` değerini taşıyor ama bunları henüz tetikleyecek, işleyecek `CallbackQueryHandler` yapısını eklemedik. Onu eklemediğimiz için şu an butonlara tıklasak bile ekranda bir hareket göremeyiz. Buton tıklamalarını yakalama ve menüler arasında canlı gezinmeyi diğer yazımda (#7) anlatacağım.
