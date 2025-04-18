Please help fix my python code for my Discord bot.
The code itself contains Ukrainian text, so I apologize.
Here is my code:

import discord
from discord.ext import commands
import random
import json
import os

    # Завантаження токена
with open("config.json", "r") as f:
        config = json.load(f)

TOKEN = config["token"]

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)

    # Символи
PLAYER = "❌"
BOT = "⭕"
EMPTY = "⬜"

    # Файл рейтингу
RATING_FILE = "battleship_rating.json"
if not os.path.exists(RATING_FILE):
        with open(RATING_FILE, "w") as f:
            json.dump({}, f)

def load_rating():
        with open(RATING_FILE, "r") as f:
            return json.load(f)

def save_rating(rating):
        with open(RATING_FILE, "w") as f:
            json.dump(rating, f)

@bot.event
async def on_ready():
        print(f"✅ Бот {bot.user} онлайн і готовий до бою!")

    # Команда запуску гри
@bot.command()
async def play(ctx):
        view = GameModeView()
        await ctx.send("🎮 Обери гру:", view=view)

    # Команда рейтингу
@bot.command()
async def rating(ctx):
        rating = load_rating()
        if not rating:
            await ctx.send("📊 Ще немає записів у рейтингу.")
            return
        sorted_rating = sorted(rating.items(), key=lambda x: x[1], reverse=True)
        text = "🏆 **Рейтинг морського бою:**\n"
        for i, (user_id, points) in enumerate(sorted_rating[:10], 1):
            user = await bot.fetch_user(int(user_id))
            text += f"{i}. {user.name}: {points} очок\n"
        await ctx.send(text)

    # ---------- Вибір гри ----------
class GameModeView(discord.ui.View):
        @discord.ui.button(label="Хрестики-нулики", style=discord.ButtonStyle.primary)
        async def ttt(self, interaction: discord.Interaction, button: discord.ui.Button):
            await interaction.response.edit_message(content="🧠 Обери режим гри:", view=TTTModeView())

        @discord.ui.button(label="Морський бій", style=discord.ButtonStyle.secondary)
        async def sea_battle(self, interaction: discord.Interaction, button: discord.ui.Button):
            view = SeaBattleGame(interaction.user)
            await interaction.response.edit_message(content="🚢 Морський бій! Поціль по кораблях!", view=view)

    # ---------- Вибір режиму хрестиків ----------
class TTTModeView(discord.ui.View):
        @discord.ui.button(label="PvP", style=discord.ButtonStyle.primary)
        async def pvp(self, interaction: discord.Interaction, button: discord.ui.Button):
            view = TicTacToePvP(interaction.user)
            await interaction.response.edit_message(content="🆚 Хрестики-нулики (PvP). Очікуємо другого гравця...", view=view)

        @discord.ui.button(label="Проти бота", style=discord.ButtonStyle.secondary)
        async def vs_bot(self, interaction: discord.Interaction, button: discord.ui.Button):
            await interaction.response.edit_message(content="🎮 Обери складність:", view=DifficultyView())

    # ---------- Хрестики-нулики PvP ----------
class TicTacToePvP(discord.ui.View):
        def __init__(self, player1):
            super().__init__(timeout=None)
            self.board = [[EMPTY for _ in range(3)] for _ in range(3)]
            self.player1 = player1
            self.player2 = None
            self.turn = player1
            self.game_over = False
            self.update_buttons()

        def update_buttons(self):
            self.clear_items()
            for row in range(3):
                for col in range(3):
                    label = self.board[row][col]
                    disabled = self.game_over or label != EMPTY
                    self.add_item(PvPButton(row, col, label, disabled))

        def check_winner(self, symbol):
            b = self.board
            for row in b:
                if all(cell == symbol for cell in row):
                    return True
            for col in range(3):
                if all(b[row][col] == symbol for row in range(3)):
                    return True
            if all(b[i][i] == symbol for i in range(3)) or all(b[i][2 - i] == symbol for i in range(3)):
                return True
            return False

        def is_full(self):
            return all(cell != EMPTY for row in self.board for cell in row)

