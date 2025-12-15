# 2025-CT-
# -------------------------------------------
# yonsei_escape_game_ui.py (Start 화면 추가 버전)
# -------------------------------------------

import pygame
import sys
import random
import os
from typing import Tuple

pygame.init()

# =======================
# 폰트
# =======================
FONT_PATH = os.path.join("assets", "NanumGothic.ttf")

def load_font(size):
    try:
        return pygame.font.Font(FONT_PATH, size)
    except:
        return pygame.font.SysFont(None, size)

FONT_SMALL = load_font(18)
FONT_MED = load_font(24)
FONT_BIG = load_font(30)

# =======================
# 화면 설정
# =======================
SCREEN_WIDTH = 1280
SCREEN_HEIGHT = 720
FPS = 30

MAX_HP = 10
MAX_STEALTH = 5

WHITE = (245,245,245)
BLACK = (18,18,18)
UI_BG = (32,32,36)
HEART_RED = (220,40,40)
EFFECT_BG = (50,50,55)

STUDENT_TYPES = {
    '일반': (170,170,255),
    '냄새남': (170,90,80),
    '일러바침': (240,180,110),
    '졸음': (160,220,160),
    '괴짜': (210,140,240),
}

# =======================
# 스프라이트 로딩 함수
# =======================
def load_sprite(path, size=48):
    full_path = os.path.join("assets", path)
    try:
        img = pygame.image.load(full_path).convert_alpha()
        img = pygame.transform.scale(img, (size, size))
        return img
    except Exception as e:
        print("이미지 로드 실패:", full_path, e)
        surf = pygame.Surface((size, size), pygame.SRCALPHA)
        surf.fill((255, 0, 0, 180))
        return surf


# =======================
# 퀴즈
# =======================
QUIZ_BANK = [
    {'q':'연세대학교의 교색은?','choices':['푸른색','빨간색','노란색','보라색'],'a':0},
    {'q':'대한민국 수도는?','choices':['서울','부산','대구','인천'],'a':0},
    {'q':'Python은 무엇인가?','choices':['뱀','프로그래밍 언어','자동차','케이크'],'a':1},
    {'q':'4 × 6 = ?','choices':['20','24','28','22'],'a':1},
    {'q':'CPU의 역할은?','choices':['그래픽','중앙 연산 처리','전원','사운드'],'a':1},
    {'q':'연세대학교가 위치한 도시는?','choices':['서울','부산','대전','광주'],'a':0},

    # ---- 연세대 문제 ----
    {'q':'연세대학교의 개교 연도는?','choices':['1885년','1895년','1900년','1910년'], 'a':0},
    {'q':'연세대학교의 영문 약자는?','choices':['YSU','YNU','Yonsei U.','YU'], 'a':2},
    {'q':'연세대 교내 상징 동물은?','choices':['호랑이','독수리','사자','곰'], 'a':1},
    {'q':'연세대학교와 고려대학교의 스포츠 대결 이름은?','choices':['연고전','고연전','서연전','스포츠챌린지'],'a':0},
    {'q':'연세대학교의 대표 슬로건은?','choices':['Leading the Future','Proud to be Yonsei','The First and the Best','Light & Freedom'], 'a':3},
    {'q': '연세대 학생 식당 이름은?',
     'choices': ['백양관 식당', '학생회관 식당', '연세푸드센터', '청송관'], 'a': 1},

    {'q': '연세대 축제 이름은?',
     'choices': ['대동제', '아카라카', '청춘제', '연고제'], 'a': 1},

    {'q': '아카라카에서 흔히 외치는 응원은?',
     'choices': ['아카라카!', '아카라카를 위하여!', '연세 파이팅!', '고연전 승리!'], 'a': 1},

    {'q': '연세대학교의 건학 이념 중 하나는?',
     'choices': ['진리와 자유', '창조와 도전', '자유와 정의', '정의와 진실'], 'a': 0},

    {'q': '연세대의 대표적인 건물인 언더우드관의 영어 명칭은?',
     'choices': ['Underwood Hall', 'Underwood House', 'Underwood Building', 'U-Hall'], 'a': 0},

    {'q': '연세대에 있는 언덕의 이름은?',
     'choices': ['독수리언덕', '백양언덕', '연세언덕', '언더우드언덕'], 'a': 1},
]


