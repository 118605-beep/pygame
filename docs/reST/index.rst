Pygame Front Pageimport pygame
import random
import math

# Initialize Pygame
pygame.init()

# Setup Screen
WIDTH, HEIGHT = 900, 700
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Animal Squad vs Zombie Horde")
clock = pygame.time.Clock()

# Colors
GREEN = (40, 120, 50)
CRATER_COLOR = (80, 60, 40)
WHITE = (255, 255, 255)
POISON_GREEN = (50, 205, 50)

# Animal Configuration
ANIMALS = {
    "Rat": {"color": (128, 128, 128), "offset": (-20, 20)},
    "Shark": {"color": (70, 130, 180), "offset": (0, 0)},
    "Cow": {"color": (139, 69, 19), "offset": (20, 20)},
    "Eagle": {"color": (205, 133, 63)}
}

# Projectiles and FX Lists
projectiles = []
craters = []
particles = []

class Squad:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.base_speed = 4
        self.speed = self.base_speed
        self.health = 100
        self.max_health = 100
        
        # Ability Timers
        self.moo_timer = 0
        self.chomp_cooldown = 0
        self.is_chomping = False
        self.chomp_timer = 0
        self.rat_shoot_timer = 0
        
        # Eagle Position Tracker
        self.eagle_angle = 0

    def update(self, keys, zombies):
        # 1. Movement Handling
        dx, dy = 0, 0
        if keys[pygame.K_w]: dy -= 1
        if keys[pygame.K_s]: dy += 1
        if keys[pygame.K_a]: dx -= 1
        if keys[pygame.K_d]: dx += 1
        
        if dx != 0 or dy != 0:
            dist = math.hypot(dx, dy)
            self.x += (dx / dist) * self.speed
            self.y += (dy / dist) * self.speed

        # Bound squad to screen
        self.x = max(30, min(WIDTH - 30, self.x))
        self.y = max(30, min(HEIGHT - 30, self.y))

        # 2. Cow Moo Ability (Speed Boost)
        if keys[pygame.K_LSHIFT] and self.moo_timer == 0:
            self.moo_timer = 180 # Boost lasts/cooldown ticks down
            self.speed = self.base_speed * 1.75
        
        if self.moo_timer > 0:
            self.moo_timer -= 1
            if self.moo_timer < 120: # Speed boost lasts 1 second (60 frames)
                self.speed = self.base_speed

        # 3. Shark Chomp Ability
        if self.chomp_cooldown > 0:
            self.chomp_cooldown -= 1
            
        if keys[pygame.K_SPACE] and self.chomp_cooldown == 0 and not self.is_chomping:
            self.is_chomping = True
            self.chomp_timer = 30 # 0.5 second animation
            
        if self.is_chomping:
            self.chomp_timer -= 1
            if self.chomp_timer == 0:
                self.is_chomping = False
                self.chomp_cooldown = 120 # 2 second cooldown
                # Create Crater & Area Damage
                craters.append({"x": self.x, "y": self.y, "radius": 70, "duration": 90})
                for z in zombies:
                    if math.hypot(z.x - self.x, z.y - self.y) < 70:
                        z.health -= 50

        # 4. Rat Poison Shooting
        self.rat_shoot_timer += 1
        if self.rat_shoot_timer >= 25: # Fire every ~0.4 seconds
            self.rat_shoot_timer = 0
            if zombies:
                target = min(zombies, key=lambda z: math.hypot(z.x - self.x, z.y - self.y))
                if math.hypot(target.x - self.x, target.y - self.y) < 300:
                    ang = math.atan2(target.y - self.y, target.x - self.x)
                    projectiles.append({"x": self.x + ANIMALS["Rat"]["offset"][0], "y": self.y + ANIMALS["Rat"]["offset"][1], "vx": math.cos(ang)*8, "vy": math.sin(ang)*8})

        # 5. Cow Kick Guard (Passive)
        for z in zombies:
            # If zombie is directly behind movement direction and close, kick them back
            if math.hypot(z.x - self.x, z.y - self.y) < 45:
                z.x += (z.x - self.x) * 0.5
                z.y += (z.y - self.y) * 0.5
                z.health -= 2 # Small contact kick damage

        # 6. Eagle AI Behavior
        self.eagle_angle += 0.05

    def draw(self, surface):
        # Draw Cow and Rat relative to the central anchor point
        for name, data in ANIMALS.items():
            if name == "Eagle" or (name == "Shark" and self.is_chomping):
                continue
            pos = (int(self.x + data["offset"][0]), int(self.y + data["offset"][1]))
            pygame.draw.circle(surface, data["color"], pos, 14)

        # Draw Shark Jump sequence
        if self.is_chomping:
            # Scale visual radius based on jump arc height
            jump_offset = math.sin((self.chomp_timer / 30.0) * math.pi) * 20
            pygame.draw.circle(surface, ANIMALS["Shark"]["color"], (int(self.x), int(self.y - jump_offset)), int(16 + jump_offset * 0.5))
        else:
            pygame.draw.circle(surface, ANIMALS["Shark"]["color"], (int(self.x), int(self.y)), 16)

        # Draw Flying Eagle circling overhead
        ex = self.x + math.cos(self.eagle_angle) * 60
        ey = self.y + math.sin(self.eagle_angle) * 60 - 40 # Elevated shadow effect offset
        pygame.draw.polygon(surface, ANIMALS["Eagle"]["color"], [(ex, ey-10), (ex-12, ey+8), (ex+12, ey+8)])