class PvPButton(discord.ui.Button):
        def __init__(self, row, col, label, disabled):
            super().__init__(label=label, style=discord.ButtonStyle.secondary, row=row, disabled=disabled)
            self.row = row
            self.col = col

        async def callback(self, interaction: discord.Interaction):
            view: TicTacToePvP = self.view
            player = interaction.user

            if view.game_over:
                await interaction.response.send_message("Гра завершена!", ephemeral=True)
                return

            if view.player2 is None and player != view.player1:
                view.player2 = player

            if player != view.turn:
                await interaction.response.send_message("Зараз не твій хід!", ephemeral=True)
                return

            if view.board[self.row][self.col] != EMPTY:
                await interaction.response.send_message("Поле зайняте!", ephemeral=True)
                return

            symbol = PLAYER if player == view.player1 else BOT
            view.board[self.row][self.col] = symbol

            if view.check_winner(symbol):
                view.game_over = True
                view.update_buttons()
                await interaction.response.edit_message(content=f"🎉 {player.name} переміг!", view=view)
                return

            if view.is_full():
                view.game_over = True
                view.update_buttons()
                await interaction.response.edit_message(content="⚖️ Нічия!", view=view)
                return

            view.turn = view.player2 if player == view.player1 else view.player1
            view.update_buttons()
            await interaction.response.edit_message(view=view)

    # ---------- Хрестики-нулики проти бота ----------
class TicTacToeView(discord.ui.View):
        def __init__(self, difficulty="hard"):
            super().__init__(timeout=None)
            self.board = [[EMPTY for _ in range(3)] for _ in range(3)]
            self.difficulty = difficulty
            self.game_over = False
            self.update_buttons()

        def update_buttons(self):
            self.clear_items()
            for row in range(3):
                for col in range(3):
                    label = self.board[row][col]
                    disabled = self.game_over or label != EMPTY
                    self.add_item(TicTacToeButton(row, col, label, disabled))

        def make_bot_move(self):
            if self.difficulty == "easy":
                empty = [(r, c) for r in range(3) for c in range(3) if self.board[r][c] == EMPTY]
                if empty:
                    r, c = random.choice(empty)
                    self.board[r][c] = BOT
            else:
                best_score = -float('inf')
                best_move = None
                for r in range(3):
                    for c in range(3):
                        if self.board[r][c] == EMPTY:
                            self.board[r][c] = BOT
                            score = self.minimax(False)
                            self.board[r][c] = EMPTY
                            if score > best_score:
                                best_score = score
                                best_move = (r, c)
                if best_move:
                    self.board[best_move[0]][best_move[1]] = BOT

        def minimax(self, is_max):
            if self.check_winner(BOT):
                return 1
            if self.check_winner(PLAYER):
                return -1
            if self.is_full():
                return 0

            scores = []
            for r in range(3):
                for c in range(3):
                    if self.board[r][c] == EMPTY:
                        self.board[r][c] = BOT if is_max else PLAYER
                        scores.append(self.minimax(not is_max))
                        self.board[r][c] = EMPTY
            return max(scores) if is_max else min(scores)

        def check_winner(self, symbol):
            b = self.board
            for row in b:
                if all(cell == symbol for cell in row):
                    return True
            for col in range(3):
                if all(b[row][col] == symbol for row in range(3)):
                    return True
            if all(b[i][i] == symbol for i in range(3)) or all(b[i][2-i] == symbol for i in range(3)):
                return True
            return False

        def is_full(self):
            return all(cell != EMPTY for row in self.board for cell in row)

class TicTacToeButton(discord.ui.Button):
        def __init__(self, row, col, label, disabled):
            super().__init__(label=label, style=discord.ButtonStyle.secondary, row=row, disabled=disabled)
            self.row = row
            self.col = col

        async def callback(self, interaction: discord.Interaction):
            view: TicTacToeView = self.view
            if view.game_over or view.board[self.row][self.col] != EMPTY:
                await interaction.response.send_message("⛔ Неможливо!", ephemeral=True)
                return

            view.board[self.row][self.col] = PLAYER
            if view.check_winner(PLAYER):
                view.game_over = True
                view.update_buttons()
                await interaction.response.edit_message(content="🎉 Ти переміг!", view=view)
                return

            view.make_bot_move()
            if view.check_winner(BOT):
                view.game_over = True
                view.update_buttons()
                await interaction.response.edit_message(content="🤖 Бот переміг!", view=view)
                return

            if view.is_full():
                view.game_over = True
                view.update_buttons()
                await interaction.response.edit_message(content="⚖️ Нічия!", view=view)
                return

            view.update_buttons()
            await interaction.response.edit_message(view=view)

