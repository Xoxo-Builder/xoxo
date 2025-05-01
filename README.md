# xoxo
import tkinter as tk
from tkinter import simpledialog, messagebox
import socket
import threading
import random

# --- Constants ---
BG_COLOR = "#23272e"  # Dark background
X_COLOR = "#4fc3f7"   # Blue for X
O_COLOR = "#f06292"   # Pink for O
WIN_COLOR = "#43ea5e" # Green for win
LOSE_COLOR = "#ea4343" # Red for lose
DRAW_COLOR = "#fbc02d" # Yellow for draw
BTN_FONT = ("Helvetica", 32, "bold")
LABEL_FONT = ("Helvetica", 14, "bold")
SMALL_FONT = ("Helvetica", 11)
BTN_SIZE = {"height": 1, "width": 3}
DEFAULT_PORT = 5555

# --- Game Logic ---
def check_winner(board):
    wins = [(0,1,2),(3,4,5),(6,7,8),(0,3,6),(1,4,7),(2,5,8),(0,4,8),(2,4,6)]
    for a,b,c in wins:
        if board[a] == board[b] == board[c] and board[a] is not None:
            return board[a]
    if all(x is not None for x in board):
        return "Draw"
    return None

def ai_move(board, ai_symbol, player_symbol):
    # Simple AI: Win if possible, block if needed, else random
    for i in range(9):
        if board[i] is None:
            board[i] = ai_symbol
            if check_winner(board) == ai_symbol:
                board[i] = None
                return i
            board[i] = None
    for i in range(9):
        if board[i] is None:
            board[i] = player_symbol
            if check_winner(board) == player_symbol:
                board[i] = None
                return i
            board[i] = None
    choices = [i for i, v in enumerate(board) if v is None]
    return random.choice(choices)

# --- Main Menu ---
class MainMenu(tk.Frame):
    def __init__(self, master, start_callback):
        super().__init__(master, bg=BG_COLOR)
        self.start_callback = start_callback
        tk.Label(self, text="XO", font=("Helvetica", 24, "bold"), bg=BG_COLOR, fg="white").pack(pady=16)
        tk.Button(self, text="Player vs Player (Local)", font=LABEL_FONT, command=lambda: self.symbol_select("pvp"), bg="#2d313a", fg="white", activebackground="#444").pack(pady=5)
        tk.Button(self, text="Player vs AI", font=LABEL_FONT, command=lambda: self.symbol_select("ai"), bg="#2d313a", fg="white", activebackground="#444").pack(pady=5)
        tk.Button(self, text="Online: Host Game", font=LABEL_FONT, command=lambda: self.symbol_select("host"), bg="#2d313a", fg="white", activebackground="#444").pack(pady=5)
        tk.Button(self, text="Online: Join Game", font=LABEL_FONT, command=lambda: self.symbol_select("join"), bg="#2d313a", fg="white", activebackground="#444").pack(pady=5)
        tk.Button(self, text="Exit", font=LABEL_FONT, command=master.quit, bg="#444", fg="white").pack(pady=16)

    def symbol_select(self, mode):
        SymbolSelect(self, mode, self.start_callback)

class SymbolSelect(tk.Toplevel):
    def __init__(self, parent, mode, start_callback):
        super().__init__(parent)
        self.title("Choose Symbol")
        self.configure(bg=BG_COLOR)
        self.resizable(False, False)
        self.mode = mode
        self.start_callback = start_callback
        tk.Label(self, text="Choose your symbol", font=LABEL_FONT, bg=BG_COLOR, fg="white").pack(pady=10)
        btn_frame = tk.Frame(self, bg=BG_COLOR)
        btn_frame.pack(pady=10)
        tk.Button(btn_frame, text="X", font=BTN_FONT, bg=BG_COLOR, fg=X_COLOR, width=4, command=lambda: self.select("X")).pack(side="left", padx=10)
        tk.Button(btn_frame, text="O", font=BTN_FONT, bg=BG_COLOR, fg=O_COLOR, width=4, command=lambda: self.select("O")).pack(side="left", padx=10)
        tk.Button(self, text="Cancel", font=SMALL_FONT, command=self.destroy, bg="#444", fg="white").pack(pady=10)

    def select(self, symbol):
        self.destroy()
        self.master.start_callback(self.mode, symbol)

