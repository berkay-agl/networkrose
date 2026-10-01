---
title: "Telegram Bot API'ye Giriş - #4"
date: 2026-10-01T22:15:00+03:00
draft: false
tags: ["Telegram", "Telegram-Bot-API", "Python"]
---

Ekstra bir pratik olarak echo bot örneği var. Bunun için yapmamız gereken kod bloğu şu şekilde, çok basit.

## Echo Bot Örneği

```python
    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        if update.message:
            user_id = update.message.chat.id
            text = update.message.text
            await self.send_text(user_id, text)

```

Şu şekilde bir değişiklik: User ID'yi yine alıyoruz, bir de üstüne `Update` object'ten text bilgisini çekiyoruz. Daha sonra `send_text` fonksiyonu `who`'ya yine `user_id` veriyor, `what` kısmına ise kullanıcının yolladığı metni veriyor. Echo botun amacı da ne yazarsan onu geri döndürmek ya; `Update` object'teki `text`'i alıp aynen geri gönderiyoruz.

## Her İçeriği Kopyalamak: filters.ALL

Bir de şimdi her şeyi fotoğraf, ses, çıkartma vs. kopyalayan bir bot deneyelim. Bunun için öncelikle bir kısıtlamamız vardı hatırlayın, `MessageHandler` kısmında yalnızca text mesajları yakalıyorduk. Bunu şöyle değiştirin:

```python
        self.application.add_handler(
            MessageHandler(filters.ALL & ~filters.COMMAND, self.on_update_received)
        )

```

`filters.ALL` sayesinde fotoğraf, ses dosyaları, çıkartmalar vesaire her türlü içerik bot tarafından yakalanabilecek.

## copy_message Fonksiyonunu Yazmak

Bundan sonra `copy_message` fonksiyonumuzu yapalım, `send_text`'e benzer şekilde aslında:

```python
    async def copy_message(self, who, msg_id):
        try:
            await self.application.bot.copy_message(chat_id=who, from_chat_id=who, message_id=msg_id)
        except Exception as e:
            print(f"{who}'a mesaj kopyalanamadı... Hata: {e}")

```

Burada `who`, `what` yerine `who`, `msg_id` kullanıyoruz çünkü amacımız yalnızca `text=what` diyerek bir yazı göndermek değil; mesaj içeriği (medya, ses, metin) her neyse tamamını kopyalamak.

`application.bot.copy_message` fonksiyonu Telegram'ın `copyMessage` metodunu çağırarak mesajı aynen kopyalamaya yarar.

* **chat_id=who:** Mesajın kopyalanıp gönderileceği hedef sohbet.
* **from_chat_id=who:** Mesajın orijinal olarak alındığı kaynak sohbet. (Kullanıcı bota attığı ve biz de aynı sohbete geri göndereceğimiz için ikisi de `who` yani kullanıcının chat ID'si oluyor.)
* **message_id=msg_id:** Orijinal mesajın ID'si; hangi mesajın kopyalanacağını belirtmek için şart.

## Medya ve Mesajları Dinamik Kopyalama

Şimdi `Update` object'lerini işlediğimiz fonksiyona gelelim:

```python
    async def on_update_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        if update.message:
            user_id = update.message.chat.id
            msg_id = update.message.message_id

            await self.copy_message(user_id, msg_id)

```

Burada yine `update.message` yakalıyoruz, `user_id` alıyoruz; sonuçta medyayı kimin yolladığı ve kime geri yollayacağımız önemli. `msg_id` ise `Update` object içindeki o mesaja ait özel ID (`message_id`). Bunu alıyoruz çünkü kopyalamak için şart.

En son `copy_message` fonksiyonumuzu `await` ile çağırıp kopyalama işlemini bitiriyoruz. Artık bota ister ses, ister fotoğraf, ister çıkartma atın; ne atarsanız aynısını size geri kopyalayacaktır.

Bu arada sanki her `Update` Object'i "mesajmış" gibi anlatmış olabilirim çünkü Telegram'ın dökümanı da bu şekilde. Mesela kullanıcı mesajını silebilir, değiştirebilir bunun gibi ihtimallerde olduğu için şuradan diğerleri neler bakabilirsiniz: [core.telegram.org/bots/api#update](https://core.telegram.org/bots/api#update)