class DifficultyView(discord.ui.View):
        @discord.ui.button(label="Легко 🎲", style=discord.ButtonStyle.success)
        async def easy(self, interaction: discord.Interaction, button: discord.ui.Button):
            await interaction.response.edit_message(content="🎮 Хрестики-нулики (легко)", view=TicTacToeView("easy"))

        @discord.ui.button(label="Складно 🧠", style=discord.ButtonStyle.danger)
        async def hard(self, interaction: discord.Interaction, button: discord.ui.Button):
            await interaction.response.edit_message(content="🎮 Хрестики-нулики (складно)", view=TicTacToeView("hard"))

# ---------- Морський бій ----------
class SeaBattleGame(discord.ui.View):
    def __init__(self, player):
        super().__init__(timeout=None)
        self.player = player
        self.board_size = 4  
        self.board = [["🌊" for _ in range(self.board_size)] for _ in range(self.board_size)]
        self.ship_count = 5  # 5 кораблів
        self.max_shots = 12  # 12 пострілів
        self.remaining_shots = self.max_shots
        self.ships = self._generate_non_overlapping_ships() 
        self.hits = 0
        self.multiplier = 1
        self.points = 0
        self.scan_used = False
        self.bomb_used = False
        self.selected_cell = None
        self.update_buttons()

    def get_button_style(self, label):
        """Повертає стиль кнопки на основі її вмісту."""
        if label == "🚢":  # Корабель
            return discord.ButtonStyle.primary
        elif label == "🔥":  # Потрапляння
            return discord.ButtonStyle.green
        elif label == "❌":  # Промах
            return discord.ButtonStyle.grey
        elif label == "💥":  # Знищення
            return discord.ButtonStyle.red
        else:  # Вода
            return discord.ButtonStyle.secondary

    def get_cell_label(self, x, y):
        """Повертає емоціону для клітинки на основі її стану."""
        if (x, y) in self.ships:
            return "🚢"  # Корабель
        elif self.board[x][y] == "🔥":
            return "🔥"  # Потрапляння
        elif self.board[x][y] == "❌":
            return "❌"  # Промах
        elif self.board[x][y] == "💥":
            return "💥"  # Знищення
        else:
            return "🌊"  # Вода

    def _generate_non_overlapping_ships(self):
        """Генерує кораблі, які не торкаються один одного"""
        ships = set()
        attempts = 0
        max_attempts = 100
        
        while len(ships) < self.ship_count and attempts < max_attempts:
            x, y = random.randint(0, self.board_size-1), random.randint(0, self.board_size-1)
            new_ship = (x, y)
            
            # Перевіряємо, чи новий корабель не перетинається з існуючими
            overlapping = False
            for (ship_x, ship_y) in ships:
                if abs(ship_x - x) <= 1 and abs(ship_y - y) <= 1:
                    overlapping = True
                    break
            
            if not overlapping:
                ships.add(new_ship)
            attempts += 1
        
        return ships

    async def scan_area(self, x, y):
        """Сканує область 1x1 навколо заданих координат"""
        ships_in_area = 0
        
        nx, ny = x, y
        if 0 <= nx < self.board_size and 0 <= ny < self.board_size:
            if (nx, ny) in self.ships:
                ships_in_area += 1
        
        return ships_in_area

    async def bomb_area(self, x, y):
        """Бомбить область 1x1 навколо заданих координат"""
        destroyed_ships = 0
        
        nx, ny = x, y
        if 0 <= nx < self.board_size and 0 <= ny < self.board_size:
            if (nx, ny) in self.ships:
                self.ships.remove((nx, ny))
                self.board[nx][ny] = "💥"
                destroyed_ships += 1
            elif self.board[nx][ny] == "🌊":
                self.board[nx][ny] = "❌"
        
        return destroyed_ships

    def update_buttons(self):
        self.clear_items()
        for y in range(self.board_size):
            for x in range(self.board_size):
                label = self.get_cell_label(x, y)
                style = self.get_button_style(label)
                self.add_item(SeaButton(x, y, label, style, self))

        # Додаємо кнопки дій, якщо обрана клітинка
        if self.selected_cell:
            if not self.scan_used:
                self.add_item(ScanButton())
            if not self.bomb_used:
                self.add_item(BombButton())
            self.add_item(ShootButton())  # нова кнопка для пострілу
            self.add_item(CancelButton())

