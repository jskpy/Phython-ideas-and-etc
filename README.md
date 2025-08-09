import tkinter as tk
import math
import random
import time

# Configurações gerais
WIDTH, HEIGHT = 900, 600
MAP = [
    "111111111111111",
    "1             1",
    "1  11  1      1",
    "1    1        1",
    "1 1          11",
    "1             1",
    "111111111111111"
]
TILE_SIZE = 64
FOV = math.pi / 3
NUM_RAYS = 120
MAX_DEPTH = 800
DELTA_ANGLE = FOV / NUM_RAYS
DIST_PROJ_PLANE = (WIDTH / 2) / math.tan(FOV / 2)
SCALE = WIDTH // NUM_RAYS
MAP_WIDTH = len(MAP[0])
MAP_HEIGHT = len(MAP)

class Player:
    def __init__(self):
        self.x = TILE_SIZE + TILE_SIZE / 2
        self.y = TILE_SIZE + TILE_SIZE / 2
        self.angle = 0
        self.speed = 0
        self.rotation_speed = 0
        self.health = 100
        self.ammo = 12
        self.score = 0

class Enemy:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.health = 50
        self.size = 22
        self.speed = 1.5
        self.alive = True

    def update(self, player):
        if not self.alive:
            return
        dx = player.x - self.x
        dy = player.y - self.y
        dist = math.hypot(dx, dy)
        if dist > 25:
            dx /= dist
            dy /= dist
            nx = self.x + dx * self.speed
            ny = self.y + dy * self.speed
            # Checa colisão parede simples
            if MAP[int(ny // TILE_SIZE)][int(nx // TILE_SIZE)] != '1':
                self.x = nx
                self.y = ny

class Bullet:
    def __init__(self, x, y, angle):
        self.x = x
        self.y = y
        self.angle = angle
        self.speed = 20
        self.active = True

    def update(self):
        self.x += math.cos(self.angle) * self.speed
        self.y += math.sin(self.angle) * self.speed
        mx = int(self.x // TILE_SIZE)
        my = int(self.y // TILE_SIZE)
        if mx < 0 or mx >= MAP_WIDTH or my < 0 or my >= MAP_HEIGHT or MAP[my][mx] == '1':
            self.active = False

class Game:
    def __init__(self, root):
        self.root = root
        self.canvas = tk.Canvas(root, width=WIDTH, height=HEIGHT, bg='black')
        self.canvas.pack()

        self.player = Player()
        self.enemies = [Enemy(random.randint(2, MAP_WIDTH - 2) * TILE_SIZE,
                              random.randint(2, MAP_HEIGHT - 2) * TILE_SIZE) for _ in range(5)]
        self.bullets = []
        self.keys = set()
        self.last_shot = 0
        self.shot_cooldown = 300

        self.root.bind("<KeyPress>", self.key_down)
        self.root.bind("<KeyRelease>", self.key_up)

        self.game_loop()

    def key_down(self, e):
        self.keys.add(e.keysym)

    def key_up(self, e):
        if e.keysym in self.keys:
            self.keys.remove(e.keysym)

    def move_player(self):
        speed = 3
        rot_speed = 0.06
        if 'w' in self.keys:
            nx = self.player.x + math.cos(self.player.angle) * speed
            ny = self.player.y + math.sin(self.player.angle) * speed
            if MAP[int(ny // TILE_SIZE)][int(nx // TILE_SIZE)] != '1':
                self.player.x = nx
                self.player.y = ny
        if 's' in self.keys:
            nx = self.player.x - math.cos(self.player.angle) * speed
            ny = self.player.y - math.sin(self.player.angle) * speed
            if MAP[int(ny // TILE_SIZE)][int(nx // TILE_SIZE)] != '1':
                self.player.x = nx
                self.player.y = ny
        if 'Left' in self.keys:
            self.player.angle -= rot_speed
        if 'Right' in self.keys:
            self.player.angle += rot_speed
        self.player.angle %= 2 * math.pi

    def shoot(self):
        now = time.time() * 1000
        if 'space' in self.keys and now - self.last_shot > self.shot_cooldown and self.player.ammo > 0:
            px = self.player.x + math.cos(self.player.angle) * 20
            py = self.player.y + math.sin(self.player.angle) * 20
            self.bullets.append(Bullet(px, py, self.player.angle))
            self.player.ammo -= 1
            self.last_shot = now

    def update_bullets(self):
        for b in self.bullets[:]:
            b.update()
            if not b.active:
                self.bullets.remove(b)
            else:
                for e in self.enemies:
                    if e.alive and math.hypot(b.x - e.x, b.y - e.y) < e.size:
                        e.health -= 20
                        b.active = False
                        if e.health <= 0:
                            e.alive = False
                            self.player.score += 100
                        break

    def update_enemies(self):
        for e in self.enemies:
            e.update(self.player)
            if e.alive and math.hypot(e.x - self.player.x, e.y - self.player.y) < 25:
                self.player.health -= 1
                if self.player.health <= 0:
                    self.game_over()

    def cast_ray(self, angle):
        sin_a = math.sin(angle)
        cos_a = math.cos(angle)
        for depth in range(MAX_DEPTH):
            x = self.player.x + depth * cos_a
            y = self.player.y + depth * sin_a
            mx = int(x // TILE_SIZE)
            my = int(y // TILE_SIZE)
            if 0 <= mx < MAP_WIDTH and 0 <= my < MAP_HEIGHT:
                if MAP[my][mx] == '1':
                    return depth, (mx, my)
        return MAX_DEPTH, None

    def draw_3d(self):
        self.canvas.delete("all")
        for ray in range(NUM_RAYS):
            ray_angle = self.player.angle - FOV / 2 + ray * DELTA_ANGLE
            depth, _ = self.cast_ray(ray_angle)
            depth *= math.cos(self.player.angle - ray_angle)
            proj_height = (TILE_SIZE * DIST_PROJ_PLANE) / (depth + 0.0001)
            shade = int(255 / (1 + depth * depth * 0.0001))
            color = f'#{shade:02x}{shade:02x}{shade:02x}'
            x = ray * SCALE
            self.canvas.create_rectangle(x, HEIGHT / 2 - proj_height / 2, x + SCALE, HEIGHT / 2 + proj_height / 2,
                                         fill=color, outline='')

        # Desenhar inimigos como retângulos vermelhos com sombra
        for e in self.enemies:
            if e.alive:
                dx = e.x - self.player.x
                dy = e.y - self.player.y
                angle = math.atan2(dy, dx) - self.player.angle
                if -FOV / 2 < angle < FOV / 2:
                    dist = math.hypot(dx, dy)
                    dist *= math.cos(angle)
                    size = (TILE_SIZE * DIST_PROJ_PLANE) / (dist + 0.0001)
                    screen_x = WIDTH // 2 + math.tan(angle) * DIST_PROJ_PLANE
                    self.canvas.create_rectangle(screen_x - size // 4, HEIGHT / 2 - size // 2,
                                                 screen_x + size // 4, HEIGHT / 2 + size // 2, fill="red")

        self.draw_hud()
        self.draw_minimap()
        self.draw_weapon()

    def draw_hud(self):
        # Barra de vida
        self.canvas.create_rectangle(20, 20, 220, 50, fill="gray20", outline="white", width=2)
        self.canvas.create_rectangle(20, 20, 20 + (self.player.health * 2), 50, fill="red")
        self.canvas.create_text(120, 35, text=f"Vida: {self.player.health}", fill="white", font=("Arial", 16))

        # Ammo
        self.canvas.create_text(WIDTH - 120, 35, text=f"Munição: {self.player.ammo}", fill="yellow", font=("Arial", 16))

        # Score
        self.canvas.create_text(WIDTH // 2, 35, text=f"Pontos: {self.player.score}", fill="white", font=("Arial", 16))

    def draw_minimap(self):
        mini_size = 150
        scale = mini_size / (MAP_WIDTH * TILE_SIZE)
        offset_x, offset_y = 20, HEIGHT - mini_size - 20

        # Fundo
        self.canvas.create_rectangle(offset_x, offset_y, offset_x + mini_size, offset_y + mini_size, fill="gray10")

        # Desenhar mapa
        for y in range(MAP_HEIGHT):
            for x in range(MAP_WIDTH):
                if MAP[y][x] == '1':
                    self.canvas.create_rectangle(offset_x + x * TILE_SIZE * scale,
                                                 offset_y + y * TILE_SIZE * scale,
                                                 offset_x + (x + 1) * TILE_SIZE * scale,
                                                 offset_y + (y + 1) * TILE_SIZE * scale,
                                                 fill="gray70")

        # Jogador
        px = offset_x + self.player.x * scale
        py = offset_y + self.player.y * scale
        self.canvas.create_oval(px - 5, py - 5, px + 5, py + 5, fill="blue")

        # Direção do jogador
        dx = math.cos(self.player.angle) * 20 * scale
        dy = math.sin(self.player.angle) * 20 * scale
        self.canvas.create_line(px, py, px + dx, py + dy, fill="blue", width=2)

        # Inimigos
        for e in self.enemies:
            if e.alive:
                ex = offset_x + e.x * scale
                ey = offset_y + e.y * scale
                self.canvas.create_oval(ex - 5, ey - 5, ex + 5, ey + 5, fill="red")

    def draw_weapon(self):
        base_x = WIDTH // 2
        base_y = HEIGHT - 100
        self.canvas.create_rectangle(base_x - 30, base_y, base_x + 30, base_y + 60, fill="gray20", outline="black")
        self.canvas.create_rectangle(base_x + 10, base_y - 10, base_x + 50, base_y + 10, fill="darkgray", outline="black")
        self.canvas.create_oval(base_x - 40, base_y + 10, base_x - 10, base_y + 40, fill="dimgray", outline="black")

    def game_over(self):
        self.canvas.delete("all")
        self.canvas.create_text(WIDTH // 2, HEIGHT // 2, text="GAME OVER", fill="red", font=("Arial", 50))

    def game_loop(self):
        if self.player.health > 0:
            self.move_player()
            self.shoot()
            self.update_bullets()
            self.update_enemies()
            self.draw_3d()
        else:
            self.game_over()

        self.root.after(20, self.game_loop)

root = tk.Tk()
root.title("Doom-like Python Tkinter")
game = Game(root)
root.mainloop()
