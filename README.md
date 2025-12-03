import tkinter as tk
from tkinter import ttk, filedialog
import socket
import threading
import json
import datetime

class LogViewer:
    def __init__(self, root):
        self.root = root
        self.root.title("Enhanced Device Log Viewer")
        self.root.geometry("1400x800")
        self.root.configure(bg="#1a1d29")

        # State
        self.autoscroll = tk.BooleanVar(value=True)
        self.seen_devices = set()
        self.device_logs_buffer = {}
        self.all_logs = []  # Store all logs for filtering

        # Custom fonts
        self.title_font = ("SF Pro Display", 14, "bold")
        self.text_font = ("JetBrains Mono", 10)
        self.ui_font = ("SF Pro Display", 11)

        # Color scheme
        self.colors = {
            'bg_dark': '#1a1d29',
            'bg_card': '#242837',
            'bg_hover': '#2d3348',
            'accent': '#6366f1',
            'accent_hover': '#4f46e5',
            'success': '#10b981',
            'warning': '#f59e0b',
            'error': '#ef4444',
            'text_primary': '#e2e8f0',
            'text_secondary': '#94a3b8',
            'border': '#334155'
        }

        self.create_ui()
        
        # Start UDP listener
        threading.Thread(target=self.udp_listener, daemon=True).start()

    def create_ui(self):
        # Top bar
        top = tk.Frame(self.root, bg=self.colors['bg_card'], height=80)
        top.pack(fill="x", padx=0, pady=0)
        top.pack_propagate(False)

        # Title
        title_frame = tk.Frame(top, bg=self.colors['bg_card'])
        title_frame.pack(side="left", padx=30, pady=20)
        
        tk.Label(title_frame, text="🔍", font=("SF Pro Display", 24), 
                bg=self.colors['bg_card'], fg=self.colors['accent']).pack(side="left", padx=(0, 10))
        tk.Label(title_frame, text="Log Viewer", font=("SF Pro Display", 18, "bold"),
                bg=self.colors['bg_card'], fg=self.colors['text_primary']).pack(side="left")

        # Search frame with glow effect
        search_frame = tk.Frame(top, bg=self.colors['bg_dark'], bd=0, relief="flat",
                               highlightthickness=2, highlightbackground=self.colors['border'])
        search_frame.pack(side="left", padx=20, pady=20, fill="x", expand=True)

        def on_search_focus(e):
            search_frame.config(highlightbackground=self.colors['accent'])
            
        def on_search_blur(e):
            search_frame.config(highlightbackground=self.colors['border'])

        search_inner = tk.Frame(search_frame, bg=self.colors['bg_hover'], bd=0)
        search_inner.pack(fill="both", padx=2, pady=2)

        tk.Label(search_inner, text="🔎", font=("SF Pro Display", 14),
                bg=self.colors['bg_hover'], fg=self.colors['text_secondary']).pack(side="left", padx=(15, 5))

        self.search_var = tk.StringVar()
        self.search_var.trace_add("write", lambda *_: self.apply_search_filter())
        
        search_entry = tk.Entry(search_inner, textvariable=self.search_var, 
                               font=self.ui_font, bg=self.colors['bg_hover'],
                               fg=self.colors['text_primary'], bd=0, 
                               insertbackground=self.colors['accent'],
                               relief="flat", width=40)
        search_entry.pack(side="left", padx=(0, 15), pady=12, fill="x", expand=True)
        search_entry.bind("<FocusIn>", on_search_focus)
        search_entry.bind("<FocusOut>", on_search_blur)

        # Controls frame
        controls_frame = tk.Frame(top, bg=self.colors['bg_card'])
        controls_frame.pack(side="right", padx=30, pady=20)

        # Auto-scroll with custom style and hover effect
        autoscroll_frame = tk.Frame(controls_frame, bg=self.colors['bg_hover'], bd=0,
                                   highlightthickness=1, highlightbackground=self.colors['border'])
        autoscroll_frame.pack(side="left", padx=(0, 10))
        
        self.autoscroll_btn = tk.Label(autoscroll_frame, text="✓ Auto-scroll", 
                                       font=self.ui_font, bg=self.colors['bg_hover'],
                                       fg=self.colors['success'], cursor="hand2", padx=15, pady=8)
        self.autoscroll_btn.pack()
        
        def on_autoscroll_enter(e):
            autoscroll_frame.config(highlightbackground=self.colors['accent'])
            self.autoscroll_btn.config(bg=self.colors['bg_card'])
            
        def on_autoscroll_leave(e):
            autoscroll_frame.config(highlightbackground=self.colors['border'])
            self.autoscroll_btn.config(bg=self.colors['bg_hover'])
            
        self.autoscroll_btn.bind("<Button-1>", self.toggle_autoscroll)
        self.autoscroll_btn.bind("<Enter>", on_autoscroll_enter)
        self.autoscroll_btn.bind("<Leave>", on_autoscroll_leave)

        # Buttons
        self.create_button(controls_frame, "💾 Save", self.save_logs, self.colors['accent']).pack(side="left", padx=5)
        self.create_button(controls_frame, "🗑️ Clear", self.clear_logs, self.colors['error']).pack(side="left", padx=5)

        # Main container
        main = tk.Frame(self.root, bg=self.colors['bg_dark'])
        main.pack(fill="both", expand=True, padx=20, pady=(0, 20))

        main.columnconfigure(0, weight=1, minsize=250)
        main.columnconfigure(1, weight=3)

        # Device list panel
        device_panel = tk.Frame(main, bg=self.colors['bg_card'], bd=0, 
                               highlightthickness=2, highlightbackground=self.colors['border'])
        device_panel.grid(row=0, column=0, sticky="nsew", padx=(0, 10))
        
        # Add subtle shadow effect
        device_panel.bind("<Enter>", lambda e: device_panel.config(highlightbackground=self.colors['accent']))
        device_panel.bind("<Leave>", lambda e: device_panel.config(highlightbackground=self.colors['border']))

        tk.Label(device_panel, text="DEVICES", font=("SF Pro Display", 11, "bold"),
                bg=self.colors['bg_card'], fg=self.colors['text_secondary'],
                anchor="w").pack(fill="x", padx=20, pady=(15, 10))

        list_frame = tk.Frame(device_panel, bg=self.colors['bg_hover'], bd=0)
        list_frame.pack(fill="both", expand=True, padx=15, pady=(0, 15))

        scrollbar = tk.Scrollbar(list_frame, bg=self.colors['bg_hover'], 
                                troughcolor=self.colors['bg_card'],
                                activebackground=self.colors['accent'])
        scrollbar.pack(side="right", fill="y")

        self.device_list = tk.Listbox(list_frame, exportselection=False, 
                                     font=self.ui_font, bg=self.colors['bg_hover'],
                                     fg=self.colors['text_primary'], bd=0, 
                                     selectmode=tk.SINGLE, activestyle="none",
                                     highlightthickness=0, selectbackground=self.colors['accent'],
                                     selectforeground="white", yscrollcommand=scrollbar.set)
        self.device_list.pack(side="left", fill="both", expand=True)
        scrollbar.config(command=self.device_list.yview)
        self.device_list.bind("<<ListboxSelect>>", self.on_device_select)

        # Logs panel
        logs_panel = tk.Frame(main, bg=self.colors['bg_dark'])
        logs_panel.grid(row=0, column=1, sticky="nsew")

        logs_panel.rowconfigure(0, weight=1)
        logs_panel.columnconfigure(0, weight=1)
        logs_panel.columnconfigure(1, weight=1)

        # General logs
        self.create_log_panel(logs_panel, "GENERAL LOGS", 0, is_general=True)
        
        # Device logs
        self.create_log_panel(logs_panel, "DEVICE LOGS", 1, is_general=False)

    def create_log_panel(self, parent, title, column, is_general):
        # Panel with border for card effect
        panel = tk.Frame(parent, bg=self.colors['bg_card'], bd=0,
                        highlightthickness=2, highlightbackground=self.colors['border'])
        panel.grid(row=0, column=column, sticky="nsew", padx=(0, 10) if column == 0 else (0, 0))
        
        # Hover effect for panels
        def on_panel_enter(e):
            panel.config(highlightbackground=self.colors['accent'])
            
        def on_panel_leave(e):
            panel.config(highlightbackground=self.colors['border'])
            
        panel.bind("<Enter>", on_panel_enter)
        panel.bind("<Leave>", on_panel_leave)

        # Header
        header = tk.Frame(panel, bg=self.colors['bg_card'])
        header.pack(fill="x", padx=20, pady=(15, 10))
        
        tk.Label(header, text=title, font=("SF Pro Display", 11, "bold"),
                bg=self.colors['bg_card'], fg=self.colors['text_secondary'],
                anchor="w").pack(side="left")

        # Log count badge with pulse animation on update
        count_label = tk.Label(header, text="0", font=("SF Pro Display", 9, "bold"),
                              bg=self.colors['accent'], fg="white",
                              padx=8, pady=2, bd=0)
        count_label.pack(side="right")
        
        # Store reference for badge animation
        if is_general:
            self.general_count_label = count_label
        else:
            self.device_count_label = count_label

        # Text widget container
        text_container = tk.Frame(panel, bg=self.colors['bg_hover'], bd=0)
        text_container.pack(fill="both", expand=True, padx=15, pady=(0, 15))

        scrollbar = tk.Scrollbar(text_container, bg=self.colors['bg_hover'],
                                troughcolor=self.colors['bg_card'],
                                activebackground=self.colors['accent'])
        scrollbar.pack(side="right", fill="y")

        text_widget = tk.Text(text_container, wrap="word", font=self.text_font,
                            bg=self.colors['bg_hover'], fg=self.colors['text_primary'],
                            bd=0, relief="flat", padx=15, pady=15,
                            insertbackground=self.colors['accent'],
                            selectbackground=self.colors['accent'],
                            selectforeground="white",
                            yscrollcommand=scrollbar.set)
        text_widget.pack(side="left", fill="both", expand=True)
        scrollbar.config(command=text_widget.yview)

        # Configure tags with your requested colors
        text_widget.tag_config("timestamp", foreground="#a78bfa")  # Purple
        text_widget.tag_config("device_id", foreground="#60a5fa")  # Blue
        text_widget.tag_config("info", foreground=self.colors['text_primary'])  # White/light for info
        text_widget.tag_config("warning", foreground=self.colors['warning'])  # Orange
        text_widget.tag_config("error", foreground=self.colors['error'])  # Red

        if is_general:
            self.general_logs = text_widget
            self.general_count = count_label
        else:
            self.device_logs = text_widget
            self.device_count = count_label

    def create_button(self, parent, text, command, color):
        # Button container for shadow effect
        btn_container = tk.Frame(parent, bg=self.colors['bg_card'], bd=0)
        
        btn = tk.Label(btn_container, text=text, font=self.ui_font,
                      bg=color, fg="white", cursor="hand2",
                      padx=20, pady=10, bd=0)
        btn.pack(padx=2, pady=2)
        
        def on_enter(e):
            btn.config(bg=self.lighten_color(color))
            # Lift effect
            btn.pack(padx=1, pady=1)
            
        def on_leave(e):
            btn.config(bg=color)
            # Return to normal
            btn.pack(padx=2, pady=2)
            
        def on_click(e):
            # Press effect
            btn.pack(padx=3, pady=3)
            parent.after(100, lambda: btn.pack(padx=2, pady=2))
            command()
        
        btn.bind("<Enter>", on_enter)
        btn.bind("<Leave>", on_leave)
        btn.bind("<Button-1>", on_click)
        return btn_container

    def lighten_color(self, color):
        # Simple color lightening
        color_map = {
            self.colors['accent']: self.colors['accent_hover'],
            self.colors['error']: '#dc2626',
            self.colors['success']: '#059669'
        }
        return color_map.get(color, color)
    
    def pulse_animation(self, widget, step=0):
        """Smooth pulse animation for badges"""
        if step < 5:
            scale = 1.0 + (0.15 * (1 - abs(step - 2.5) / 2.5))
            font_size = int(9 * scale)
            widget.config(font=("SF Pro Display", font_size, "bold"))
            self.root.after(50, lambda: self.pulse_animation(widget, step + 1))
        else:
            widget.config(font=("SF Pro Display", 9, "bold"))

    def toggle_autoscroll(self, event=None):
        current = self.autoscroll.get()
        self.autoscroll.set(not current)
        
        if self.autoscroll.get():
            self.autoscroll_btn.config(text="✓ Auto-scroll", fg=self.colors['success'])
        else:
            self.autoscroll_btn.config(text="✗ Auto-scroll", fg=self.colors['text_secondary'])

    def determine_level(self, msg: str):
        msg_lower = msg.lower()
        if "error" in msg_lower:
            return "error"
        if "warn" in msg_lower:
            return "warning"
        return "info"

    def udp_listener(self):
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.bind(("127.0.0.1", 9000))

        while True:
            data, _ = sock.recvfrom(4096)
            try:
                log = json.loads(data)
            except:
                continue

            device = log.get("device", "unknown")
            msg = log.get("msg", "")
            ts = datetime.datetime.now().strftime("%H:%M:%S")
            full = f"[{ts}] [{device}] {msg}"
            level = self.determine_level(msg)

            # Add device if new
            if device not in self.seen_devices:
                self.seen_devices.add(device)
                self.device_logs_buffer[device] = []
                self.root.after(0, self.device_list.insert, tk.END, f"  {device}")

            # Store log
            self.device_logs_buffer[device].append((full, level))
            self.all_logs.append((full, level))

            # Update UI
            self.root.after(0, self.append_log, self.general_logs, full, level)
            self.root.after(0, self.update_count, self.general_count, len(self.all_logs))

            # Update device logs if selected
            if self.get_selected_device() == device:
                self.root.after(0, self.update_device_logs)

    def append_log(self, text_widget, text, level):
        try:
            # Parse the log format: [timestamp] [device_id] message
            parts = text.split("] ", 2)
            if len(parts) >= 3:
                timestamp = parts[0] + "]"
                device_id = "[" + parts[1] + "]"
                message = parts[2]
                
                # Insert with colors: purple timestamp, blue device_id, colored message
                text_widget.insert(tk.END, timestamp + " ", "timestamp")
                text_widget.insert(tk.END, device_id + " ", "device_id")
                text_widget.insert(tk.END, message + "\n", level)
            else:
                # Fallback if format doesn't match
                text_widget.insert(tk.END, text + "\n", level)

            if self.autoscroll.get():
                text_widget.see(tk.END)
        except:
            pass

    def update_count(self, label, count):
        old_count = label.cget("text")
        label.config(text=str(count))
        
        # Pulse animation when count increases
        if str(count) != old_count:
            self.pulse_animation(label)

    def on_device_select(self, event):
        self.update_device_logs()

    def get_selected_device(self):
        sel = self.device_list.curselection()
        if sel:
            device = self.device_list.get(sel[0]).strip()
            return device
        return None

    def update_device_logs(self):
        device = self.get_selected_device()
        self.device_logs.delete(1.0, tk.END)

        if device and device in self.device_logs_buffer:
            logs = self.device_logs_buffer[device]
            for entry, level in logs:
                self.append_log(self.device_logs, entry, level)
            self.update_count(self.device_count, len(logs))

        if self.autoscroll.get():
            self.device_logs.see(tk.END)

    def apply_search_filter(self):
        query = self.search_var.get().lower().strip()
        
        self.general_logs.delete(1.0, tk.END)
        
        if not query:
            # Show all logs
            for entry, level in self.all_logs:
                self.append_log(self.general_logs, entry, level)
        else:
            # Filter logs
            filtered = [(entry, level) for entry, level in self.all_logs if query in entry.lower()]
            for entry, level in filtered:
                self.append_log(self.general_logs, entry, level)
            self.update_count(self.general_count, len(filtered))

    def save_logs(self):
        path = filedialog.asksaveasfilename(defaultextension=".txt",
                                           filetypes=[("Text files", "*.txt"), ("All files", "*.*")])
        if not path:
            return

        with open(path, "w", encoding="utf-8") as f:
            f.write("="*80 + "\n")
            f.write("DEVICE LOG EXPORT\n")
            f.write(f"Generated: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
            f.write("="*80 + "\n\n")
            
            for dev, logs in self.device_logs_buffer.items():
                f.write(f"\n{'='*80}\n")
                f.write(f"DEVICE: {dev}\n")
                f.write(f"{'='*80}\n")
                for line, _ in logs:
                    f.write(line + "\n")
                f.write("\n")

    def clear_logs(self):
        self.general_logs.delete(1.0, tk.END)
        self.device_logs.delete(1.0, tk.END)
        self.device_logs_buffer.clear()
        self.all_logs.clear()
        self.device_list.delete(0, tk.END)
        self.seen_devices.clear()
        self.update_count(self.general_count, 0)
        self.update_count(self.device_count, 0)


if __name__ == "__main__":
    root = tk.Tk()
    app = LogViewer(root)
    root.mainloop()
