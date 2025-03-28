# deadlock-prevention-and-recovery-toolkit
OS PROJECT
import threading
import time
import sqlite3
from collections import defaultdict

# ================================
# 1️⃣ DATABASE INITIALIZATION (Create Table)
# ================================

def initialize_database():
    conn = sqlite3.connect("bank.db")
    cursor = conn.cursor()

    # Create the accounts table if it doesn't exist
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS accounts (
        id INTEGER PRIMARY KEY,
        balance INTEGER NOT NULL
    );
    """)

    # Insert initial balances if the table is empty
    cursor.execute("SELECT COUNT(*) FROM accounts;")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO accounts (id, balance) VALUES (1, 10000), (2, 5000);")

    conn.commit()
    conn.close()
    print("✅ Database initialized successfully!")

initialize_database()

# ================================
# 2️⃣ DEADLOCK PREVENTION (Fixed Resource Ordering)
# ================================

lockA = threading.Lock()
lockB = threading.Lock()

def task1():
    with lockA:  # Always lock A first
        print("Task 1 acquired Lock A")
        with lockB:
            print("Task 1 acquired Lock B")

def task2():
    with lockA:  # Always lock A first (Prevention)
        print("Task 2 acquired Lock A")
        with lockB:
            print("Task 2 acquired Lock B")

t1 = threading.Thread(target=task1)
t2 = threading.Thread(target=task2)
t1.start()
t2.start()
t1.join()
t2.join()

# ================================
# 3️⃣ DEADLOCK DETECTION (Wait-for Graph)
# ================================

class DeadlockDetector:
    def _init_(self):
        self.graph = defaultdict(list)

    def add_edge(self, process, resource):
        self.graph[process].append(resource)

    def detect_cycle(self, visited, stack, node):
        visited[node] = True
        stack[node] = True
        
        for neighbor in self.graph[node]:
            if not visited[neighbor]:
                if self.detect_cycle(visited, stack, neighbor):
                    return True
            elif stack[neighbor]:  # Cycle detected
                return True

        stack[node] = False
        return False

    def is_deadlock(self):
        visited = {node: False for node in self.graph}
        stack = {node: False for node in self.graph}

        for node in self.graph:
            if not visited[node]:
                if self.detect_cycle(visited, stack, node):
                    return True  # Deadlock detected
        return False

detector = DeadlockDetector()
detector.add_edge("P1", "R1")
detector.add_edge("P2", "R2")
detector.add_edge("R1", "P2")
detector.add_edge("R2", "P1")  # Deadlock cycle

if detector.is_deadlock():
    print("⚠️ Deadlock Detected!")

# ================================
# 4️⃣ DEADLOCK RECOVERY (Preempting Resources)
# ================================

class DeadlockRecovery:
    def _init_(self):
        self.resources = {"R1": 1, "R2": 1}
        self.allocation = {"P1": ["R1"], "P2": ["R2"]}
        self.waiting = {"P1": ["R2"], "P2": ["R1"]}  # Circular wait condition

    def detect_deadlock(self):
        for p, req in self.waiting.items():
            for r in req:
                if r in self.allocation and self.allocation[self.allocation[r][0]] == [p]:
                    return p  # Deadlocked process found
        return None

    def recover_from_deadlock(self):
        victim = self.detect_deadlock()
        if victim:
            print(f"🔴 Deadlock detected! Preempting resources from {victim}")
            for res in self.allocation[victim]:
                self.resources[res] += 1  # Release resources
            self.allocation[victim] = []  # Reset allocation
            del self.waiting[victim]  # Remove from waitlist
        else:
            print("✅ No deadlock detected.")

recovery = DeadlockRecovery()
recovery.recover_from_deadlock()

# ================================
# 5️⃣ DATABASE DEADLOCK HANDLING (Retry Mechanism)
# ================================

def execute_transaction():
    for attempt in range(3):  # Retry up to 3 times
        try:
            conn = sqlite3.connect("bank.db")
            cursor = conn.cursor()

            cursor.execute("BEGIN TRANSACTION;")
            cursor.execute("UPDATE accounts SET balance = balance - 500 WHERE id = 1;")
            cursor.execute("UPDATE accounts SET balance = balance + 500 WHERE id = 2;")
            conn.commit()

            print("✅ Transaction Successful!")
            return

        except sqlite3.OperationalError as e:
            if "deadlock" in str(e).lower():
                print(f"⚠️ Deadlock detected! Retrying... Attempt {attempt+1}")
                time.sleep(2)  # Wait and retry
            else:
                print("❌ Error:", e)
                break
        finally:
            conn.close()

execute_transaction()
