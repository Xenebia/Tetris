# Tetris Tkinter - 120 FPS + plateau large

import tkinter as tk
import random

COLUMNS = 12
ROWS = 30
BASE_DELAY = 500  # vitesse chute en ms
FPS = 120          # rendu à 120 images par seconde

COLORS = {
    0: "#000000",
    1: "#00f0f0",
    2: "#0000f0",
    3: "#f0a000",
    4: "#f0f000",
    5: "#00f000",
    6: "#a000f0",
    7: "#f00000",
}

TETROMINOS = {
    1: [[0,0,0,0, 1,1,1,1, 0,0,0,0, 0,0,0,0],
        [0,0,1,0, 0,0,1,0, 0,0,1,0, 0,0,1,0]],
    2: [[1,0,0,0, 1,1,1,0, 0,0,0,0, 0,0,0,0],
        [0,1,1,0, 0,1,0,0, 0,1,0,0, 0,0,0,0],
        [0,0,0,0, 1,1,1,0, 0,0,1,0, 0,0,0,0],
        [0,1,0,0, 0,1,0,0, 1,1,0,0, 0,0,0,0]],
    3: [[0,0,1,0, 1,1,1,0, 0,0,0,0, 0,0,0,0],
        [0,1,0,0, 0,1,0,0, 0,1,1,0, 0,0,0,0],
        [0,0,0,0, 1,1,1,0, 1,0,0,0, 0,0,0,0],
        [1,1,0,0, 0,1,0,0, 0,1,0,0, 0,0,0,0]],
    4: [[0,1,1,0, 0,1,1,0, 0,0,0,0, 0,0,0,0]],
    5: [[0,1,1,0, 1,1,0,0, 0,0,0,0, 0,0,0,0],
        [0,1,0,0, 0,1,1,0, 0,0,1,0, 0,0,0,0]],
    6: [[0,1,0,0, 1,1,1,0, 0,0,0,0, 0,0,0,0],
        [0,1,0,0, 0,1,1,0, 0,1,0,0, 0,0,0,0],
        [0,0,0,0, 1,1,1,0, 0,1,0,0, 0,0,0,0],
        [0,1,0,0, 1,1,0,0, 0,1,0,0, 0,0,0,0]],
    7: [[1,1,0,0, 0,1,1,0, 0,0,0,0, 0,0,0,0],
        [0,0,1,0, 0,1,1,0, 0,1,0,0, 0,0,0,0]]
}