# =======================
# 클래스 정의
# =======================
class Student:
    def __init__(self, stype, sprite):
        self.type = stype
        self.sprite = sprite

class Player:
    def __init__(self, row, col, sprite):
        self.row = row
        self.col = col
        self.hp = MAX_HP
        self.stealth = MAX_STEALTH
        self.move_count = 0
        self.effects = {}
        self.sprite = sprite

    def add_effect(self, name, turns):
        self.effects[name] = max(self.effects.get(name, 0), turns)

    def decrement_effects(self):
        remove = []
        for k in self.effects:
            self.effects[k] -= 1
            if self.effects[k] <= 0:
                remove.append(k)
        for k in remove:
            del self.effects[k]

# =======================
# 게임 클래스
# =======================
class Game:
    def __init__(self):
        self.state = 'question'
        self.last_message = ''
        self.current_question = None
        self.effect_log = []

        # 배경
        bg_path = os.path.join("assets", "classroom.png")
        self.background = pygame.image.load(bg_path).convert_alpha()
        self.background = pygame.transform.scale(self.background, (1024,1024))
        self.bg_rect = self.background.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 - 40))

        # 책상 좌표
        self.DESKS = [
            [(225,145),(360,145),(500,145),(785,145),(920,145),(1055,145)],
            [(225,280),(360,280),(500,280),(785,280),(920,280),(1055,280)],
            [(225,415),(360,415),(500,415),(785,415),(920,415),(1055,415)],
        ]

        self.rows = 3
        self.cols = 6

        # 스프라이트
        self.player_sprite = load_sprite("player.png", 48)
        self.student_sprites = [
            load_sprite("student1.png", 48),
            load_sprite("student2.png", 48),
            load_sprite("student3.png", 48),
            load_sprite("student4.png", 48),
        ]

        self.player = Player(0, 0, self.player_sprite)

        self.students = {}
        self.populate_students()

        self.win_img = load_sprite("win.png", size=720)
        self.lose_img = load_sprite("lose.png", size=720)

        self.ask_new_question()

    def populate_students(self):
        types = list(STUDENT_TYPES.keys())
        coords = [(r,c) for r in range(self.rows) for c in range(self.cols)]
        coords.remove((self.player.row, self.player.col))
        random.shuffle(coords)

        for i in range(7):
            r,c = coords[i]
            stype = random.choices(types,[40,15,10,20,15])[0]
            sprite = random.choice(self.student_sprites)
            self.students[(r,c)] = Student(stype, sprite)

    def ask_new_question(self):
        self.current_question = random.choice(QUIZ_BANK)
        self.last_message = '퀴즈 정답 시 이동 가능'
        self.state = 'question'
        self.player.decrement_effects()

    def check_answer(self, idx):
        if idx == self.current_question['a']:
            self.state = 'move'
            self.last_message = '정답! 방향키로 이동!'
        else:
            self.player.hp -= 1
            self.player.stealth -= 1
            self.last_message = '오답! 체력/위험 -1'
            if self.player.hp <= 0 or self.player.stealth <= 0:
                self.state = 'gameover'
            else:
                self.ask_new_question()

    def try_move(self, dr, dc):
        nr = self.player.row + dr
        nc = self.player.col + dc

        if not (0 <= nr < self.rows and 0 <= nc < self.cols):
            self.last_message = '범위 밖'
            return

        self.player.row = nr
        self.player.col = nc

        self.handle_tile(nr, nc)

        if nr == self.rows-1 and nc == self.cols-1:
            self.state = 'win'
            return

        if self.state != 'gameover':
            self.ask_new_question()

    def handle_tile(self, r, c):
        if (r,c) not in self.students:
            self.last_message = '빈 자리'
            return

        st = self.students.pop((r,c))
        t = st.type

        def log_effect(text):
            self.effect_log.append(text)
            if len(self.effect_log) > 3:
                self.effect_log.pop(0)

        if t == '냄새남':
            self.player.hp -= 2
            self.player.add_effect('냄새', 2)
            log_effect("냄새남 → HP -2, 냄새(2턴)")

        elif t == '일러바침':
            self.player.stealth -= 2
            self.player.add_effect('신고', 2)
            log_effect("일러바침 → 위험 -2, 신고(2턴)")

        elif t == '졸음':
            self.player.hp = min(MAX_HP, self.player.hp + 1)
            self.player.add_effect('휴식', 1)
            log_effect("졸음 → HP +1, 휴식(1턴)")

        elif t == '괴짜':
            roll = random.randint(0,2)
            if roll == 0:
                self.player.hp -= 1
                log_effect("괴짜 → HP -1")
            elif roll == 1:
                self.player.add_effect('이동+1', 1)
                log_effect("괴짜 → 이동 +1")
            else:
                log_effect("괴짜 → 변화 없음")

        if self.player.hp <= 0 or self.player.stealth <= 0:
            self.state = 'gameover'

    # -------------------------------------------
    # 화면 그리기
    # -------------------------------------------
    def draw(self, surf):

        if self.state == 'win':
            surf.fill((0,0,0))
            surf.blit(self.win_img, (280,0))
            return

        if self.state == 'gameover':
            surf.fill((0,0,0))
            surf.blit(self.lose_img, (280,0))
            return

        surf.blit(self.background, self.bg_rect)

        # 학생들
        for (r,c), student in self.students.items():
            x, y = self.DESKS[r][c]
            surf.blit(student.sprite, (x-24, y-24))

        # 플레이어
        px, py = self.DESKS[self.player.row][self.player.col]
        surf.blit(self.player.sprite, (px-24, py-24))

        # UI 아래
        ui = pygame.Rect(0,SCREEN_HEIGHT-180,SCREEN_WIDTH,180)
        pygame.draw.rect(surf,UI_BG,ui)

        if self.state == 'question':
            surf.blit(FONT_MED.render("퀴즈:",True,WHITE),(40,SCREEN_HEIGHT-150))
            surf.blit(FONT_SMALL.render(self.current_question['q'],True,WHITE),(130,SCREEN_HEIGHT-150))
            for i,ch in enumerate(self.current_question['choices']):
                surf.blit(FONT_SMALL.render(f"{i+1}. {ch}",True,WHITE),(60,SCREEN_HEIGHT-120+i*28))
            surf.blit(FONT_SMALL.render("1~4 키로 정답 선택",True,WHITE),(SCREEN_WIDTH-260,SCREEN_HEIGHT-40))

        else:
            surf.blit(FONT_MED.render(self.last_message,True,WHITE),(40,SCREEN_HEIGHT-150))

        self.draw_status_panel(surf)
        self.draw_effect_panel(surf)
        self.draw_effect_log(surf)

    # 체력 패널
    def draw_status_panel(self, surf):
        panel_w = 280
        panel_h = 140
        effect_w = 260
        panel_x = SCREEN_WIDTH - effect_w - panel_w - 60
        panel_y = 40

        rect = pygame.Rect(panel_x, panel_y, panel_w, panel_h)
        pygame.draw.rect(surf, (20,20,20), rect, border_radius=8)

        bar_x = panel_x + 12
        bar_y = panel_y + 12
        bar_w = panel_w - 30
        bar_h = 16

        pygame.draw.rect(surf,(90,90,90),(bar_x,bar_y,bar_w,bar_h),2)
        fill = int((self.player.hp/MAX_HP)*(bar_w-4))
        if fill>0: pygame.draw.rect(surf,HEART_RED,(bar_x+2,bar_y+2,fill,bar_h-4))
        surf.blit(FONT_SMALL.render(f"체력 {self.player.hp}/{MAX_HP}",True,WHITE),(bar_x, bar_y+22))

        s_y = bar_y + 46
        pygame.draw.rect(surf,(90,90,90),(bar_x,s_y,bar_w,14),2)
        s_fill = int((self.player.stealth/MAX_STEALTH)*(bar_w-4))
        if s_fill>0: pygame.draw.rect(surf,(220,200,40),(bar_x+2,s_y+2,s_fill,10))
        surf.blit(FONT_SMALL.render(f"들킬 위험 {self.player.stealth}/{MAX_STEALTH}",True,WHITE),(bar_x, s_y+18))

        surf.blit(FONT_SMALL.render(f"이동 {self.player.move_count}",True,WHITE),(bar_x, panel_y+100))

    # 특수효과 패널
    def draw_effect_panel(self, surf):
        panel_w = 260
        panel_h = 120
        panel_x = SCREEN_WIDTH - panel_w - 20
        panel_y = 40

        pygame.draw.rect(surf,EFFECT_BG,(panel_x,panel_y,panel_w,panel_h),border_radius=6)
        surf.blit(FONT_SMALL.render("특수효과",True,WHITE),(panel_x+12,panel_y+8))

        y = panel_y + 36
        if not self.player.effects:
            surf.blit(FONT_SMALL.render("없음",True,(180,180,180)),(panel_x+12,y))
        else:
            for i,(name,turns) in enumerate(self.player.effects.items()):
                surf.blit(FONT_SMALL.render(f"{name} ({turns})",True,WHITE),(panel_x+12,y+i*26))

    # 효과 로그
    def draw_effect_log(self, surf):
        base_x = SCREEN_WIDTH - 260
        base_y = 40 + 120 + 20

        surf.blit(FONT_MED.render("최근 효과", True, WHITE), (base_x, base_y))

        for i, text in enumerate(reversed(self.effect_log)):
            surf.blit(FONT_SMALL.render(f"- {text}", True, WHITE), (base_x + 6, base_y + 30 + i*22))


