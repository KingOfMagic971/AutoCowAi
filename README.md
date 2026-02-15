# AutoCowAi
Полностью автоматизирует походы в лес и дойку
#              _     _             _aa
#    _  Branch| |__ | | ___   ___ | |_  _   _
#   | |/ / _ \| '_ \| |/ _ \ / _ \| __|| | | |
#   |   < (_) | | | | | (_) | (_) | |_ | |_| |
#   |_|\_\___/|_| |_|_|\___/ \___/ \__| \__,_|
#
# meta developer: @k1sIotaa
# scope: auto_cow_pro

import asyncio
import re
from .. import loader, utils
from telethon.tl.types import Message

@loader.tds
class AutoCowAIMod(loader.Module):
    """AI-АвтоКоровка: Лес, Дойка, Еда и фильтр животных"""
    
    strings = {"name": "АвтоКоровка PRO"}

    def __init__(self):
        self.config = loader.ModuleConfig(
            loader.ConfigValue("chat_id", 0, "ID игрового чата"),
            loader.ConfigValue("forest", False, "Автоматический лес"),
            loader.ConfigValue("cow", False, "Автоматическая дойка"),
            loader.ConfigValue("eat", False, "Автоматическая еда (50%+)"),
            loader.ConfigValue("skip_list", ["Тедди", "Белочка", "Жабомразь", "Единорожка", "Винди", "Джун"], "Кого пропускать"),
        )
        self.active_tasks = {}

    @loader.command()
    async def cwg(self, message: Message):
        """Меню управления и настройки"""
        await self.show_main_menu(message)

    async def show_main_menu(self, message):
        target = self.config["chat_id"] if self.config["chat_id"] != 0 else "НЕ УСТАНОВЛЕН"
        text = (f"<b>🐄 Настройки АвтоКоровки</b>\n"
                f"<b>📍 Чат:</b> <code>{target}</code>\n"
                f"<b>🤖 Статус ИИ:</b> Активен")
        
        btns = [
            [
                ("🌲 Лес: " + ("✅" if self.config["forest"] else "❌"), "toggle_f"),
                ("🐮 Дойка: " + ("✅" if self.config["cow"] else "❌"), "toggle_c")
            ],
            [
                ("🥦 Еда: " + ("✅" if self.config["eat"] else "❌"), "toggle_e"),
                ("⚙️ Указать ID чата", "set_cid")
            ],
            [("🐾 Настройка лесных жителей", "set_anim")],
            [("❌ Закрыть", "close")]
        ]
        await self.inline.form(message=message, text=text, reply_markup=btns)

    # --- CALLBACKS ---
    async def toggle_f_callback(self, call):
        self.config["forest"] = not self.config["forest"]
        await self.show_main_menu(call.message)

    async def toggle_c_callback(self, call):
        self.config["cow"] = not self.config["cow"]
        await self.show_main_menu(call.message)

    async def toggle_e_callback(self, call):
        self.config["eat"] = not self.config["eat"]
        await self.show_main_menu(call.message)

    async def set_cid_callback(self, call):
        async with self._client.conversation(call.sender_id) as conv:
            m = await conv.send_message("<b>Отправь ID чата или перешли сообщение из бота:</b>")
            r = await conv.get_reply()
            cid = r.chat_id if r.forward else int(r.text)
            self.config["chat_id"] = cid
            await m.edit(f"✅ ID чата установлен: <code>{cid}</code>")
            await self.show_main_menu(call.message)

    async def set_anim_callback(self, call):
        btns = []
        for anim in ["Тедди", "Белочка", "Жабомразь", "Единорожка", "Винди", "Джун"]:
            status = "✅" if anim in self.config["skip_list"] else "❌"
            btns.append([(f"{anim} {status}", f"anim_{anim}")])
        btns.append([("⬅️ Назад", "back_main")])
        await call.edit("<b>🐾 Выбери кого нужно ПРОПУСКАТЬ:</b>", reply_markup=btns)

    async def anim_callback(self, call):
        anim_name = call.data.split("_")[1]
        current = list(self.config["skip_list"])
        if anim_name in current: current.remove(anim_name)
        else: current.append(anim_name)
        self.config["skip_list"] = current
        await self.set_anim_callback(call)

    # --- LOGIC ---
    async def watcher(self, message: Message):
        if not isinstance(message, Message) or message.chat_id != self.config["chat_id"]:
            return
        
        text = message.text or ""

        # 1. АВТО-ЛЕС (Таймеры и Пропуск)
        if self.config["forest"]:
            # Распознавание времени до леса
            timer = re.search(r"(?:через|откат|ждать)\s*(\d+)\s*(?:мин|м)", text.lower())
            if timer:
                minutes = int(timer.group(1))
                await self.schedule_task("forest", minutes * 60, message.chat_id, "Мулс")
            
            # Если появилось животное из списка - пропускаем
            for anim in self.config["skip_list"]:
                if anim in text:
                    await asyncio.sleep(1)
                    await message.click(text="Пропустить")
                    break

        # 2. АВТО-ДОЙКА
        if self.config["cow"]:
            if "пора доить" in text.lower() or "можно подоить" in text.lower():
                await self._client.send_message(message.chat_id, "Мук")
                await asyncio.sleep(2)
                await message.click(text="Подоить")

        # 3. АВТО-ПОЕДАНИЕ (Сытость)
        if self.config["eat"]:
            hunger_match = re.search(r"Сытость:\s*(\d+)%", text)
            if (hunger_match and int(hunger_match.group(1)) <= 50) or "голодает" in text.lower():
                await self._client.send_message(message.chat_id, "Мук")
                await asyncio.sleep(2)
                # Ищем кнопки с едой
                for btn in ["🌿", "🥦", "Травка", "Брокколи"]:
                    if await message.click(text=btn): break

    async def schedule_task(self, name, seconds, chat_id, command):
        """Интеллектуальный планировщик команд"""
        task_id = f"{name}_{chat_id}"
        if task_id in self.active_tasks:
            self.active_tasks[task_id].cancel()
        
        async def _waiter():
            await asyncio.sleep(seconds + 2)
            await self._client.send_message(chat_id, command)
            del self.active_tasks[task_id]
        
        self.active_tasks[task_id] = asyncio.create_task(_waiter())

    @loader.command()
    async def auto_forest(self, message):
        """Вкл/Выкл лес"""
        self.config["forest"] = not self.config["forest"]
        await utils.answer(message, f"🌲 Лес: {self.config['forest']}")

    @loader.command()
    async def auto_cow(self, message):
        """Вкл/Выкл дойку"""
        self.config["cow"] = not self.config["cow"]
        await utils.answer(message, f"🐮 Дойка: {self.config['cow']}")

    @loader.command()
    async def auto_eating(self, message):
        """Вкл/Выкл еду"""
        self.config["eat"] = not self.config["eat"]
        await utils.answer(message, f"🥦 Еда: {self.config['eat']}")
