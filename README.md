import sqlite3
import tkinter as tk
from tkinter import messagebox, ttk
from collections import deque

# ==========================================
# 1. DATABASE SETUP (SQLite)
# ==========================================
def init_db():
    conn = sqlite3.connect("campus.db")
    cursor = conn.cursor()
    
    # Create table for campus locations
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS locations (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE,
            block TEXT,
            floor TEXT,
            description TEXT
        )
    """)
    
    # Sample campus data
    sample_data = [
        ("Main Gate", "Entrance", "Ground Floor", "Main entry and security check"),
        ("Admin Block", "Administrative Building", "Ground Floor", "Principal Office, Admissions, Accounts"),
        ("Library", "Main Building", "Second Floor", "Central library and reading hall"),
        ("Computer Lab 1", "CS Block", "Ground Floor", "BCA & MCA programming lab"),
        ("Seminar Hall", "Main Building", "First Floor", "Events and guest lectures"),
        ("Canteen", "South Wing", "Ground Floor", "Cafeteria and seating area")
    ]
    
    cursor.executemany("""
        INSERT OR IGNORE INTO locations (name, block, floor, description)
        VALUES (?, ?, ?, ?)
    """, sample_data)
    
    conn.commit()
    conn.close()

# ==========================================
# 2. NAVIGATION GRAPH (Pathfinding)
# ==========================================
# Define which locations connect directly to each other
campus_graph = {
    "Main Gate": ["Admin Block", "Canteen"],
    "Admin Block": ["Main Gate", "Library", "Seminar Hall"],
    "Canteen": ["Main Gate", "Computer Lab 1"],
    "Seminar Hall": ["Admin Block", "Library"],
    "Library": ["Admin Block", "Seminar Hall", "Computer Lab 1"],
    "Computer Lab 1": ["Canteen", "Library"]
}

def find_shortest_route(start, end):
    if start == end:
        return [start]
    visited = set()
    queue = deque([[start]])
    
    while queue:
        path = queue.popleft()
        node = path[-1]
        if node not in visited:
            neighbors = campus_graph.get(node, [])
            for neighbor in neighbors:
                new_path = list(path)
                new_path.append(neighbor)
                queue.append(new_path)
                if neighbor == end:
                    return new_path
            visited.add(node)
    return None

# ==========================================
# 3. GUI APPLICATION (Tkinter)
# ==========================================
class CollegeNavApp:
    def __init__(self, root):
        self.root = root
        self.root.title("College Navigation System")
        self.root.geometry("600x520")
        self.root.configure(bg="#f4f6f9")
        
        self.locations = self.fetch_locations()
        self.location_names = [loc[0] for loc in self.locations]
        
        self.build_ui()

    def fetch_locations(self):
        conn = sqlite3.connect("campus.db")
        cursor = conn.cursor()
        cursor.execute("SELECT name, block, floor, description FROM locations")
        rows = cursor.fetchall()
        conn.close()
        return rows

    def build_ui(self):
        # Header
        header = tk.Label(
            self.root, text="Campus Navigator", 
            font=("Helvetica", 18, "bold"), bg="#1e3a8a", fg="white", pady=15
        )
        header.pack(fill="x")

        # --- Section 1: Location Details ---
        search_frame = tk.LabelFrame(self.root, text="Find Location Details", font=("Helvetica", 11, "bold"), bg="#f4f6f9", padx=15, pady=10)
        search_frame.pack(fill="x", padx=20, pady=15)

        tk.Label(search_frame, text="Select Place:", bg="#f4f6f9").grid(row=0, column=0, sticky="w")
        self.detail_combo = ttk.Combobox(search_frame, values=self.location_names, state="readonly", width=30)
        self.detail_combo.grid(row=0, column=1, padx=10)
        self.detail_combo.current(0)

        tk.Button(search_frame, text="View Info", command=self.show_details, bg="#2563eb", fg="white", padx=10).grid(row=0, column=2)

        self.info_label = tk.Label(search_frame, text="Select a location to see block and floor info.", bg="#f4f6f9", fg="#333", justify="left", wraplength=500)
        self.info_label.grid(row=1, column=0, columnspan=3, sticky="w", pady=(10, 0))

        # --- Section 2: Route Directions ---
        nav_frame = tk.LabelFrame(self.root, text="Get Directions", font=("Helvetica", 11, "bold"), bg="#f4f6f9", padx=15, pady=10)
        nav_frame.pack(fill="x", padx=20, pady=5)

        tk.Label(nav_frame, text="Start Point:", bg="#f4f6f9").grid(row=0, column=0, sticky="w", pady=5)
        self.start_combo = ttk.Combobox(nav_frame, values=self.location_names, state="readonly", width=30)
        self.start_combo.grid(row=0, column=1, padx=10, pady=5)
        self.start_combo.current(0)

        tk.Label(nav_frame, text="Destination:", bg="#f4f6f9").grid(row=1, column=0, sticky="w", pady=5)
        self.end_combo = ttk.Combobox(nav_frame, values=self.location_names, state="readonly", width=30)
        self.end_combo.grid(row=1, column=1, padx=10, pady=5)
        self.end_combo.current(3)

        tk.Button(nav_frame, text="Find Route", command=self.calculate_route, bg="#16a34a", fg="white", padx=12).grid(row=0, column=2, rowspan=2, padx=5)

        # --- Output Box ---
        result_frame = tk.LabelFrame(self.root, text="Route Instructions", font=("Helvetica", 11, "bold"), bg="#f4f6f9", padx=15, pady=10)
        result_frame.pack(fill="both", expand=True, padx=20, pady=15)

        self.route_text = tk.Text(result_frame, height=6, font=("Helvetica", 10), state="disabled", bg="white")
        self.route_text.pack(fill="both", expand=True)

    def show_details(self):
        selected = self.detail_combo.get()
        for loc in self.locations:
            if loc[0] == selected:
                info = f"Building/Block: {loc[1]}  |  Floor: {loc[2]}\nDetails: {loc[3]}"
                self.info_label.config(text=info, fg="#1e3a8a")
                break

    def calculate_route(self):
        start = self.start_combo.get()
        end = self.end_combo.get()
        route = find_shortest_route(start, end)

        self.route_text.config(state="normal")
        self.route_text.delete("1.0", tk.END)

        if route:
            if len(route) == 1:
                self.route_text.insert(tk.END, f"You are already at {start}.")
            else:
                steps = "\n  ↓\n".join([f"Step {i+1}: {place}" for i, place in enumerate(route)])
                self.route_text.insert(tk.END, f"Shortest Path from {start} to {end}:\n\n{steps}")
        else:
            self.route_text.insert(tk.END, "No direct path found between these locations.")

        self.route_text.config(state="disabled")

# ==========================================
# 4. RUN THE APPLICATION
# ==========================================
if __name__ == "__main__":
    init_db()
    root = tk.Tk()
    app = CollegeNavApp(root)
    root.mainloop()