class Tetris:
    def __init__(self, root): #fenêtre
        self.root = root
        self.root.title("Tetris")

        screen_w = root.winfo_screenwidth()
        screen_h = root.winfo_screenheight()
        self.cell_size = min(screen_w // (COLUMNS+6), screen_h // ROWS)
        self.width = self.cell_size * COLUMNS
        self.height = self.cell_size * ROWS

        self.canvas = tk.Canvas(root, width=self.width+200, height=self.height, bg="black")
        self.canvas.pack()

        self.root.bind("<Key>", self.key_pressed)
        self.root.bind("<F11>", self.toggle_fullscreen)
        self.root.bind("<Escape>", self.exit_fullscreen)
        self.fullscreen = False

        self.init_game()
        self.root.after(self.drop_delay, self.gravity_loop)
        self.root.after(1000//FPS, self.render_loop)

    def init_game(self): #Scorebord
        self.board = [[0 for _ in range(COLUMNS)] for _ in range(ROWS)]
        self.score = 0
        self.level = 1
        self.lines = 0
        self.drop_delay = BASE_DELAY
        self.active = self.random_piece()
        self.next_piece = self.random_piece()
        self.game_over = False

    def random_piece(self):
        kind = random.randint(1,7)
        return {"kind": kind, "rot": 0, "shape": TETROMINOS[kind], "x": 3, "y": 0}

    def valid_position(self, piece, px, py, rot=None):
        if rot is None: rot = piece["rot"]
        shape = piece["shape"][rot]
        for i in range(4):
            for j in range(4):
                if shape[i*4+j]:
                    x = px+j
                    y = py+i
                    if x<0 or x>=COLUMNS or y<0 or y>=ROWS: return False
                    if self.board[y][x]: return False
        return True

    def place_piece(self):
        shape = self.active["shape"][self.active["rot"]]
        kind = self.active["kind"]
        for i in range(4):
            for j in range(4):
                if shape[i*4+j]:
                    self.board[self.active["y"]+i][self.active["x"]+j] = kind
        self.clear_lines()
        self.active = self.next_piece
        self.active["x"], self.active["y"] = 3,0
        self.next_piece = self.random_piece()
        if not self.valid_position(self.active, self.active["x"], self.active["y"]):
            self.game_over = True

    def clear_lines(self):
        new_board=[]
        cleared=0
        for row in self.board:
            if all(row): cleared+=1
            else: new_board.append(row)
        for _ in range(cleared): new_board.insert(0,[0]*COLUMNS)
        self.board=new_board
        if cleared:
            self.lines+=cleared
            self.score+={1:40,2:100,3:300,4:1200}.get(cleared,0)*self.level
            self.level=1+(self.lines//10)
            self.drop_delay=max(50, BASE_DELAY-(self.level-1)*30)

    def step(self):
        if self.valid_position(self.active,self.active["x"],self.active["y"]+1):
            self.active["y"]+=1
        else: self.place_piece()

    def draw(self):
        self.canvas.delete("all")
        s=self.cell_size
        for r in range(ROWS):
            for c in range(COLUMNS):
                if self.board[r][c]:
                    self.canvas.create_rectangle(c*s,r*s,(c+1)*s,(r+1)*s,
                                                 fill=COLORS[self.board[r][c]], outline="#111")
        shape=self.active["shape"][self.active["rot"]]
        for i in range(4):
            for j in range(4):
                if shape[i*4+j]:
                    x=(self.active["x"]+j)*s
                    y=(self.active["y"]+i)*s
                    self.canvas.create_rectangle(x,y,x+s,y+s,
                                                 fill=COLORS[self.active["kind"]], outline="#111")
        self.canvas.create_line(self.width+2,0,self.width+2,self.height,fill="white",width=2)
        self.canvas.create_text(self.width+20,50,text=f"Score: {self.score}",fill="white",anchor="nw",font=("Arial",14))
        self.canvas.create_text(self.width+20,80,text=f"Niveau: {self.level}",fill="white",anchor="nw",font=("Arial",14))
        self.canvas.create_text(self.width+20,110,text=f"Lignes: {self.lines}",fill="white",anchor="nw",font=("Arial",14))
        if self.game_over:
            self.canvas.create_text(self.width//2,self.height//2,text="GAME OVER",fill="red",font=("Arial",24,"bold"))

    def gravity_loop(self):
        if not self.game_over: self.step()
        self.root.after(self.drop_delay,self.gravity_loop)

    def render_loop(self):
        self.draw()
        self.root.after(1000//FPS,self.render_loop)

    def key_pressed(self,e):
        if self.game_over: return
        if e.keysym=="Left" and self.valid_position(self.active,self.active["x"]-1,self.active["y"]): self.active["x"]-=1
        elif e.keysym=="Right" and self.valid_position(self.active,self.active["x"]+1,self.active["y"]): self.active["x"]+=1
        elif e.keysym=="Down" and self.valid_position(self.active,self.active["x"],self.active["y"]+1): self.active["y"]+=1
        elif e.keysym=="Up":
            new_rot=(self.active["rot"]+1)%len(self.active["shape"])
            if self.valid_position(self.active,self.active["x"],self.active["y"],new_rot): self.active["rot"]=new_rot
        elif e.keysym=="space":
            while self.valid_position(self.active,self.active["x"],self.active["y"]+1): self.active["y"]+=1
            self.place_piece()

    def toggle_fullscreen(self,event=None):
        self.fullscreen=not self.fullscreen
        self.root.attributes("-fullscreen",self.fullscreen)

    def exit_fullscreen(self,event=None):
        self.fullscreen=False
        self.root.attributes("-fullscreen",False)

if __name__=="__main__":
    root=tk.Tk()
    game=Tetris(root)
    root.mainloop()