class SeaButton(discord.ui.Button):
    def __init__(self, row, col, emoji, disabled, view):
        super().__init__(label=emoji, style=discord.ButtonStyle.secondary, row=row, disabled=disabled)
        self.row = row
        self.col = col
        self._view = view  # Змінили тут

    @property
    def view(self):
        return self._view  # Використовуємо властивість для отримання view

    async def callback(self, interaction: discord.Interaction):
        if interaction.user != self.view.player:
            await interaction.response.send_message("Це не твоя гра!", ephemeral=True)
            return

        # Запам'ятовуємо обрану клітинку
        self.view.selected_cell = (self.row, self.col)
        self.view.update_buttons()
        await interaction.response.edit_message(
            content=f"🎯 **Обрано клітинку ({self.row}, {self.col}). Обери дію:**",
            view=self.view
        )

# Кнопка для пострілу
class ShootButton(discord.ui.Button):
    def __init__(self):
        super().__init__(label="🎯 Вистрілити", style=discord.ButtonStyle.green, row=4)

    async def callback(self, interaction: discord.Interaction):
        view: SeaBattleGame = self.view
        if view.remaining_shots <= 0:
            await interaction.response.send_message("У тебе закінчились постріли!", ephemeral=True)
            return

        x, y = view.selected_cell
        coord = (x, y)
        view.selected_cell = None
        view.remaining_shots -= 1

        if coord in view.ships:
            view.board[x][y] = "🔥"
            view.ships.remove(coord)
            view.add_points()
            result = f"🔥 Влучив по кораблю на ({x}, {y})!"
        else:
            view.board[x][y] = "❌"
            view.miss()
            result = f"❌ Промах на ({x}, {y})!"

        view.update_buttons()

        if not view.ships:
            rating = load_rating()
            user_id = str(view.player.id)
            rating[user_id] = rating.get(user_id, 0) + view.points
            save_rating(rating)

            await interaction.response.edit_message(
                content=f"{result}\n🎯 Гру завершено! Твої очки: {view.points}",
                view=PlayAgainView()
            )
        elif view.remaining_shots <= 0:
            await interaction.response.edit_message(
                content=f"{result}\n💥 Постріли закінчилися! Очки: {view.points}",
                view=PlayAgainView()
            )
        else:
            await interaction.response.edit_message(
                content=result,
                view=view
            )

# Кнопка для сканування
class ScanButton(discord.ui.Button):
    def __init__(self):
        super().__init__(label="🔍 Сканувати", style=discord.ButtonStyle.blurple, row=4)

    async def callback(self, interaction: discord.Interaction):
        view: SeaBattleGame = self.view
        if view.scan_used:
            await interaction.response.send_message("Ви вже використали сканування!", ephemeral=True)
            return
        
        x, y = view.selected_cell
        ships_found = await view.scan_area(x, y)
        view.scan_used = True
        view.selected_cell = None
        
        await interaction.response.edit_message(
            content=f"🔍 Сканування області ({x}, {y}): знайдено {ships_found} кораблі(в)!",
            view=view
        )
        view.update_buttons()