class Zombie:
    def __init__(self, x, y, z_type="normal"):
        self.x = x
        self.y = y
        self.z_type = z_type
        self.poison_ticks = 0
        
        if z_type == "normal":
            self.color = (30, 100, 30)
            self.radius = 12
            self.speed = 2.0
            self.health = 20
            self.damage = 0.2
        elif z_type == "fast":
            self.color = (120, 110, 20)
            self.radius = 10
            self.speed = 3.5
            self.health = 12
            self.damage = 0.15
        elif z_type == "strong":
            self.color = (140, 20, 20)
            self.radius = 18
            self.speed = 1.2
            self.health = 60
            self.damage = 0.5
        elif z_type == "boss":
            self.color = (0, 0, 0)
            self.radius = 35
            self.speed = 0.8
            self.health = 400
            self.damage = 1.0
            self.summon_timer = 0

    def update(self, target_x, target_y, zombies_list):
        # Handle Poison Status Effect
        if self.poison_ticks > 0:
            self.health -= 0.15
            self.poison_ticks -= 1
            if random.random() < 0.1:
                particles.append({"x": self.x, "y": self.y, "color": POISON_GREEN, "dur": 15})

        # Track Target Location
        dx = target_x - self.x
        dy = target_y - self.y
        dist = math.hypot(dx, dy)
        if dist > 0:
            self.x += (dx / dist) * self.speed
            self.y += (dy / dist) * self.speed

        # Boss Unique Ability: Global Wave Summoner
        if self.z_type == "boss":
            self.summon_timer += 1
            if self.summon_timer >= 150: # Summon every 2.5 seconds
                self.summon_timer = 0
                for _ in range(3):
                    zombies_list.append(Zombie(self.x + random.randint(-50, 50), self.y + random.randint(-50, 50), random.choice(["normal", "fast", "strong"])))

    def draw(self, surface):
        pygame.draw.circle(surface, self.color, (int(self.x), int(self.y)), self.radius)
        if self.poison_ticks > 0: # Green poison glow border
            pygame.draw.circle(surface, POISON_GREEN, (int(self.x), int(self.y)), self.radius + 2, 2)

