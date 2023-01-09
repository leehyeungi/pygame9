# pygame9
import math
import pygame
import time

from pygame import draw
from pygame.color import Color
from pygame.sprite import Sprite

from catapult import Catapult
from stone import Stone
from alien import Alien
from missile import Missile
from explosion import Explosion
from const import *

FPS = 60
stone_count = 5
alien_count = 3
missile_count = 1000
catapult_count = 3

def decrement_stones():
    global stone_count
    stone_count -= 1

def decrement_missiles():
    global missile_count
    missile_count -= 1

class Background(Sprite):
    def __init__(self):
        self.sprite_image = 'background.png'
        self.image = pygame.image.load(
            self.sprite_image).convert()
        self.rect = self.image.get_rect()
        self.rect.x = 0
        self.dx = 1

        Sprite.__init__(self)
        
    def update(self):
        self.rect.x -= self.dx
        if self.rect.x == -800:
            self.rect.x = 0

if __name__ == "__main__":
    pygame.init()

    size = (400, 300)
    screen = pygame.display.set_mode(size)
    #pygame.display.set_mode(size, pygame.FULLSCREEN) #화면 커짐
    pygame.display.set_caption("Catapult VS Alien")

    run = True
    move = False
    
    clock = pygame.time.Clock()
    
    t = 0
    tt = 0

    movexy = 0
    power = 15
    PI = math.pi

    game_state = GAME_INIT
    background = Background()
    background_group = pygame.sprite.Group()
    background_group.add(background)

    stone = Stone() #스톤
    stone.rect.y = -100 #위치변경
    stone_group = pygame.sprite.Group()
    stone_group.add(stone)

    missile = Missile() #미사일
    missile.rect.y = -100
    missile_group = pygame.sprite.Group()
    missile_group.add(missile)

    catapult = Catapult(stone)
    catapult.rect.x = 50 #위치변경
    catapult.rect.y = BASE_Y
    catapult_group = pygame.sprite.Group()
    catapult_group.add(catapult)

    alien = Alien(missile)
    alien.rect.x = 350
    alien.rect.y = BASE_Y
    alien_group = pygame.sprite.Group()
    alien_group.add(alien)

    explosion = Explosion()
    explosion_group = pygame.sprite.Group()
    explosion_group.add(explosion)
    mouse_pos = (300,100)

    mouse_check = 0
    alien_shooting = 0
    time_counting = 1
    directions = 180
    powers = 10
    
    #게임루프
    while run:
        #1)사용자 입력 처리
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                run = False
            elif event.type == pygame.KEYUP:
                if event.key == pygame.K_SPACE:
                    # 초기화면에서 스페이스를 입력하면 시작
                    if game_state == GAME_INIT:
                        game_state = GAME_PLAY

            #마우스 모션으로 direction 움직이기
            if event.type == pygame.MOUSEMOTION:
                mouse_pos = pygame.mouse.get_pos()
                rad = math.atan2(catapult.rect.y - mouse_pos[1], abs(catapult.rect.x + 32 - mouse_pos[0]))
                deg = (rad * 180) / PI
                direction = int(deg)
                    
        #에일리언 움직이기    
        if game_state == GAME_PLAY:
            if movexy == 0: #앞으로 나간다
                if alien.rect.x > 250:
                    alien.rect.x -= 1
                if alien.rect.x == 250:
                    movexy = 1 
            elif movexy == 1: #위로 올라간다
                if alien.rect.y > 150:
                    alien.rect.y -= 1
                if alien.rect.y == 150:
                    movexy = 2
            elif movexy == 2: #뒤로 나간다
                if alien.rect.x < 350:
                    alien.rect.x += 1
                if alien.rect.x == 350:
                    movexy = 3
            elif movexy == 3: #아래로 내려간다
                if alien.rect.y < BASE_Y:
                    alien.rect.y += 1
                if alien.rect.y == BASE_Y:
                    movexy = 0
                
        if game_state == GAME_PLAY:
            #누르고 있는 키 확인하기.
            keys = pygame.key.get_pressed()
             #투석기 움직이기
            if keys[pygame.K_LEFT]: 
                catapult.backward()
            elif keys[pygame.K_RIGHT]:
                catapult.forward()
            elif keys[pygame.K_UP]:
                catapult.upward()
            elif keys[pygame.K_DOWN]:
                catapult.downward()

        #L마우스 버튼이 눌려지면 발사 파워 증가
        if event.type == pygame.MOUSEBUTTONDOWN or mouse_check == 1:
            mouse_check = 1
            if power > MAX_POWER:
                power = MIN_POWER
            else:
                power += 0.2   
                
        #GAME_PLAY 상태일 때 마우스 L버튼이 떨어지면 투석기 공 발사  
        if event.type == pygame.MOUSEBUTTONUP:
            mouse_check = 0
            if stone.state == STONE_READY:
                t = 0
                catapult.fire(power, direction)

        #GAME_PLAY 상태일 때 에일리언이 돌을 던짐
        if game_state == GAME_PLAY:
            if time_counting % 120 == 0:
                if missile.state == MISSILE_READY:
                    tt = 0
                    alien.fire(powers, directions) #직선으로 날아가게 만들어야함.

        #2) 게임 상태 업데이트
        if stone.state == STONE_FLY:
            t += 0.5
            stone.move(t, (screen.get_width(), screen.get_height()),
                       decrement_stones)

        if missile.state == MISSILE_FLY:
            tt += 0.5
            missile.move(tt, (screen.get_width(), screen.get_height()),
                         decrement_missiles)
            
        if alien.alive(): #stone과 alien의 충돌 여부를 확인함.
            collided = pygame.sprite.groupcollide(stone_group, alien_group, False, True)
            if collided:
                pp = collided
                explosion.rect.x = \
                    (alien.rect.x + alien.rect.width/2) -\
                    explosion.rect.width/2
                explosion.rect.y = \
                    (alien.rect.y + alien.rect.width/2) -\
                    explosion.rect.height/2

        elif not explosion.alive():
            #외계인도 죽고 폭발 애니메이션도 끝났을 때.
            alien_group.add(alien)
            alien_count -= 1
            if alien_count == 0:
                game_state = GAME_CLEAR

        if catapult.alive(): #missile과 catepult의 충돌 여부를 확인함.
            collideds = pygame.sprite.groupcollide(missile_group, catapult_group, False, True)
            if collideds:
                k = collideds #collideds의 값이 k에 안들어감
                explosion.rect.x = \
                    (catapult.rect.x + catapult.rect.width/2) -\
                    explosion.rect.width/2
                explosion.rect.y = \
                    (catapult.rect.y + catapult.rect.width/2) -\
                    explosion.rect.height/2

        elif not explosion.alive():
            #투석기가 죽고 폭발 애니메이션도 끝났을 떄.
            catapult_group.add(catapult)
            catapult_count -= 1
            if catapult_count == 0:
                game_state = GAME_OVER

        #외계인이 살아 있는데 돌멩이 수가 0이면 게임 오버.
        if alien.alive() and stone_count == 0:
            game_state = GAME_OVER

        #투석기가 살아 있는데 미사일 수가 0이면 게임 오버.
        if catapult.alive() and missile_count == 0:
            game_state = GAME_OVER
    
        if game_state == GAME_PLAY: #게임 객체 업데이트
            catapult_group.update()
            stone_group.update()
            alien_group.update()
            missile_group.update()

        #3)게임 상태 그리기
        background_group.update()
        background_group.draw(screen)

        if game_state == GAME_INIT:
            #초기화면
            sf = pygame.font.SysFont('Arial', 20, bold = True)
            title_str = "Catapult VS Alien"
            title = sf.render(title_str, True, (255, 0, 0))
            title_size = sf.size(title_str)
            title_pos = (screen.get_width()/2 - title_size[0]/2, 100)

            sub_title_str = "Press [Space] key To Start"
            sub_title = sf.render(sub_title_str, True, (255, 0, 0))
            sub_title_size = sf.size(sub_title_str)
            sub_title_pos = (screen.get_width()/2 - sub_title_size[0]/2, 200)

            screen.blit(title, title_pos)
            screen.blit(sub_title, sub_title_pos)

        elif game_state == GAME_PLAY:
            #플레이 화면
            time_counting += 1 #한바퀴 돌아갈 때마다 1씩 증가
            catapult_group.draw(screen)
            stone_group.draw(screen)
            alien_group.draw(screen)
            missile_group.draw(screen)

            #파워와 각도를 선으로 표현
            line_len = power*5
            r = math.radians(direction)
            pos1 = (catapult.rect.x+32, catapult.rect.y)
            pos2 = (pos1[0] + math.cos(r)*line_len,
                    pos1[1] - math.sin(r)*line_len)
            draw.line(screen, Color(255, 0, 0), pos1, pos2)

            #파워와 각도를 텍스트로 표현
            sf = pygame.font.SysFont("Arial", 15)
            text = sf.render("{0} , {1}m/s".
                             format(direction, int(power)), True, (0, 0, 0))
            screen.blit(text, pos2)

            #돌의 개수를 표시
            sf = pygame.font.SysFont("Monospace", 20)
            text = sf.render("Stones : {0}".
                             format(stone_count), True, (0,0,255))
            screen.blit(text, (10, 10))

            #투석기의 개수 표시
            sf = pygame.font.SysFont("Monospace", 20)
            text = sf.render("catapult : {0}".
                             format(catapult_count), True, (0, 0, 255))
            screen.blit(text, (10, 30))

            #alien이 돌에 맞아 죽었다면 애니메이션 재생
            if not alien.alive():
                explosion_group.update()
                explosion_group.draw(screen)

            #catepult가 돌에 맞아 죽었다면 애니메이션 재생
            if not catapult.alive():
                explosion_group.update()
                explosion_group.draw(screen)
                
            #에일리언의 개수 표시
            sf = pygame.font.SysFont("Monospace", 20)
            text = sf.render("alien : {0}".
                             format(alien_count), True, (0, 0, 255))
            screen.blit(text, (270, 10))
                
        elif game_state == GAME_CLEAR:
            #게임 클리어
            sf = pygame.font.SysFont("Arial", 20, bold=True)
            title_str = "Congratulations! Mission Complete"
            title = sf.render(title_str, True, (0, 0, 255))
            title_size = sf.size(title_str)
            title_pos = (screen.get_width()/2 - title_size[0]/2, 100)
            screen.blit(title, title_pos)

        elif game_state == GAME_OVER:
            #게임오버
            sf = pygame.font.SysFont("Arial", 20, bold=True)
            title_str = "Game Over"
            title = sf.render(title_str, True, (255,0,0))
            title_size = sf.size(title_str)
            title_pos = (screen.get_width()/2 - title_size[0]/2, 100)
            screen.blit(title, title_pos)

        pygame.display.flip()
        clock.tick(FPS)

    pygame.quit()