# Кнопка для бомбардування
class BombButton(discord.ui.Button):
    def __init__(self):
        super().__init__(label="💣 Бомбити", style=discord.ButtonStyle.red, row=4)

    async def callback(self, interaction: discord.Interaction):
        view: SeaBattleGame = self.view
        if view.bomb_used:
            await interaction.response.send_message("Ви вже використали бомбу!", ephemeral=True)
            return
        
        x, y = view.selected_cell
        destroyed = await view.bomb_area(x, y)
        view.bomb_used = True
        view.selected_cell = None
        view.remaining_shots -= 1
        
        if destroyed > 0:
            view.points += destroyed * 15
        
        await interaction.response.edit_message(
            content=f"💥 Бомбардування ({x}, {y}): знищено {destroyed} кораблі(в)!",
            view=view
        )
        view.update_buttons()

# Кнопка для скасування дії
class CancelButton(discord.ui.Button):
    def __init__(self):
        super().__init__(label="❌ Скасувати", style=discord.ButtonStyle.grey, row=4)

    async def callback(self, interaction: discord.Interaction):
        view: SeaBattleGame = self.view
        view.selected_cell = None
        view.update_buttons()
        await interaction.response.edit_message(
            content="🎯 Дія скасована. Обери нову клітинку!",
            view=view
        )

# Перегляд на завершення гри
class PlayAgainView(discord.ui.View):
    @discord.ui.button(label="Зіграємо ще раз?", style=discord.ButtonStyle.success)
    async def again(self, interaction: discord.Interaction, button: discord.ui.Button):
        view = GameModeView()
        await interaction.response.edit_message(content="🎮 Обери гру:", view=view)

    @discord.ui.button(label="Ні, дякую", style=discord.ButtonStyle.danger)
    async def no(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.edit_message(content="Дякую за гру! 🎮", view=None)

# Додаткові функції для збереження та завантаження рейтингу
def load_rating():
    # Логіка для завантаження рейтингу з файлу або бази даних
    pass

def save_rating(rating):
    # Логіка для збереження рейтингу у файл або базу даних
    pass

# Запуск бота
bot.run(TOKEN)


PS: sorry for my bad English, I use a translator

Also, when I press the "Sea Battle" button in Discord to play, I get this error:

2025-04-18 14:50:28 ERROR    discord.ui.view Ignoring exception in view <GameModeView timeout=180.0 children=2> for item <Button style=<ButtonStyle.secondary: 2> url=None disabled=False label='Морський бій' emoji=None row=None sku_id=None>
Traceback (most recent call last):
  File "C:\Users\Uncle Bogdan\Desktop\discord bot\.venv\Lib\site-packages\discord\ui\view.py", line 435, in _scheduled_task
    await item.callback(interaction)
  File "c:\Users\Uncle Bogdan\Desktop\discord bot\main.py", line 70, in sea_battle
    await interaction.response.edit_message(content="🚢 Морський бій! Поціль по кораблях!", view=view)
  File "C:\Users\Uncle Bogdan\Desktop\discord bot\.venv\Lib\site-packages\discord\interactions.py", line 1154, in edit_message        
    response = await adapter.create_interaction_response(
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Uncle Bogdan\Desktop\discord bot\.venv\Lib\site-packages\discord\webhook\async_.py", line 226, in request
    raise HTTPException(response, data)
discord.errors.HTTPException: 400 Bad Request (error code: 50035): Invalid Form Body
In data.components.0.components.0.disabled: Must be either true or false.
In data.components.0.components.1.disabled: Must be either true or false.
In data.components.0.components.2.disabled: Must be either true or false.
In data.components.0.components.3.disabled: Must be either true or false.
In data.components.1.components.0.disabled: Must be either true or false.
In data.components.1.components.1.disabled: Must be either true or false.
In data.components.1.components.2.disabled: Must be either true or false.
In data.components.1.components.3.disabled: Must be either true or false.
In data.components.2.components.0.disabled: Must be either true or false.
In data.components.2.components.1.disabled: Must be either true or false.
In data.components.2.components.2.disabled: Must be either true or false.
In data.components.2.components.3.disabled: Must be either true or false.
In data.components.3.components.0.disabled: Must be either true or false.
In data.components.3.components.1.disabled: Must be either true or false.
In data.components.3.components.2.disabled: Must be either true or false.
In data.components.3.components.3.disabled: Must be either true or false.