# =======================
# main — ★ start 화면 추가
# =======================
def main():
    screen = pygame.display.set_mode((SCREEN_WIDTH,SCREEN_HEIGHT))
    pygame.display.set_caption("연세대학교 강의실 탈출")

    clock = pygame.time.Clock()

    # ★ Start 화면 이미지 로드
    start_img = pygame.image.load(os.path.join("assets", "start.png"))
    start_img = pygame.transform.scale(start_img, (SCREEN_WIDTH, SCREEN_HEIGHT))

    state = "start"
    game = None

    running=True
    while running:
        clock.tick(FPS)
        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                running=False
            elif e.type == pygame.KEYDOWN:
                if e.key == pygame.K_ESCAPE:
                    running=False

                # ★ START 화면 → 엔터 누르면 본게임 시작
                if state == "start":
                    if e.key == pygame.K_RETURN:
                        game = Game()
                        state = "game"

                # ★ 게임 중 이벤트
                elif state == "game":
                    if game.state=='question' and e.key in (pygame.K_1,pygame.K_2,pygame.K_3,pygame.K_4):
                        game.check_answer({pygame.K_1:0,pygame.K_2:1,pygame.K_3:2,pygame.K_4:3}[e.key])

                    elif game.state=='move':
                        if e.key==pygame.K_RIGHT: game.try_move(0,1)
                        if e.key==pygame.K_LEFT:  game.try_move(0,-1)
                        if e.key==pygame.K_DOWN:  game.try_move(1,0)
                        if e.key==pygame.K_UP:    game.try_move(-1,0)

                    elif game.state in ('win','gameover') and e.key==pygame.K_RETURN:
                        game = Game()

        # ---------------------------------------
        # 화면 출력
        # ---------------------------------------
        if state == "start":
            screen.blit(start_img, (0,0))

        else:
            game.draw(screen)

        pygame.display.flip()

    pygame.quit()
    sys.exit()


if __name__=="__main__":
    main()
