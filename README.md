# Edit
import os
import textwrap
from PIL import Image, ImageDraw, ImageFont, UnidentifiedImageError
from telebot.types import Photo # Пример для pybot, если Photo - это объект фото

FONT_PATH = os.path.join(os.path.dirname(__file__), "Ubuntu-R.ttf")

async def make_mindfule_quote(ctx, quote_text: str):
    if not ctx.replied or not ctx.replied.attachments:
        await ctx.reply("Ответьте на сообщение, содержащее фотографию, на которую хотите наложить текст.")
        return

    photo_attachment = None
    for att in ctx.replied.attachments:
        if isinstance(att, Photo):
            photo_attachment = att
            break

    if not photo_attachment:
        await ctx.reply("В отвеченном сообщении не найдено поддерживаемое изображение.")
        return

    try:
        bio = await photo_attachment.download_bytes()
    except Exception as e:
        await ctx.reply(f"Не удалось загрузить фотографию. Возможно, проблема с сетью или доступом. Ошибка: {e}")
        return

    try:
        image = Image.open(io.BytesIO(bio))
    except (UnidentifiedImageError, ValueError):
        await ctx.reply("Не удалось распознать изображение. Возможно, файл поврежден или имеет неподдерживаемый формат.")
        return
    except Exception as e:
        await ctx.reply(f"Произошла неизвестная ошибка при открытии изображения: {e}")
        return

    original_width, original_height = image.size
    
    max_side = 1024
    if original_width > max_side or original_height > max_side:
        if original_width > original_height:
            new_width = max_side
            new_height = int(original_height * (max_side / original_width))
        else:
            new_height = max_side
            new_width = int(original_width * (max_side / original_height))
        image = image.resize((new_width, new_height), Image.Resampling.LANCZOS)

    border_height = int(image.height * 0.25)
    image_with_border = Image.new("RGB", (image.width, image.height + border_height), (0, 0, 0))
    image_with_border.paste(image, (0, border_height))

    draw = ImageDraw.Draw(image_with_border)

    font_size = 40
    try:
        font = ImageFont.truetype(FONT_PATH, font_size)
    except OSError:
        await ctx.reply(f"Ошибка: Не найден файл шрифта '{FONT_PATH}'. Убедитесь, что он существует и доступен.")
        return
    except Exception as e:
        await ctx.reply(f"Произошла ошибка при загрузке шрифта: {e}")
        return

    max_text_width = image_with_border.width - 40
    wrapped_text = textwrap.fill(quote_text, width=max_text_width // (font_size // 2))

    text_width, text_height = draw.textsize(wrapped_text, font=font)

    text_x = (image_with_border.width - text_width) / 2
    text_y = (border_height - text_height) / 2

    if text_height > border_height * 0.9:
        await ctx.reply("Текст слишком длинный, чтобы полностью поместиться в рамку. Возможно, он будет обрезан.")

    draw.text((text_x, text_y), wrapped_text, font=font, fill=(255, 255, 255))

    output_buffer = io.BytesIO()
    try:
        image_with_border.save(output_buffer, format="PNG")
        output_buffer.seek(0)
        await ctx.reply_photo(output_buffer)
    except Exception as e:
        await ctx.reply(f"Произошла ошибка при отправке фотографии: {e}")
