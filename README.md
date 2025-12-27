# The-Christmas-Wheel
The Christmas Wheel
A creation by Taxiarchis with the help of his dad and ChatGPT.

Pygame παιχνίδι τύπου “Τροχός της Τύχης” με:
- 3 παίκτες
- νεόν τίτλο και εφέ
- χιόνι / sparkles / glow wheel
- αγορά φωνήεντος και λύση λέξης
- τελική οθόνη νικητή


import math
import random
import pygame
import os

os.system('cls' if os.name == 'nt' else 'clear')
BG_PATH = r"C:\Users\tsifa\Desktop\V.S Code\taxiarchis\1.jpg"
FINAL_IMAGE_PATH = r"C:\Users\tsifa\Desktop\V.S Code\taxiarchis\taxiarchis.png"
FPS = 60

WORDS = [
    "PYTHON", "JAVASCRIPT", "VARIABLE", "FUNCTION", "ALGORITHMICS",
    "KEYBOARD", "PROCESS", "MEMORY", "POINTER", "MOUSE", "COMPUTER"]

VOWELS = set("A E I O U Y")

WHEEL_SLICES = [
    ("ΛΑΜΠΑΚΙΑ", 100),
    ("ΜΠΑΛΛΕΣ", 200),
    ("ΧΑΝΕΙΣ ΣΕΙΡΑ", "LOSE_TURN"),
    ("ΔΕΝΔΡΟ", 300),
    ("ΣΤΟΛΙΔΙΑ", 400),
    ("BONUS", "BONUS"),
    ("ΧΙΟΝΙ", 500),
    ("ΑΣΤΕΡΙΑ", 600),
    ("ΦΑΤΝΗ", 700),
    ("ΧΡΕΩΚΟΠΙΑ", "BANKRUPT"),]

VOWEL_COST = 100
WIN_BONUS = 1000
GOLD = (255, 215, 0)
BLACK = (0, 0, 0)
SNOW_WHITE = (245, 245, 255)
NEON_CYAN = (120, 240, 255)
NEON_PURPLE = (210, 140, 255)

def lerp(a, b, t):
    return a + (b - a) * t

def lerp_color(c1, c2, t):
    return (
        int(lerp(c1[0], c2[0], t)),
        int(lerp(c1[1], c2[1], t)),
        int(lerp(c1[2], c2[2], t)),)

NEON_TITLE_COLORS = [
    (120, 240, 255),  
    (210, 140, 255),  
    (255, 90, 190),   
    (255, 170, 90),]

def neon_cycle_color(t, speed=0.75):
    n = len(NEON_TITLE_COLORS)
    x = (t * speed) % n
    i = int(x)
    f = x - i
    c1 = NEON_TITLE_COLORS[i]
    c2 = NEON_TITLE_COLORS[(i + 1) % n]
    return lerp_color(c1, c2, f)

def draw_neon_text_center(surface, text, font, cx, cy, t):
    base = neon_cycle_color(t, speed=0.75)
    pulse = 0.5 + 0.5 * math.sin(t * 3.0)
    glow_a = int(70 + 130 * pulse)
    out_a = int(60 + 90 * (1 - pulse))
    img = font.render(text, True, base)
    rect = img.get_rect(center=(cx, cy))
    glow = font.render(text, True, base).convert_alpha()
    glow.set_alpha(glow_a)
    for dx, dy in [(-3, 0), (3, 0), (0, -3), (0, 3), (-2, -2), (2, 2), (-2, 2), (2, -2)]:
        surface.blit(glow, (rect.x + dx, rect.y + dy))
    outline = font.render(text, True, (10, 10, 10)).convert_alpha()
    outline.set_alpha(out_a)
    for dx, dy in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
        surface.blit(outline, (rect.x + dx, rect.y + dy))
    surface.blit(img, rect.topleft)
    return rect

def mask_word(word, guessed):
    return " ".join([ch if ch in guessed else "_" for ch in word])

def all_revealed(word, guessed):
    return all(ch in guessed for ch in word)

def angle_norm(a):
    twopi = 2 * math.pi
    a = a % twopi
    if a < 0:
        a += twopi
    return a

