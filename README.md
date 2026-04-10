[BaharSoftwarePro1.py](https://github.com/user-attachments/files/26637395/BaharSoftwarePro1.py)
# Game-Recommendation-Systemimport tkinter as tk
from tkinter import messagebox, ttk
import sqlite3
import hashlib
import os

# --- 1. DATABASE AND SECURITY SETTINGS (Epic 2 & OWASP Protection) ---

def setup_database():
    # We are using Python's built-in SQLite database instead of PostgreSQL
    conn = sqlite3.connect("secure_games.db")
    cursor = conn.cursor()
    
    # Users table (Passwords are stored as hashes instead of plain text)
    cursor.execute('''CREATE TABLE IF NOT EXISTS users
                      (id INTEGER PRIMARY KEY, username TEXT UNIQUE, password_hash TEXT, salt TEXT)''')
    
    # Games table
    cursor.execute('''CREATE TABLE IF NOT EXISTS games
                      (id INTEGER PRIMARY KEY, title TEXT, genre TEXT)''')
    
    # Adding sample games to the system (If the database is empty)
    cursor.execute("SELECT COUNT(*) FROM games")
    if cursor.fetchone()[0] == 0:
        sample_games = [
            ("The Witcher 3", "RPG"), ("Cyberpunk 2077", "RPG"), ("Skyrim", "RPG"),
            ("Valorant", "FPS"), ("CS:GO 2", "FPS"), ("Apex Legends", "FPS"),
            ("Stardew Valley", "Simulation"), ("The Sims 4", "Simulation")
        ]
        cursor.executemany("INSERT INTO games (title, genre) VALUES (?, ?)", sample_games)
        
    conn.commit()
    return conn

# Securing the password (Using built-in hashlib instead of bcrypt)
def hash_password(password, salt=None):
    if salt is None:
        salt = os.urandom(16).hex() # Generate a random salt
    # Securing the password by hashing it 100,000 times
    pwd_hash = hashlib.pbkdf2_hmac('sha256', password.encode('utf-8'), salt.encode('utf-8'), 100000).hex()
    return pwd_hash, salt

# --- 2. GRAPHICAL USER INTERFACE (GUI) AND APP LOGIC ---

class GameApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Secure Game Recommendation System")
        self.root.geometry("400x300")
        self.conn = setup_database()
        
        self.create_login_screen()

    def clear_screen(self):
        for widget in self.root.winfo_children():
            widget.destroy()

    # --- EPIC 1: Secure User Registration and Login ---
    def create_login_screen(self):
        self.clear_screen()
        
        tk.Label(self.root, text="System Login", font=("Arial", 16, "bold")).pack(pady=10)
        
        tk.Label(self.root, text="Username:").pack()
        self.username_entry = tk.Entry(self.root)
        self.username_entry.pack(pady=5)
        
        tk.Label(self.root, text="Password:").pack()
        self.password_entry = tk.Entry(self.root, show="*") # Password is typed hidden
        self.password_entry.pack(pady=5)
        
        btn_frame = tk.Frame(self.root)
        btn_frame.pack(pady=15)
        
        tk.Button(btn_frame, text="Login", command=self.login, width=10, bg="lightgreen").pack(side=tk.LEFT, padx=5)
        tk.Button(btn_frame, text="Register", command=self.register, width=10, bg="lightblue").pack(side=tk.LEFT, padx=5)

    def register(self):
        username = self.username_entry.get()
        password = self.password_entry.get()
        
        if not username or not password:
            messagebox.showwarning("Error", "Please fill in all fields! (Input validation)")
            return
            
        pwd_hash, salt = hash_password(password)
        cursor = self.conn.cursor()
        
        try:
            # Using parameterized queries (?) for SQL Injection protection
            cursor.execute("INSERT INTO users (username, password_hash, salt) VALUES (?, ?, ?)", 
                           (username, pwd_hash, salt))
            self.conn.commit()
            messagebox.showinfo("Success", "Registration successful! You can now log in.")
        except sqlite3.IntegrityError:
            messagebox.showerror("Error", "This username is already taken!")

    def login(self):
        username = self.username_entry.get()
        password = self.password_entry.get()
        
        cursor = self.conn.cursor()
        cursor.execute("SELECT password_hash, salt FROM users WHERE username=?", (username,))
        result = cursor.fetchone()
        
        if result:
            stored_hash, salt = result
            # We hash the entered password again with the salt from the database
            pwd_hash, _ = hash_password(password, salt)
            
            if pwd_hash == stored_hash:
                self.create_recommendation_screen(username)
            else:
                messagebox.showerror("Error", "Incorrect password!")
        else:
            messagebox.showerror("Error", "User not found!")

    # --- EPIC 3: Game Recommendation System (Content-Based Filtering) ---
    def create_recommendation_screen(self, username):
        self.clear_screen()
        self.root.geometry("400x350")
        
        tk.Label(self.root, text=f"Welcome, {username}!", font=("Arial", 14, "bold"), fg="blue").pack(pady=10)
        tk.Label(self.root, text="What genre of games do you like?").pack(pady=5)
        
        self.genre_var = tk.StringVar()
        genres = ["RPG", "FPS", "Simulation"]
        self.genre_combobox = ttk.Combobox(self.root, textvariable=self.genre_var, values=genres, state="readonly")
        self.genre_combobox.pack(pady=5)
        self.genre_combobox.current(0)
        
        tk.Button(self.root, text="Recommend Games!", command=self.get_recommendations, bg="orange").pack(pady=15)
        
        self.result_text = tk.Text(self.root, height=8, width=40, state=tk.DISABLED)
        self.result_text.pack(pady=5)
        
        tk.Button(self.root, text="Logout", command=self.create_login_screen, fg="red").pack(pady=10)

    def get_recommendations(self):
        selected_genre = self.genre_var.get()
        cursor = self.conn.cursor()
        
        # Personalized recommendation algorithm (Based on category)
        cursor.execute("SELECT title FROM games WHERE genre=?", (selected_genre,))
        games = cursor.fetchall()
        
        self.result_text.config(state=tk.NORMAL)
        self.result_text.delete(1.0, tk.END)
        
        if games:
            self.result_text.insert(tk.END, f"Recommended {selected_genre} games for you:\n\n")
            for game in games:
                self.result_text.insert(tk.END, f"🎮 {game[0]}\n")
        else:
            self.result_text.insert(tk.END, "No games found in this genre.")
            
        self.result_text.config(state=tk.DISABLED)

# Code to start the program
if __name__ == "__main__":
    root = tk.Tk()
    app = GameApp(root)
    root.mainloop()