def main():
    squad = Squad(WIDTH // 2, HEIGHT // 2)
    zombies = []
    
    score = 0
    spawn_timer = 0
    boss_timer = 0
    game_over = False

    running = True
    while running:
        clock.tick(60)
        keys = pygame.key.get_pressed()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            if event.type == pygame.KEYDOWN and game_over and event.key == pygame.K_r:
                main()
                return

        if not game_over:
            score += 1
            
            # --- Enemy Spawning Manager ---
            spawn_timer += 1
            boss_timer += 1
            
            if spawn_timer >= 50:
                spawn_timer = 0
                # Spawn around screen boundary borders
                edge = random.choice(["T", "B", "L", "R"])
                sx = random.randint(0, WIDTH) if edge in ["T", "B"] else (-20 if edge == "L" else WIDTH + 20)
                sy = random.randint(0, HEIGHT) if edge in ["L", "R"] else (-20 if edge == "T" else HEIGHT + 20)
                
                # Pick standard types dynamically
                zombies.append(Zombie(sx, sy, random.choice(["normal", "fast", "strong"])))

            if boss_timer >= 1500: # Spawn a Mega-Boss every 25 seconds
                boss_timer = 0
                zombies.append(Zombie(random.choice([-40, WIDTH+40]), random.choice([-40, HEIGHT+40]), "boss"))

            # --- Update Simulation Layers ---
            squad.update(keys, zombies)

            # Eagle Strike Automator
            if score % 90 == 0 and zombies: # Every 1.5 seconds eagle pecks nearest target
                target = min(zombies, key=lambda z: math.hypot(z.x - squad.x, z.y - squad.y))
                target.health -= 15
                # Spawn feather hit particles
                for _ in range(5):
                    particles.append({"x": target.x, "y": target.y, "color": ANIMALS["Eagle"]["color"], "dur": 12})

            # Update Poison Projectiles
            for p in projectiles[:]:
                p["x"] += p["vx"]
                p["y"] += p["vy"]
                # Collision handling
                for z in zombies:
                    if math.hypot(z.x - p["x"], z.y - p["y"]) < z.radius + 4:
                        z.poison_ticks = 180 # 3 seconds of continuous poison ticks
                        if p in projectiles: projectiles.remove(p)
                        break
                # Clean up out of bounds darts
                if p["x"] < 0 or p["x"] > WIDTH or p["y"] < 0 or p["y"] > HEIGHT:
                    if p in projectiles: projectiles.remove(p)

            # Process Zombie AI Core loops
            for z in zombies[:]:
                z.update(squad.x, squad.y, zombies)
                
                # Damage contact calculations
                if math.hypot(z.x - squad.x, z.y - squad.y) < z.radius + 20:
                    squad.health -= z.damage
                    if squad.health <= 0:
                        game_over = True
                
                if z.health <= 0:
                    zombies.remove(z)

            # Manage Visual Ground Craters
            for c in craters[:]:
                c["duration"] -= 1
                if c["duration"] <= 0:
                    craters.remove(c)

            # Visual Particles processing
            for pt in particles[:]:
                pt["dur"] -= 1
                pt["y"] += random.choice([-1, 1])
                if pt["dur"] <= 0: particles.remove(pt)

        # --- Graphics Engine Render Stack ---
        screen.fill(GREEN)

        # 1. Background Environmental Elements (Craters)
        for c in craters:
            pygame.draw.circle(screen, CRATER_COLOR, (int(c["x"]), int(c["y"])), c["radius"])
            pygame.draw.circle(screen, (50, 40, 30), (int(c["x"]), int(c["y"])), c["radius"] - 6, 2)

        # 2. Combat Entities & Action Particles
        for pt in particles:
            pygame.draw.circle(screen, pt["color"], (int(pt["x"]), int(pt["y"])), 3)
            
        for p in projectiles:
            pygame.draw.circle(screen, POISON_GREEN, (int(p["x"]), int(p["y"])), 4)

        for z in zombies:
            z.draw(screen)

        squad.draw(screen)

        # 3. Heads-Up Display UI Overlay
        pygame.draw.rect(screen, (20, 20, 20), (15, 15, 260, 95))
        
        # Draw Health Bar
        pygame.draw.rect(screen, (150, 0, 0), (25, 30, 200, 15))
        if squad.health > 0:
            pygame.draw.rect(screen, (0, 200, 50), (25, 30, int(squad.health * 2), 15))
            
        font = pygame.font.SysFont("Impact", 18)
        lbl_hp = font.render(f"SQUAD HEALTH: {int(squad.health)}%", True, WHITE)
        lbl_score = font.render(f"SURVIVAL SCORE: {score // 10}", True, WHITE)
        lbl_cd = font.render(f"SHARK CHOMP (SPACE): {'READY' if squad.chomp_cooldown == 0 else 'COOLDOWN'}", True, (100, 200, 255) if squad.chomp_cooldown == 0 else (200, 100, 100))
        
        screen.blit(lbl_hp, (25, 10))
        screen.blit(lbl_score, (25, 50))
        screen.blit(lbl_cd, (25, 75))

        # Game Over Interface
        if game_over:
            overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            overlay.fill((0, 0, 0, 200))
            screen.blit(overlay, (0, 0))
            font_go = pygame.font.SysFont("Impact", 60)
            go_text = font_go.render("THE SQUAD FELL!", True, (220, 20, 20))
            restart_text = font.render("Press 'R' to Deploy Next Squad", True, WHITE)
            screen.blit(go_text, (WIDTH // 2 - 180, HEIGHT // 2 - 40))
            screen.blit(restart_text, (WIDTH // 2 - 120, HEIGHT // 2 + 30))

        pygame.display.flip()

    pygame.quit()

if __name__ == "__main__":
    main()=================

.. toctree::
   :maxdepth: 2
   :glob:
   :hidden:

   ref/*
   tut/*
   tut/en/**/*
   tut/ko/**/*
   c_api
   filepaths
   logos

Quick start
-----------

Welcome to pygame! Once you've got pygame installed (:code:`pip install pygame` or
:code:`pip3 install pygame` for most people), the next question is how to get a game
loop running. Pygame, unlike some other libraries, gives you full control of program
execution. That freedom means it is easy to mess up in your initial steps.

Here is a good example of a basic setup (opens the window, updates the screen, and handles events)--

.. literalinclude:: ref/code_examples/base_script.py

Here is a slightly more fleshed out example, which shows you how to move something
(a circle in this case) around on screen--

.. literalinclude:: ref/code_examples/base_script_example.py

For more in depth reference, check out the :ref:`tutorials-reference-label`
section below, check out a video tutorial (`I'm a fan of this one
<https://www.youtube.com/watch?v=AY9MnQ4x3zk>`_), or reference the API
documentation by module.

Documents
---------

`Readme`_
  Basic information about pygame: what it is, who is involved, and where to find it.

`Install`_
  Steps needed to compile pygame on several platforms.
  Also help on finding and installing prebuilt binaries for your system.

:doc:`filepaths`
  How pygame handles file system paths.

:doc:`Pygame Logos <logos>`
   The logos of Pygame in different resolutions.


`LGPL License`_
  This is the license pygame is distributed under.
  It provides for pygame to be distributed with open source and commercial software.
  Generally, if pygame is not changed, it can be used with any type of program.

.. _tutorials-reference-label:

Tutorials
---------

:doc:`Introduction to Pygame <tut/PygameIntro>`
  An introduction to the basics of pygame.
  This is written for users of Python and appeared in volume two of the Py magazine.

:doc:`Import and Initialize <tut/ImportInit>`
  The beginning steps on importing and initializing pygame.
  The pygame package is made of several modules.
  Some modules are not included on all platforms.

:doc:`How do I move an Image? <tut/MoveIt>`
  A basic tutorial that covers the concepts behind 2D computer animation.
  Information about drawing and clearing objects to make them appear animated.

:doc:`Chimp Tutorial, Line by Line <tut/ChimpLineByLine>`
  The pygame examples include a simple program with an interactive fist and a chimpanzee.
  This was inspired by the annoying flash banner of the early 2000s.
  This tutorial examines every line of code used in the example.

:doc:`Sprite Module Introduction <tut/SpriteIntro>`
  Pygame includes a higher level sprite module to help organize games.
  The sprite module includes several classes that help manage details found in almost all games types.
  The Sprite classes are a bit more advanced than the regular pygame modules,
  and need more understanding to be properly used.

:doc:`Surfarray Introduction <tut/SurfarrayIntro>`
  Pygame used the NumPy python module to allow efficient per pixel effects on images.
  Using the surface arrays is an advanced feature that allows custom effects and filters.
  This also examines some of the simple effects from the pygame example, arraydemo.py.

:doc:`Camera Module Introduction <tut/CameraIntro>`
  Pygame, as of 1.9, has a camera module that allows you to capture images,
  watch live streams, and do some basic computer vision.
  This tutorial covers those use cases.

:doc:`Newbie Guide <tut/newbieguide>`
  A list of thirteen helpful tips for people to get comfortable using pygame.

:doc:`Making Games Tutorial <tut/MakeGames>`
  A large tutorial that covers the bigger topics needed to create an entire game.

:doc:`Display Modes <tut/DisplayModes>`
  Getting a display surface for the screen.

:doc:`한국어 튜토리얼 (Korean Tutorial) <tut/ko/빨간블록 검은블록/개요>`
  빨간블록 검은블록


Reference
---------

:ref:`genindex`
  A list of all functions, classes, and methods in the pygame package.

:doc:`ref/bufferproxy`
  An array protocol view of surface pixels

:doc:`ref/color`
  Color representation.

:doc:`ref/cursors`
  Loading and compiling cursor images.

:doc:`ref/display`
  Configure the display surface.

:doc:`ref/draw`
  Drawing simple shapes like lines and ellipses to surfaces.

:doc:`ref/event`
  Manage the incoming events from various input devices and the windowing platform.

:doc:`ref/examples`
  Various programs demonstrating the use of individual pygame modules.

:doc:`ref/font`
  Loading and rendering TrueType fonts.

:doc:`ref/freetype`
  Enhanced pygame module for loading and rendering font faces.

:doc:`ref/gfxdraw`
  Anti-aliasing draw functions.

:doc:`ref/image`
  Loading, saving, and transferring of surfaces.

:doc:`ref/joystick`
  Manage the joystick devices.

:doc:`ref/key`
  Manage the keyboard device.

:doc:`ref/locals`
  Pygame constants.

:doc:`ref/mixer`
  Load and play sounds

:doc:`ref/mouse`
  Manage the mouse device and display.

:doc:`ref/music`
  Play streaming music tracks.

:doc:`ref/pygame`
  Top level functions to manage pygame.

:doc:`ref/pixelarray`
  Manipulate image pixel data.

:doc:`ref/rect`
  Flexible container for a rectangle.

:doc:`ref/scrap`
  Native clipboard access.

:doc:`ref/sndarray`
  Manipulate sound sample data.

:doc:`ref/sprite`
  Higher level objects to represent game images.

:doc:`ref/surface`
  Objects for images and the screen.

:doc:`ref/surfarray`
  Manipulate image pixel data.

:doc:`ref/tests`
  Test pygame.

:doc:`ref/time`
  Manage timing and framerate.

:doc:`ref/transform`
  Resize and move images.

:doc:`pygame C API <c_api>`
  The C api shared amongst pygame extension modules.

:ref:`search`
  Search pygame documents by keyword.

.. _Readme: ../wiki/about

.. _Install: ../wiki/GettingStarted#Pygame%20Installation

.. _LGPL License: LGPL.txt