def pick_wheel_result(rotation, n_slices, pointer_angle=-math.pi / 2):
    slice_angle = 2 * math.pi / n_slices
    wheel_angle = angle_norm(pointer_angle - rotation)
    idx = int(wheel_angle // slice_angle)
    return idx

def wrap_letters(letters, max_per_line=14):
    if not letters:
        return ["-"]
    parts = list(sorted(letters))
    lines, line = [], []
    for p in parts:
        line.append(p)
        if len(line) >= max_per_line:
            lines.append(" ".join(line))
            line = []
    if line:
        lines.append(" ".join(line))
    return lines

def draw_gold_text(surface, text, font, x, y, color=GOLD):
    shadow_img = font.render(text, True, (0, 0, 0))
    surface.blit(shadow_img, (x + 2, y + 2))
    outline_img = font.render(text, True, (20, 20, 20))
    for dx, dy in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
        surface.blit(outline_img, (x + dx, y + dy))
    img = font.render(text, True, color)
    surface.blit(img, (x, y))
    return img.get_rect(topleft=(x, y))

def draw_gold_text_center(surface, text, font, cx, cy, color=GOLD):
    img = font.render(text, True, color)
    rect = img.get_rect(center=(cx, cy))
    draw_gold_text(surface, text, font, rect.x, rect.y, color)
    return rect

def make_rounded_mask(size, radius):
    mask = pygame.Surface(size, pygame.SRCALPHA)
    pygame.draw.rect(mask, (255, 255, 255, 255), mask.get_rect(), border_radius=radius)
    return mask

def draw_image_panel(screen, bg_image, rect, alpha=130, radius=18, border_color=(245, 245, 245), border_w=2):
    screen_rect = screen.get_rect()
    safe_rect = rect.clip(screen_rect)
    panel = pygame.Surface(rect.size, pygame.SRCALPHA)
    if safe_rect.width > 0 and safe_rect.height > 0:
        cut = bg_image.subsurface(safe_rect).copy()
        ox = safe_rect.x - rect.x
        oy = safe_rect.y - rect.y
        panel.blit(cut, (ox, oy))
    panel.set_alpha(alpha)
    mask = make_rounded_mask(rect.size, radius)
    panel.blit(mask, (0, 0), special_flags=pygame.BLEND_RGBA_MULT)
    screen.blit(panel, rect.topleft)
    pygame.draw.rect(screen, border_color, rect, border_w, border_radius=radius)

def draw_image_button(screen, bg_image, rect, text, font, enabled=True):
    alpha = 175 if enabled else 110
    border = (245, 245, 245)
    draw_image_panel(screen, bg_image, rect, alpha=alpha, radius=14, border_color=border, border_w=2)
    draw_gold_text_center(screen, text, font, rect.centerx, rect.centery, GOLD)

def draw_exit_button(screen, rect, font, hover=False):
    fill = (220, 60, 60) if not hover else (235, 90, 90)
    pygame.draw.rect(screen, fill, rect, border_radius=8)
    pygame.draw.rect(screen, (40, 40, 40), rect, 2, border_radius=8)
    draw_gold_text_center(screen, "X", font, rect.centerx, rect.centery, GOLD)

def make_glow_circle(radius, color, strength=220):
    size = radius * 2 + 2
    surf = pygame.Surface((size, size), pygame.SRCALPHA)
    cx, cy = size // 2, size // 2
    for r in range(radius, 0, -2):
        a = int(strength * (r / radius) ** 2)
        pygame.draw.circle(surf, (*color, a), (cx, cy), r)
    return surf

def draw_neon_ring(screen, center, base_r, t, color_a=NEON_CYAN, color_b=NEON_PURPLE):
    cx, cy = center
    pulse = 0.5 + 0.5 * math.sin(t * 2.2)
    ring_r = int(base_r + 10 + pulse * 10)
    surf = pygame.Surface((ring_r * 2 + 6, ring_r * 2 + 6), pygame.SRCALPHA)
    rc = surf.get_rect(center=(cx, cy))
    scx, scy = surf.get_width() // 2, surf.get_height() // 2
    a1 = int(70 + 80 * pulse)
    a2 = int(55 + 70 * (1 - pulse))
    pygame.draw.circle(surf, (*color_a, a1), (scx, scy), ring_r, 4)
    pygame.draw.circle(surf, (*color_b, a2), (scx, scy), ring_r - 8, 3)
    ticks = 18
    for i in range(ticks):
        ang = (i / ticks) * 2 * math.pi + t * 0.8
        x1 = scx + math.cos(ang) * (ring_r - 2)
        y1 = scy + math.sin(ang) * (ring_r - 2)
        x2 = scx + math.cos(ang) * (ring_r + 8)
        y2 = scy + math.sin(ang) * (ring_r + 8)
        pygame.draw.line(surf, (*NEON_CYAN, 60), (x1, y1), (x2, y2), 2)
    screen.blit(surf, rc.topleft)

def draw_sparkles(screen, center, base_r, t, count=14):
    cx, cy = center
    for i in range(count):
        ang = (i / count) * 2 * math.pi + t * 1.5
        wobble = 6 * math.sin(t * 3 + i)
        rr = base_r + 22 + wobble
        x = cx + math.cos(ang) * rr
        y = cy + math.sin(ang) * rr
        a = int(70 + 100 * (0.5 + 0.5 * math.sin(t * 4 + i)))
        pygame.draw.circle(screen, (*NEON_CYAN, a), (int(x), int(y)), 3)
        pygame.draw.circle(screen, (*NEON_PURPLE, a // 2), (int(x), int(y)), 6, 1)

class Snowflake:
    def __init__(self, w, h):
        self.reset(w, h, start_top=True)
    def reset(self, w, h, start_top=False):
        self.x = random.uniform(0, w)
        self.y = random.uniform(-h * 0.2, h) if not start_top else random.uniform(-h, -10)
        self.r = random.uniform(1.0, 3.6)
        self.speed = random.uniform(40, 140) * (0.7 + self.r / 4.0)
        self.drift = random.uniform(-30, 30)
        self.phase = random.uniform(0, math.pi * 2)
    def update(self, dt, w, h, wind):
        self.phase += dt * random.uniform(1.2, 2.2)
        self.x += (self.drift + wind + math.sin(self.phase) * 18) * dt
        self.y += self.speed * dt
        if self.x < -10:
            self.x = w + 10
        elif self.x > w + 10:
            self.x = -10
        if self.y > h + 10:
            self.reset(w, h, start_top=True)
    def draw(self, screen):
        pygame.draw.circle(screen, (*SNOW_WHITE, 180), (int(self.x), int(self.y)), int(self.r))
        pygame.draw.circle(screen, (*NEON_CYAN, 45), (int(self.x), int(self.y)), int(self.r * 2), 1)

class Game:
    def __init__(self):
        self.players = [
            {"name": "ΠΑΝΑΓΙΩΤΗΣ", "score": 0},
            {"name": "ΘΕΟΔΟΡΗΣ", "score": 0},
            {"name": "ΤΑΞΙΑΡΧΗΣ", "score": 0},]
        self.turn = 0
        self.word = random.choice(WORDS)
        self.guessed = set()
        self.used_letters = set()
        self.phase = "SPIN"
        self.last_spin = None
        self.message = "Σειρά έχει ο ΠΑΝΑΓΙΩΤΗΣ — Πάτα SPACE για να γυρίσει ο τροχός."
        self.rotation = 0.0
        self.angular_vel = 0.0
        self.spinning = False
        self.friction = 0.985
        self.solve_mode = False
        self.solve_text = ""
        self.await_vowel = False
    def current_player(self):
        return self.players[self.turn]
    def next_turn(self):
        self.turn = (self.turn + 1) % len(self.players)
        self.phase = "SPIN"
        self.last_spin = None
        self.solve_mode = False
        self.solve_text = ""
        self.await_vowel = False
        self.message = f"Σειρά έχει ο {self.current_player()['name']} — Πάτα SPACE για SPIN."

    def start_spin(self):
        if self.spinning or self.phase != "SPIN":
            return
        self.spinning = True
        self.angular_vel = random.uniform(6.0, 10.5)
        self.message = "Ο τροχός γυρίζει..."

    def stop_spin(self):
        self.spinning = False
        self.angular_vel = 0.0
        idx = pick_wheel_result(self.rotation, len(WHEEL_SLICES))
        label, value = WHEEL_SLICES[idx]
        self.last_spin = (label, value)
        if value == "LOSE_TURN":
            self.message = "❌ ΧΑΝΕΙΣ ΣΕΙΡΑ!"
            self.next_turn()
            return
        if value == "BANKRUPT":
            self.current_player()["score"] = 0
            self.message = "💥 ΧΡΕΩΚΟΠΙΑ! ΜΗΔΕΝΙΣΜΟΣ ΣΚΟΡ! Πας στον επόμενο."
            self.next_turn()
            return
        if value == "BONUS":
            self.message = "✨ BONUS! Διάλεξε γράμμα (A-Z) ή λύσε (ENTER)."
            self.phase = "ACTION"
            return
        self.message = f"🎯 Έπεσες σε {value} πόντους. Διάλεξε ΣΥΜΦΩΝΟ (B-Z) ή λύσε (ENTER)."
        self.phase = "ACTION"

    def update(self, dt):
        if self.spinning:
            self.rotation += self.angular_vel * dt
            self.angular_vel *= (self.friction ** (dt * 60.0))
            if self.angular_vel < 0.25:
                self.stop_spin()

    def guess_letter(self, letter):
        letter = letter.upper()
        if self.phase != "ACTION" or self.await_vowel or self.solve_mode:
            return
        if not letter.isalpha():
            return
        if letter in self.used_letters:
            self.message = f"Το γράμμα {letter} έχει ήδη χρησιμοποιηθεί."
            return
        if letter in VOWELS:
            self.message = "Φωνήεν! Για φωνήεν πάτα V (αγορά) και μετά γράψε το φωνήεν."
            return
        self.used_letters.add(letter)
        count = self.word.count(letter)
        if count == 0:
            self.message = f"❌ Δεν υπάρχει το {letter}. Πας στον επόμενο."
            self.next_turn()
            return
        if self.last_spin and self.last_spin[1] == "BONUS":
            gained = 300 * count
        else:
            points = self.last_spin[1] if self.last_spin and isinstance(self.last_spin[1], int) else 0
            gained = points * count
        self.current_player()["score"] += gained
        self.guessed.add(letter)
        if all_revealed(self.word, self.guessed):
            self.current_player()["score"] += WIN_BONUS
            self.message = f"🏆 ΜΠΡΑΒΟ! Η λέξη ήταν: {self.word} (+{WIN_BONUS} bonus). Τέλος γύρου! (N)"
            self.phase = "END"
            return
        self.message = f"✅ Υπάρχει {count} φορά(ές) το {letter}! +{gained} πόντοι. Συνέχισε ή λύσε (ENTER)."

    def buy_vowel(self):
        if self.phase != "ACTION" or self.solve_mode:
            return
        p = self.current_player()
        if p["score"] < VOWEL_COST:
            self.message = f"Δεν έχεις αρκετούς πόντους (χρειάζεσαι {VOWEL_COST})."
            return
        self.await_vowel = True
        self.message = "🔤 Αγορά φωνήεντος: γράψε A / E / I / O / U / Y."

    def guess_vowel(self, letter):
        if self.phase != "ACTION" or not self.await_vowel or self.solve_mode:
            return
        letter = letter.upper()
        if not letter.isalpha():
            return
        if letter not in VOWELS:
            self.message = "Αυτό δεν είναι φωνήεν. Δοκίμασε A / E / I / O / U /  Y."
            return
        if letter in self.used_letters:
            self.await_vowel = False
            self.message = f"Το {letter} έχει ήδη χρησιμοποιηθεί. Συνέχισε."
            return
        p = self.current_player()
        p["score"] -= VOWEL_COST
        self.used_letters.add(letter)
        count = self.word.count(letter)
        self.await_vowel = False
        if count == 0:
            self.message = f"❌ Δεν υπάρχει το {letter}. -{VOWEL_COST} πόντοι και πας στον επόμενο."
            self.next_turn()
            return
        self.guessed.add(letter)
        if all_revealed(self.word, self.guessed):
            p["score"] += WIN_BONUS
            self.message = f"🏆 ΜΠΡΑΒΟ! Η λέξη ήταν: {self.word} (+{WIN_BONUS} bonus). Τέλος γύρου! (N)"
            self.phase = "END"
            return
        self.message = f"✅ Υπάρχει {count} φορά(ές) το {letter}! (αγορά -{VOWEL_COST}). Συνέχισε!"

    def start_solve(self):
        if self.phase != "ACTION":
            return
        self.solve_mode = True
        self.solve_text = ""
        self.await_vowel = False
        self.message = "✍️ Γράψε τη λύση και πάτα ENTER (ESC για ακύρωση)."

    def submit_solve(self):
        if not self.solve_mode:
            return
        attempt = self.solve_text.strip().upper().replace(" ", "")
        target = self.word.replace(" ", "")
        self.solve_mode = False

        if attempt == target:
            self.current_player()["score"] += WIN_BONUS
            self.message = f"🏆 ΣΩΣΤΟ! Η λέξη ήταν: {self.word} (+{WIN_BONUS} bonus). Τέλος γύρου! (N)"
            self.phase = "END"
        else:
            self.message = f"❌ ΛΑΘΟΣ! Η σωστή λέξη ήταν: {self.word}. Πας στον επόμενο."
            self.next_turn()

    def cancel_solve(self):
        if self.solve_mode:
            self.solve_mode = False
            self.solve_text = ""
            self.message = "Ακύρωση λύσης. Συνέχισε με γράμμα ή ENTER για λύση."

    def reset_round(self):
        self.word = random.choice(WORDS)
        self.guessed = set()
        self.used_letters = set()
        self.phase = "SPIN"
        self.last_spin = None
        self.solve_mode = False
        self.solve_text = ""
        self.await_vowel = False
        self.message = f"Νέος γύρος! Σειρά έχει ο {self.current_player()['name']} — Πάτα SPACE για SPIN."
        
def main():
    pygame.init()
    pygame.display.set_caption("Ο Τροχός των Χριστουγέννων 2060")
    info = pygame.display.Info()
    WIDTH, HEIGHT = info.current_w, info.current_h
    screen = pygame.display.set_mode((WIDTH, HEIGHT), pygame.NOFRAME)
    try:
        bg_image = pygame.image.load(BG_PATH).convert()
    except Exception as e:
        print("❌ Δεν μπόρεσα να φορτώσω την εικόνα:", BG_PATH)
        print("Σφάλμα:", e)
        pygame.quit()
        return
    bg_image = pygame.transform.smoothscale(bg_image, (WIDTH, HEIGHT))
    final_image = None
    try:
        final_image = pygame.image.load(FINAL_IMAGE_PATH).convert_alpha()
    except Exception as e:
        print("❌ Δεν μπόρεσα να φορτώσω την τελική εικόνα:", FINAL_IMAGE_PATH)
        print("Σφάλμα:", e)
        final_image = None
    clock = pygame.time.Clock()

    font_title = pygame.font.SysFont("Segoe UI", 56, bold=True)
    font_players = pygame.font.SysFont("Segoe UI", 26, bold=True)
    font_players_turn = pygame.font.SysFont("Segoe UI", 30, bold=True)
    font_header = pygame.font.SysFont("Segoe UI", 34, bold=True)
    font_header_small = pygame.font.SysFont("Segoe UI", 28, bold=True)
    font_med = pygame.font.SysFont("Segoe UI", 22, bold=True)
    font_small = pygame.font.SysFont("Segoe UI", 18, bold=True)
    font_wheel = pygame.font.SysFont("Segoe UI", 18, bold=True)

    wheel_radius = 260
    wheel_surf_size = wheel_radius * 2 + 40
    wheel_base = pygame.Surface((wheel_surf_size, wheel_surf_size), pygame.SRCALPHA)
    wc = wheel_surf_size // 2

    n_slices = len(WHEEL_SLICES)
    slice_angle = 2 * math.pi / n_slices

    NEON_PALETTE = [
        (70, 240, 255),
        (140, 110, 255),
        (255, 90, 190),
        (255, 160, 80),
        (90, 255, 190),
        (255, 110, 110),]

    def tint(col, k):
        return (
            max(0, min(255, int(col[0] * k))),
            max(0, min(255, int(col[1] * k))),
            max(0, min(255, int(col[2] * k))),)

    slice_colors = []
    for i in range(n_slices):
        base = NEON_PALETTE[i % len(NEON_PALETTE)]
        col = tint(base, 1.00 if i % 2 == 0 else 0.72)
        slice_colors.append(col)

    def render_wheel():
        wheel_base.fill((0, 0, 0, 0))
        for i, (label, _) in enumerate(WHEEL_SLICES):
            start = i * slice_angle
            end = start + slice_angle
            pts = [(wc, wc)]
            steps = 30
            for s in range(steps + 1):
                a = start + (end - start) * (s / steps)
                x = wc + math.cos(a) * wheel_radius
                y = wc + math.sin(a) * wheel_radius
                pts.append((x, y))
            pygame.draw.polygon(wheel_base, slice_colors[i], pts)
            shade = pygame.Surface((wheel_surf_size, wheel_surf_size), pygame.SRCALPHA)
            pygame.draw.polygon(shade, (0, 0, 0, 55), pts)
            pygame.draw.polygon(shade, (255, 255, 255, 25), pts, 1)
            wheel_base.blit(shade, (0, 0))
            pygame.draw.polygon(wheel_base, (15, 15, 15), pts, 2)
            mid = start + slice_angle / 2
            tx = wc + math.cos(mid) * (wheel_radius * 0.62)
            ty = wc + math.sin(mid) * (wheel_radius * 0.62)
            rect = font_wheel.render(label, True, GOLD).get_rect(center=(tx, ty))
            draw_gold_text(wheel_base, label, font_wheel, rect.x, rect.y, GOLD)

        pygame.draw.circle(wheel_base, (10, 10, 10), (wc, wc), wheel_radius, 6)

    render_wheel()
    cached_deg = None
    cached_img = None

    def get_wheel_image(rotation):
        nonlocal cached_deg, cached_img
        deg = -math.degrees(rotation)
        qdeg = int(round(deg / 2) * 2)
        if cached_img is None or qdeg != cached_deg:
            cached_deg = qdeg
            cached_img = pygame.transform.rotate(wheel_base, qdeg)
        return cached_img
    glow_outer = make_glow_circle(wheel_radius + 80, NEON_CYAN, strength=120)
    glow_inner = make_glow_circle(wheel_radius + 40, NEON_PURPLE, strength=110)

    CENTER_X = WIDTH // 2
    cx = (WIDTH // 2) - 60
    cy = (HEIGHT // 2) + 40
    pointer_y = cy - wheel_radius - 18
    btn_w = 220
    btn_h = 52
    btn_gap = 13
    total_h = btn_h * 3 + btn_gap * 2
    start_y = (HEIGHT // 2) - (total_h // 2) - 175
    left_x = 170
    btn_spin = pygame.Rect(left_x, start_y, btn_w, btn_h)
    btn_vowel = pygame.Rect(left_x, start_y + (btn_h + btn_gap), btn_w, btn_h)
    btn_new_round = pygame.Rect(left_x, start_y + 2 * (btn_h + btn_gap), btn_w, btn_h)
    btn_exit = pygame.Rect(WIDTH - 60, 20, 40, 40)
    right_margin = 40
    right_w = max(360, int(WIDTH * 0.32))
    right_x = WIDTH - right_w - right_margin
    base_y = int(HEIGHT * 0.20)
    players_panel = pygame.Rect(right_x, base_y, right_w, 260)
    word_panel = pygame.Rect(right_x, players_panel.bottom + 18, right_w, 175)
    used_panel = pygame.Rect(right_x, word_panel.bottom + 14, right_w, 110)
    msg_panel_width = WIDTH // 2
    msg_panel_x = (WIDTH - msg_panel_width) // 2
    msg_panel = pygame.Rect(msg_panel_x, HEIGHT - 70, msg_panel_width, 45)
    snow_count = int((WIDTH * HEIGHT) / (1920 * 1080) * 220)
    snow_count = max(140, min(320, snow_count))
    snowflakes = [Snowflake(WIDTH, HEIGHT) for _ in range(snow_count)]
    wind = 0.0
    finish_mode = False
    finish_timer = 0.0
    finish_show_image = False
    game = Game()
    t = 0.0
    running = True
    while running:
        dt = clock.tick(FPS) / 1000.0
        t += dt
        if not finish_mode:
            game.update(dt)
        wind = 40 * math.sin(t * 0.35)
        for flake in snowflakes:
            flake.update(dt, WIDTH, HEIGHT, wind)
        if finish_mode:
            finish_timer += dt
            if finish_timer >= 7.0:
                finish_show_image = True

        mx, my = pygame.mouse.get_pos()
        exit_hover = btn_exit.collidepoint(mx, my)

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            if event.type == pygame.KEYDOWN:
                # Finish mode: SPACE/ESC -> νέο γύρο
                if finish_mode:
                    if event.key in (pygame.K_SPACE, pygame.K_ESCAPE):
                        finish_mode = False
                        game.reset_round()
                    continue
                if event.key == pygame.K_ESCAPE:
                    if game.solve_mode:
                        game.cancel_solve()
                if event.key == pygame.K_SPACE:
                    if game.phase == "SPIN" and not game.spinning:
                        game.start_spin()
                if event.key == pygame.K_n and game.phase == "END":
                    finish_mode = True
                    finish_timer = 0.0
                    finish_show_image = False
                if game.phase == "ACTION":
                    if event.key == pygame.K_RETURN:
                        if not game.solve_mode:
                            game.start_solve()
                        else:
                            game.submit_solve()
                    if event.key == pygame.K_v:
                        game.buy_vowel()
                    if game.await_vowel:
                        ch = event.unicode.upper()
                        if ch and ch.isalpha():
                            game.guess_vowel(ch)
                        continue
                    if game.solve_mode:
                        if event.key == pygame.K_BACKSPACE:
                            game.solve_text = game.solve_text[:-1]
                        else:
                            ch = event.unicode
                            if ch and (ch.isalpha() or ch == " "):
                                game.solve_text += ch.upper()
                        continue
                    ch = event.unicode.upper()
                    if ch and ch.isalpha():
                        game.guess_letter(ch)
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                mx, my = event.pos
                if btn_exit.collidepoint(mx, my):
                    running = False
                if finish_mode:
                    continue
                if btn_spin.collidepoint(mx, my):
                    if game.phase == "SPIN" and not game.spinning:
                        game.start_spin()
                if btn_vowel.collidepoint(mx, my):
                    if game.phase == "ACTION" and not game.spinning:
                        game.buy_vowel()
                if btn_new_round.collidepoint(mx, my):
                    pass


        screen.blit(bg_image, (0, 0))
        for flake in snowflakes:
            flake.draw(screen)
        if finish_mode:
            overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            overlay.fill((0, 0, 0, 160))
            screen.blit(overlay, (0, 0))
            if not finish_show_image:
                box_w = int(WIDTH * 0.70)
                box_h = 160
                box = pygame.Rect((WIDTH - box_w) // 2, (HEIGHT - box_h) // 2, box_w, box_h)
                pygame.draw.rect(screen, (20, 20, 20), box, border_radius=22)
                col = neon_cycle_color(t, speed=1.1)
                pulse = 0.5 + 0.5 * math.sin(t * 6.0)
                bw = 4 + int(3 * pulse)
                pygame.draw.rect(screen, col, box, bw, border_radius=22)
                draw_neon_text_center(screen, "< ALGORITHNICS FOR EVER >", font_title, WIDTH // 2, HEIGHT // 2, t)
                winner = max(game.players, key=lambda p: p["score"])
                draw_gold_text_center(screen, f"ΝΙΚΗΤΗΣ: {winner['name']}!", font_header, WIDTH // 2, HEIGHT // 2 + 62, GOLD)

            else:
                if final_image:
                    iw, ih = final_image.get_size()
                    scale = min(WIDTH / iw, HEIGHT / ih)
                    nw, nh = int(iw * scale), int(ih * scale)
                    img = pygame.transform.smoothscale(final_image, (nw, nh))
                    screen.blit(img, img.get_rect(center=(WIDTH // 2, HEIGHT // 2)))
                else:
                    draw_gold_text_center(screen, "❌ Δεν βρέθηκε η τελική εικόνα!", font_title, WIDTH // 2, HEIGHT // 2, GOLD)

                draw_gold_text_center(screen, "Πάτα SPACE για νέο γύρο", font_med, WIDTH // 2, HEIGHT - 70, GOLD)

            pygame.display.flip()
            continue
        
        draw_exit_button(screen, btn_exit, font_med, hover=exit_hover)
        draw_neon_text_center(screen, "Ο ΤΡΟΧΟΣ ΤΩΝ ΧΡΙΣΤΟΥΓΕΝΝΩΝ 2060", font_title, CENTER_X, 36, t)
        spin_enabled = (game.phase == "SPIN" and not game.spinning)
        vowel_enabled = (game.phase == "ACTION" and not game.spinning and not game.solve_mode)
        new_enabled = (game.phase == "END")
        draw_image_button(screen, bg_image, btn_spin, "SPIN (SPACE)", font_med, enabled=spin_enabled)
        draw_image_button(screen, bg_image, btn_vowel, f"ΦΩΝΗΕΝ (V) -{VOWEL_COST}", font_med, enabled=vowel_enabled)
        draw_image_button(screen, bg_image, btn_new_round, "ΝΕΟΣ ΓΥΡΟΣ (N)", font_med, enabled=new_enabled)
        ppointer = [(cx, pointer_y), (cx - 18, pointer_y + 34), (cx + 18, pointer_y + 34)]
        pygame.draw.polygon(screen, (255, 90, 90), ppointer)
        pygame.draw.polygon(screen, (255, 255, 255), ppointer, 1)
        pg = pygame.Surface((60, 60), pygame.SRCALPHA)
        pygame.draw.circle(pg, (*NEON_CYAN, 55), (30, 30), 22, 3)
        screen.blit(pg, (cx - 30, pointer_y + 8))
        pulse = 0.5 + 0.5 * math.sin(t * 2.0)
        a_outer = int(80 + 70 * pulse)
        a_inner = int(70 + 60 * (1 - pulse))
        g1 = glow_outer.copy()
        g2 = glow_inner.copy()
        g1.set_alpha(a_outer)
        g2.set_alpha(a_inner)
        r1 = g1.get_rect(center=(cx, cy))
        r2 = g2.get_rect(center=(cx, cy))
        screen.blit(g1, r1.topleft)
        screen.blit(g2, r2.topleft)
        draw_neon_ring(screen, (cx, cy), wheel_radius, t)
        draw_sparkles(screen, (cx, cy), wheel_radius, t, count=16)
        wheel_img = get_wheel_image(game.rotation)
        wheel_rect = wheel_img.get_rect(center=(cx, cy))
        screen.blit(wheel_img, wheel_rect)
        hub_col = neon_cycle_color(t, speed=1.2)
        hub_pulse = 0.5 + 0.5 * math.sin(t * 6.0)
        hub_r = int(14 + 3 * hub_pulse)
        hub_glow_r = int(34 + 10 * hub_pulse)
        hub_glow = pygame.Surface((hub_glow_r * 2 + 2, hub_glow_r * 2 + 2), pygame.SRCALPHA)
        pygame.draw.circle(hub_glow, (*hub_col, int(70 + 120 * hub_pulse)), (hub_glow_r + 1, hub_glow_r + 1), hub_glow_r)
        pygame.draw.circle(hub_glow, (*hub_col, int(40 + 80 * (1 - hub_pulse))), (hub_glow_r + 1, hub_glow_r + 1), hub_glow_r - 10, 3)
        screen.blit(hub_glow, hub_glow.get_rect(center=(cx, cy)).topleft)
        pygame.draw.circle(screen, hub_col, (cx, cy), hub_r)
        pygame.draw.circle(screen, (10, 10, 10), (cx, cy), hub_r, 2)
        draw_image_panel(screen, bg_image, players_panel, alpha=130, radius=18)
        draw_image_panel(screen, bg_image, word_panel, alpha=130, radius=18)
        draw_image_panel(screen, bg_image, used_panel, alpha=130, radius=18)
        draw_image_panel(screen, bg_image, msg_panel, alpha=140, radius=18)
        draw_gold_text_center(screen, "ΛΕΞΗ", font_header, word_panel.centerx, word_panel.y + 34, GOLD)
        masked = mask_word(game.word, game.guessed)
        draw_gold_text_center(screen, masked, font_players_turn, word_panel.centerx, word_panel.y + 105, GOLD)
        if game.solve_mode:
            solve_panel = pygame.Rect(word_panel.x + 12, word_panel.bottom - 80, word_panel.w - 24, 65)
            draw_image_panel(screen, bg_image, solve_panel, alpha=150, radius=14)
            draw_gold_text(screen, "ΛΥΣΗ:", font_med, solve_panel.x + 12, solve_panel.y + 8, GOLD)
            draw_gold_text(screen, game.solve_text, font_med, solve_panel.x + 12, solve_panel.y + 34, GOLD)
        draw_gold_text_center(screen, "ΠΑΙΚΤΗΣ : ΠΟΝΤΟΙ", font_header, players_panel.centerx, players_panel.y + 34, GOLD)
        y = players_panel.y + 78
        for i, p in enumerate(game.players):
            is_turn = (i == game.turn and game.phase != "END")
            prefix = "➡ " if is_turn else "   "
            line = f"{prefix}{p['name']}: {p['score']} πόντοι"
            draw_gold_text(screen, line, font_players_turn if is_turn else font_players, players_panel.x + 16, y, GOLD)
            y += 56
        draw_gold_text_center(screen, "ΕΠΙΛΕΓΜΕΝΑ ΓΡΑΜΜΑΤΑ", font_header_small, used_panel.centerx, used_panel.y + 24, GOLD)
        lines = wrap_letters(game.used_letters, max_per_line=18)
        yy = used_panel.y + 52
        for line in lines[:2]:
            draw_gold_text(screen, line if line else "-", font_small, used_panel.x + 14, yy, GOLD)
            yy += 26
        draw_gold_text_center(screen, game.message, font_med, msg_panel.centerx, msg_panel.centery)
        for flake in snowflakes[: max(20, snow_count // 8)]:
            pygame.draw.circle(screen, (*SNOW_WHITE, 200), (int(flake.x), int(flake.y)), int(flake.r + 1))
        pygame.display.flip()
    pygame.quit()

if __name__ == "__main__":
    main()


❌
