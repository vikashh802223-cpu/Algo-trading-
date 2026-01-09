# hft_algo.py - Powerful HFT-Style Algo Trading Bot with GUI, WebSocket, Order Book Analysis
# Requirements: pip install dhanhq marketfeed customtkinter python-dotenv
# .env: DHAN_CLIENT_ID=your_id, DHAN_ACCESS_TOKEN=your_token
# Strategy: Monitor order book imbalance - if bid volume > ask, buy; else sell (simple market making)
# Run: python hft_algo.py

import customtkinter as ctk
from dotenv import load_dotenv
import os
import threading
import time
from dhanhq import dhanhq
from marketfeed import DhanFeed  # For WebSocket
import json  # For parsing feed

load_dotenv()
client_id = os.getenv("DHAN_CLIENT_ID")
access_token = os.getenv("DHAN_ACCESS_TOKEN")

dhan = dhanhq(client_id, access_token)
symbol = "1333"  # RELIANCE security_id (change as needed)
instrument = {"exchange_segment": "NSE_EQ", "security_id": symbol}

# WebSocket Setup
feed = DhanFeed(client_id, access_token)
instruments = [(1, symbol)]  # 1=NSE_EQ

ctk.set_appearance_mode("Dark")
ctk.set_default_color_theme("green")  # For trading vibe

class HFTDashboard(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title("HFT Algo Trader")
        self.geometry("900x700")

        self.status_label = ctk.CTkLabel(self, text="Status: Stopped", font=("Arial", 16))
        self.status_label.pack(pady=10)

        self.price_label = ctk.CTkLabel(self, text="LTP: --", font=("Arial", 14))
        self.price_label.pack(pady=5)

        self.book_label = ctk.CTkLabel(self, text="Order Book: Loading...", font=("Arial", 14))
        self.book_label.pack(pady=5)

        self.pnl_label = ctk.CTkLabel(self, text="P&L: 0.00", font=("Arial", 14))
        self.pnl_label.pack(pady=5)

        self.positions_text = ctk.CTkTextbox(self, height=200, width=700)
        self.positions_text.pack(pady=10)
        self.positions_text.insert("0.0", "Positions:\nNo open positions")

        self.start_button = ctk.CTkButton(self, text="Start HFT", command=self.start_hft)
        self.start_button.pack(pady=10)

        self.stop_button = ctk.CTkButton(self, text="Stop HFT", command=self.stop_hft)
        self.stop_button.pack(pady=10)

        self.running = False
        self.positions = []
        self.current_ltp = 0
        self.bid_volume = 0
        self.ask_volume = 0

    def on_open(self):
        print("WebSocket Connected")
        feed.subscribe(feed.Depth, instruments)  # Subscribe to order book depth
        feed.subscribe(feed.Ticker, instruments)  # Subscribe to ticks

    def on_message(self, message):
        data = json.loads(message)
        if 'type' == 'depth':
            # Parse order book
            bids = data.get('bids', [])
            asks = data.get('asks', [])
            self.bid_volume = sum(bid['quantity'] for bid in bids[:5]) if bids else 0  # Top 5 levels
            self.ask_volume = sum(ask['quantity'] for ask in asks[:5]) if asks else 0
            self.update_book_display(bids, asks)

        elif 'type' == 'ticker':
            self.current_ltp = data.get('ltp', 0)
            self.price_label.configure(text=f"LTP: {self.current_ltp}")

        # HFT Logic: Imbalance check
        if self.running:
            self.execute_strategy()

    def update_book_display(self, bids, asks):
        book_str = "Top Bids/Asks:\n"
        for i in range(min(5, len(bids))):
            book_str += f"Bid {i+1}: Price {bids[i]['price']}, Qty {bids[i]['quantity']}\n"
        for i in range(min(5, len(asks))):
            book_str += f"Ask {i+1}: Price {asks[i]['price']}, Qty {asks[i]['quantity']}\n"
        self.book_label.configure(text=book_str)

    def execute_strategy(self):
        imbalance = self.bid_volume - self.ask_volume
        quantity = 1  # Small for testing

        if imbalance > 1000 and not self.positions:  # Strong buy side (adjust threshold)
            order = dhan.place_order(
                security_id=symbol,
                exchange_segment="NSE_EQ",
                transaction_type="BUY",
                order_type="LIMIT",  # Or MARKET for faster
                product_type="INTRADAY",
                price=self.current_ltp - 0.05,  # Slight below LTP for fill
                quantity=quantity
            )
            self.positions.append({"entry": self.current_ltp, "type": "BUY"})
            self.update_positions()
            self.status_label.configure(text="HFT Buy Executed")

        elif imbalance < -1000 and self.positions:  # Strong sell side
            order = dhan.place_order(
                security_id=symbol,
                exchange_segment="NSE_EQ",
                transaction_type="SELL",
                order_type="LIMIT",
                product_type="INTRADAY",
                price=self.current_ltp + 0.05,  # Slight above
                quantity=quantity
            )
            entry = self.positions.pop()["entry"]
            pnl = (self.current_ltp - entry) * quantity
            self.pnl_label.configure(text=f"P&L: {pnl:.2f}")
            self.update_positions()
            self.status_label.configure(text="HFT Sell Executed")

    def update_positions(self):
        text = "Positions:\n" + "\n".join([f"{pos['type']} @ {pos['entry']}" for pos in self.positions]) or "No open positions"
        self.positions_text.delete("0.0", "end")
        self.positions_text.insert("0.0", text)

    def start_hft(self):
        if not self.running:
            self.running = True
            self.status_label.configure(text="Status: HFT Running")
            feed.on_open = self.on_open
            feed.on_message = self.on_message
            threading.Thread(target=feed.run_forever, daemon=True).start()

    def stop_hft(self):
        self.running = False
        feed.close()
        self.status_label.configure(text="Status: Stopped")

app = HFTDashboard()
app.mainloop()
