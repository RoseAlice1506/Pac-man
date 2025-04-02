import pygame
import sys
import random
import math
import heapq  # Needed for Dijkstra's algorithm

# Initialize pygame
pygame.init()

# Colors
BLACK = (0, 0, 0)
BLUE = (0, 0, 255)
YELLOW = (255, 255, 0)
WHITE = (255, 255, 255)
RED = (255, 0, 0)
PINK = (255, 192, 203)
CYAN = (0, 255, 255)
ORANGE = (255, 165, 0)

# Game constants
SCREEN_WIDTH = 600
SCREEN_HEIGHT = 650
CELL_SIZE = 40
GRID_WIDTH = 15
GRID_HEIGHT = 15

# Game states
PLAYING = 0
GAME_OVER = 1

# Setup screen
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Pac-Man with Dijkstra's Algorithm")
font = pygame.font.Font(None, 36)

# Game grid (1 = wall, 0 = dot, 2 = eaten dot)
grid = [
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [1,0,0,0,0,0,0,1,0,0,0,0,0,0,1],
    [1,0,1,1,0,1,0,1,0,1,0,1,1,0,1],
    [1,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
    [1,0,1,1,0,1,1,1,1,1,0,1,1,0,1],
    [1,0,0,0,0,0,0,1,0,0,0,0,0,0,1],
    [1,1,1,1,0,1,0,1,0,1,0,1,1,1,1],
    [1,1,1,1,0,1,0,0,0,1,0,1,1,1,1],
    [1,1,1,1,0,1,0,1,0,1,0,1,1,1,1],
    [1,0,0,0,0,0,0,1,0,0,0,0,0,0,1],
    [1,0,1,1,0,1,1,1,1,1,0,1,1,0,1],
    [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [1,0,1,1,0,1,0,1,0,1,0,1,1,0,1],
    [1,0,0,0,0,0,0,1,0,0,0,0,0,0,1],
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
]

# Game objects
pacman = {
    'x': 1,
    'y': 1,
    'direction': 2,  # 0=right,1=down,2=left,3=up
    'mouth_open': False
}

ghosts = [
    {'x':1,'y':13,'color':RED},
    {'x':13,'y':1,'color':PINK},
    {'x':13,'y':13,'color':CYAN},
    {'x':11,'y':11,'color':ORANGE},
]

# Game variables
score = 0
game_state = PLAYING
clock = pygame.time.Clock()
running = True

# Timing controls
pacman_move_delay = 150
ghost_move_delay = 300
mouth_anim_delay = 600
last_pacman_move_time = 0
last_ghost_move_time = 0
last_mouth_anim_time = 0

# DIJKSTRA'S ALGORITHM IMPLEMENTATION
def find_shortest_path(start_x, start_y, target_x, target_y):
    # Directions: right, down, left, up
    directions = [(1,0), (0,1), (-1,0), (0,-1)]
    
    # Priority queue: (distance, x, y)
    heap = []
    heapq.heappush(heap, (0, start_x, start_y))
    
    # Visited and distance tracking
    distances = {(x, y): float('inf') for x in range(GRID_WIDTH) for y in range(GRID_HEIGHT)}
    distances[(start_x, start_y)] = 0
    previous = {}
    
    while heap:
        dist, x, y = heapq.heappop(heap)
        
        # Early exit if we reach target
        if x == target_x and y == target_y:
            path = []
            while (x, y) != (start_x, start_y):
                path.append((x, y))
                x, y = previous[(x, y)]
            path.reverse()
            return path
        
        # Skip if we found a better path already
        if dist > distances[(x, y)]:
            continue
            
        # Explore neighbors
        for dx, dy in directions:
            nx, ny = x + dx, y + dy
            if 0 <= nx < GRID_WIDTH and 0 <= ny < GRID_HEIGHT and grid[ny][nx] != 1:
                new_dist = dist + 1
                if new_dist < distances[(nx, ny)]:
                    distances[(nx, ny)] = new_dist
                    previous[(nx, ny)] = (x, y)
                    heapq.heappush(heap, (new_dist, nx, ny))
    
    return []  # No path found

# Modified ghost movement using Dijkstra's
def move_ghost(ghost):
    path = find_shortest_path(ghost['x'], ghost['y'], pacman['x'], pacman['y'])
    if len(path) > 0:
        next_x, next_y = path[0]  # Move to first step in path
        ghost['x'], ghost['y'] = next_x, next_y

# Original game functions (unchanged)
def move_pacman():
    global score
    dx, dy = [(1,0), (0,1), (-1,0), (0,-1)][pacman['direction']]
    new_x, new_y = pacman['x'] + dx, pacman['y'] + dy
    if 0 <= new_x < GRID_WIDTH and 0 <= new_y < GRID_HEIGHT and grid[new_y][new_x] != 1:
        pacman['x'], pacman['y'] = new_x, new_y
        if grid[new_y][new_x] == 0:
            grid[new_y][new_x] = 2
            score += 10

def draw_pacman():
    x = pacman['x'] * CELL_SIZE + CELL_SIZE // 2
    y = pacman['y'] * CELL_SIZE + CELL_SIZE // 2 + 50
    
    mouth_opening = 45 if pacman['mouth_open'] else 0
    pygame.draw.circle(screen, YELLOW, (x, y), CELL_SIZE // 2)
    
    if pacman['direction'] == 0:
        start_angle = 360 - mouth_opening/2
        end_angle = mouth_opening/2
    elif pacman['direction'] == 3:
        start_angle = 90 - mouth_opening/2
        end_angle = 90 + mouth_opening/2
    elif pacman['direction'] == 2:
        start_angle = 180 - mouth_opening/2
        end_angle = 180 + mouth_opening/2
    else:
        start_angle = 270 - mouth_opening/2
        end_angle = 270 + mouth_opening/2
        
    pygame.draw.arc(screen, BLACK,
                   (x-CELL_SIZE//2, y-CELL_SIZE//2, CELL_SIZE, CELL_SIZE),
                   math.radians(start_angle), math.radians(end_angle), CELL_SIZE//2)
    
    # Draw mouth lines
    for angle in [start_angle, end_angle]:
        end_x = x + math.cos(math.radians(angle)) * CELL_SIZE//2
        end_y = y - math.sin(math.radians(angle)) * CELL_SIZE//2
        pygame.draw.line(screen, BLACK, (x,y), (end_x,end_y), 2)

def draw_ghost(ghost):
    x = ghost['x'] * CELL_SIZE + CELL_SIZE // 2
    y = ghost['y'] * CELL_SIZE + CELL_SIZE // 2 + 50
    pygame.draw.circle(screen, ghost['color'], (x, y), CELL_SIZE // 2)

def reset_game():
    global pacman, ghosts, score, grid, game_state
    pacman = {'x':1, 'y':1, 'direction':2, 'mouth_open':False}
    ghosts = [
        {'x':1,'y':13,'color':RED},
        {'x':13,'y':1,'color':PINK},
        {'x':13,'y':13,'color':CYAN},
        {'x':11,'y':11,'color':ORANGE},
    ]
    score = 0
    grid = [
        [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
        [1,0,0,0,0,0,0,1,0,0,0,0,0,0,1],
        # ... (rest of original grid initialization)
    ]
    game_state = PLAYING

def draw_game_over():
    screen.fill(BLACK)
    texts = [
        ("GAME OVER", 64, RED, SCREEN_HEIGHT//3),
        (f"Score: {score}", 48, WHITE, SCREEN_HEIGHT//2),
        ("Press SPACE to restart", 36, YELLOW, 2*SCREEN_HEIGHT//3)
    ]
    for text, size, color, y_pos in texts:
        font = pygame.font.Font(None, size)
        text_surface = font.render(text, True, color)
        screen.blit(text_surface, (SCREEN_WIDTH//2 - text_surface.get_width()//2, y_pos))

# Main game loop
while running:
    current_time = pygame.time.get_ticks()
    
    # Event handling
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == pygame.KEYDOWN:
            if game_state == PLAYING:
                if event.key == pygame.K_UP: pacman['direction'] = 3
                elif event.key == pygame.K_DOWN: pacman['direction'] = 1
                elif event.key == pygame.K_LEFT: pacman['direction'] = 2
                elif event.key == pygame.K_RIGHT: pacman['direction'] = 0
            elif game_state == GAME_OVER and event.key == pygame.K_SPACE:
                reset_game()
    
    if game_state == PLAYING:
        # Movement updates
        if current_time - last_pacman_move_time > pacman_move_delay:
            move_pacman()
            last_pacman_move_time = current_time
        
        if current_time - last_ghost_move_time > ghost_move_delay:
            for ghost in ghosts:
                move_ghost(ghost)
            last_ghost_move_time = current_time
        
        # Animation
        if current_time - last_mouth_anim_time > mouth_anim_delay:
            pacman['mouth_open'] = not pacman['mouth_open']
            last_mouth_anim_time = current_time
        
        # Drawing
        screen.fill(BLACK)
        
        # Draw maze
        for y in range(GRID_HEIGHT):
            for x in range(GRID_WIDTH):
                if grid[y][x] == 1:
                    pygame.draw.rect(screen, BLUE, (x*CELL_SIZE, y*CELL_SIZE+50, CELL_SIZE, CELL_SIZE))
                elif grid[y][x] == 0:
                    pygame.draw.circle(screen, YELLOW, (x*CELL_SIZE+CELL_SIZE//2, y*CELL_SIZE+CELL_SIZE//2+50), 3)
        
        draw_pacman()
        for ghost in ghosts:
            draw_ghost(ghost)
        
        # Score display
        score_text = font.render(f"Score: {score}", True, WHITE)
        screen.blit(score_text, (10, 10))
        
        # Collision check
        for ghost in ghosts:
            if pacman['x'] == ghost['x'] and pacman['y'] == ghost['y']:
                game_state = GAME_OVER
    
    elif game_state == GAME_OVER:
        draw_game_over()
    
    pygame.display.flip()
    clock.tick(60)

pygame.quit()
sys.exit()