# --- Game Board ---
class XOBoard(tk.Frame):
    def __init__(self, master, player1, player2, symbol, mode, network=None):
        super().__init__(master, bg=BG_COLOR)
        self.master = master
        self.player1 = player1
        self.player2 = player2
        self.symbol = symbol
        self.mode = mode
        self.network = network
        self.board = [None]*9
        self.current = "X"
        self.buttons = []
        self.locked = False
        self.create_widgets()
        if self.mode == "ai" and self.symbol == "O":
            self.master.after(500, self.ai_turn)

    def create_widgets(self):
        tk.Label(self, text=f"{self.player1} (X) vs {self.player2} (O)", font=LABEL_FONT, bg=BG_COLOR, fg="white").grid(row=0, column=0, columnspan=3, pady=6)
        for i in range(9):
            btn = tk.Button(self, text="", font=BTN_FONT, bg="#2d313a", fg="white", **BTN_SIZE,
                            command=lambda i=i: self.cell_click(i), activebackground="#444")
            btn.grid(row=1+i//3, column=i%3, padx=3, pady=3)
            self.buttons.append(btn)
        self.status = tk.Label(self, text="X's turn", font=SMALL_FONT, bg=BG_COLOR, fg="white")
        self.status.grid(row=4, column=0, columnspan=3, pady=6)
        tk.Button(self, text="Restart", font=SMALL_FONT, command=self.restart, bg="#444", fg="white").grid(row=5, column=0, pady=6)
        tk.Button(self, text="Home", font=SMALL_FONT, command=self.go_home, bg="#444", fg="white").grid(row=5, column=1, pady=6)
        tk.Button(self, text="Exit", font=SMALL_FONT, command=self.master.quit, bg="#444", fg="white").grid(row=5, column=2, pady=6)

    def cell_click(self, idx):
        if self.locked or self.board[idx] is not None:
            return
        if self.mode == "pvp":
            self.make_move(idx, self.current)
            self.current = "O" if self.current == "X" else "X"
            self.status.config(text=f"{self.current}'s turn")
        elif self.mode == "ai":
            if self.current == self.symbol:
                self.make_move(idx, self.symbol)
                self.current = "O" if self.symbol == "X" else "X"
                self.status.config(text="AI's turn")
                self.locked = True
                self.master.after(500, self.ai_turn)
        elif self.mode in ("host", "join"):
            if self.current == self.symbol:
                self.make_move(idx, self.symbol)
                self.locked = True
                self.status.config(text="Waiting for opponent...")
                self.network.send_move(idx)

    def make_move(self, idx, symbol):
        self.board[idx] = symbol
        # Always set X to blue and O to pink
        color = X_COLOR if symbol == "X" else O_COLOR
        self.buttons[idx].config(
            text=symbol,
            fg=color
        )
        winner = check_winner(self.board)
        if winner:
            self.locked = True
            for btn in self.buttons:
                btn.config(state="disabled")
            if winner == "Draw":
                self.status.config(text="It's a draw!", fg=DRAW_COLOR)
                messagebox.showinfo("Draw", "It's a draw!")
            else:
                # Determine win/lose/draw color and message
                if self.mode == "ai":
                    if winner == self.symbol:
                        self.status.config(text=f"Player {winner} wins!", fg=WIN_COLOR)
                        messagebox.showinfo("Winner", "You win!")
                    else:
                        self.status.config(text="You lost!", fg=LOSE_COLOR)
                        messagebox.showinfo("Lost", "You lost!")
                elif self.mode in ("host", "join"):
                    if winner == self.symbol:
                        self.status.config(text=f"Player {winner} wins!", fg=WIN_COLOR)
                        messagebox.showinfo("Winner", "You win!")
                    else:
                        self.status.config(text="You lost!", fg=LOSE_COLOR)
                        messagebox.showinfo("Lost", "You lost!")
                else:  # PvP local
                    if winner == self.symbol:
                        self.status.config(text=f"Player {winner} wins!", fg=WIN_COLOR)
                    else:
                        self.status.config(text=f"Player {winner} wins!", fg=WIN_COLOR)
        else:
            self.buttons[idx].config(state="disabled")

    def ai_turn(self):
        if self.locked:
            idx = ai_move(self.board, "O" if self.symbol=="X" else "X", self.symbol)
            self.make_move(idx, "O" if self.symbol=="X" else "X")
            self.current = self.symbol
            self.status.config(text=f"{self.symbol}'s turn", fg="white")
            self.locked = False

    def opponent_move(self, idx):
        opp_symbol = "O" if self.symbol == "X" else "X"
        self.make_move(idx, opp_symbol)
        self.current = self.symbol
        self.status.config(text=f"Your turn", fg="white")
        self.locked = False

    def restart(self):
        for i in range(9):
            self.board[i] = None
            self.buttons[i].config(text="", state="normal", fg="white")
        self.current = "X"
        self.locked = False
        self.status.config(text=f"{self.current}'s turn", fg="white")
        if self.mode == "ai" and self.symbol == "O":
            self.locked = True
            self.master.after(500, self.ai_turn)

    def go_home(self):
        if self.network:
            self.network.close()
        self.master.show_menu()

# --- Networking for Online Play ---
class NetworkClient:
    def __init__(self, board, host, port, symbol, is_host):
        self.board = board
        self.symbol = symbol
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((host, port))
        self.is_host = is_host
        threading.Thread(target=self.listen, daemon=True).start()
        if not is_host and symbol == "O":
            self.board.locked = True
            self.board.status.config(text="Waiting for opponent...", fg="white")

    def send_move(self, idx):
        try:
            self.sock.sendall(str(idx).encode())
        except:
            messagebox.showerror("Error", "Connection lost.")
            self.board.go_home()

    def listen(self):
        try:
            while True:
                data = self.sock.recv(1024)
                if not data:
                    break
                idx = int(data.decode())
                self.board.opponent_move(idx)
        except:
            pass
        finally:
            self.sock.close()

    def close(self):
        try:
            self.sock.close()
        except:
            pass

class NetworkServer:
    def __init__(self, board, port):
        self.board = board
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.bind(("", port))
        self.sock.listen(1)
        threading.Thread(target=self.accept, daemon=True).start()

    def accept(self):
        client, _ = self.sock.accept()
        self.client = client
        threading.Thread(target=self.listen, daemon=True).start()

    def send_move(self, idx):
        try:
            self.client.sendall(str(idx).encode())
        except:
            messagebox.showerror("Error", "Connection lost.")
            self.board.go_home()

    def listen(self):
        try:
            while True:
                data = self.client.recv(1024)
                if not data:
                    break
                idx = int(data.decode())
                self.board.opponent_move(idx)
        except:
            pass
        finally:
            self.client.close()

    def close(self):
        try:
            self.client.close()
        except:
            pass

# --- Main Application ---
class XOApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("XO")
        self.geometry("320x420")
        self.config(bg=BG_COLOR)
        self.resizable(False, False)
        self.menu = MainMenu(self, self.start_game)
        self.menu.pack(fill="both", expand=True)

    def show_menu(self):
        for widget in self.winfo_children():
            widget.destroy()
        self.menu = MainMenu(self, self.start_game)
        self.menu.pack(fill="both", expand=True)

    def start_game(self, mode, symbol):
        for widget in self.winfo_children():
            widget.destroy()
        if mode == "pvp":
            board = XOBoard(self, "Player 1", "Player 2", symbol, mode)
            board.pack(fill="both", expand=True)
        elif mode == "ai":
            board = XOBoard(self, "You", "AI", symbol, mode)
            board.pack(fill="both", expand=True)
        elif mode == "host":
            port = simpledialog.askinteger("Host Game", "Enter port to host on (e.g. 5555):", parent=self, minvalue=1024, maxvalue=65535)
            if not port:
                self.show_menu()
                return
            board = XOBoard(self, "You", "Opponent", symbol, mode)
            board.pack(fill="both", expand=True)
            net = NetworkServer(board, port)
            board.network = net
        elif mode == "join":
            ip = simpledialog.askstring("Join Game", "Enter host IP:", parent=self)
            port = DEFAULT_PORT
            if not ip:
                self.show_menu()
                return
            board = XOBoard(self, "You", "Opponent", symbol, mode)
            board.pack(fill="both", expand=True)
            net = NetworkClient(board, ip, port, symbol, is_host=False)
            board.network = net

if __name__ == "__main__":
    XOApp().mainloop()
