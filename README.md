# Strive Tech v6.0 - Final Version
# Light Gray Calendar, Professional Color Scheme, Export Button, BMI, ACWR
# Author: Strive Tech | For Coach Amine Najji

import customtkinter as ctk
import tkinter as tk
from tkinter import messagebox
from tkcalendar import Calendar
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from datetime import datetime, timedelta
import os

# -------------------------------
# Global Settings
# -------------------------------
ctk.set_appearance_mode("Dark")
ctk.set_default_color_theme("blue")

# -------------------------------
# Data Storage
# -------------------------------
athletes = {}  # {id: {'name': str, 'age': int, 'height': float, 'weight': float, 'sessions': {}}}
current_athlete_id = None

# -------------------------------
# Calculation Functions
# -------------------------------
def calculate_load(duration, rpe):
    return duration * rpe

def calculate_acwr(sessions):
    today = datetime.now()
    weeks = [[] for _ in range(4)]  # Last 4 weeks
    for date_str, data in sessions.items():
        try:
            session_date = datetime.strptime(date_str, '%Y-%m-%d')
            if today - session_date < timedelta(weeks=4):
                week_num = (today - session_date).days // 7
                if 0 <= week_num < 4:
                    weeks[week_num].append(data['load'])
        except:
            continue
    acute = sum(weeks[0]) if weeks[0] else 0
    chronic_weeks = [sum(w) for w in weeks[1:] if w]
    chronic = sum(chronic_weeks) / len(chronic_weeks) if chronic_weeks else 1
    acwr = acute / chronic if chronic > 0 else 0
    return round(acute, 2), round(chronic, 2), round(acwr, 2)

def evaluate_acwr(acwr):
    if acwr < 0.8:
        return "Undertraining", "#1f77b4"  # Blue
    elif 0.8 <= acwr <= 1.3:
        return "Optimal Zone", "#2ca02c"  # Green
    elif 1.3 < acwr <= 1.5:
        return "Caution: Fatigue", "#ffbb00"  # Yellow
    else:
        return "High Injury Risk!", "#d62728"  # Orange/Red

def calculate_bmi(weight, height_cm):
    height_m = height_cm / 100
    return round(weight / (height_m ** 2), 2)

# -------------------------------
# Main Application
# -------------------------------
class StriveTechApp(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title("Strive Tech 💙")
        self.geometry("1400x900")
        self.resizable(True, True)

        # Grid layout
        self.grid_columnconfigure(1, weight=1)
        self.grid_rowconfigure(0, weight=1)

        # Sidebar - Calendar (Light Gray Theme)
        self.sidebar = ctk.CTkFrame(self, width=700, corner_radius=0, fg_color="#1e293b")
        self.sidebar.grid(row=0, column=0, sticky="nswe")

        self.cal_label = ctk.CTkLabel(
            self.sidebar,
            text="Select a Date to Log Session",
            font=ctk.CTkFont(size=16, weight="bold"),
            text_color="#e2e8f0"
        )
        self.cal_label.pack(pady=15)

        # Calendar with Light Gray Design
        self.cal = Calendar(
            self.sidebar,
            selectmode='day',
            date_pattern='yyyy-mm-dd',
            background="#f1f5f9",           # Light gray background
            foreground="black",             # Black text
            bordercolor="#cbd5e1",
            headersbackground="#e2e8f0",    # Header bg
            headersforeground="black",      # Header text
            selectbackground="#4361ee",     # Selected day (blue)
            selectforeground="white",
            normalbackground="#f8fafc",
            weekendbackground="#f1f5f9",
            othermonthforeground="gray",
            othermonthwebackground="#f1f5f9",
            font=("Segoe UI", 10),
            showweeknumbers=False
        )
        self.cal.pack(fill="both", expand=True, padx=25, pady=10, ipadx=10, ipady=10)

        self.cal.bind("<<CalendarSelected>>", self.open_session_dialog)

        # Buttons Frame
        btn_frame = ctk.CTkFrame(self.sidebar, fg_color="transparent")
        btn_frame.pack(pady=10, padx=20, fill="x")

        self.btn_new = ctk.CTkButton(
            btn_frame,
            text="➕ New Athlete",
            command=self.new_athlete,
            fg_color="#4361ee",
            hover_color="#3a5bc7",
            height=40
        )
        self.btn_new.pack(side="top", pady=5)

        self.refresh_athlete_list()

        # Main Frame - Results
        self.main_frame = ctk.CTkFrame(self, corner_radius=15, fg_color="#10182f")
        self.main_frame.grid(row=0, column=1, padx=20, pady=20, sticky="nswe")
        self.main_frame.grid_columnconfigure(0, weight=1)
        self.main_frame.grid_rowconfigure(4, weight=1)

        self.main_label = ctk.CTkLabel(
            self.main_frame,
            text="Select an athlete from the list",
            font=ctk.CTkFont(size=16, weight="bold"),
            text_color="#a9d6e5"
        )
        self.main_label.grid(row=0, column=0, pady=20)

        self.canvas_chart = None
        self.create_nav_buttons()
        self.create_export_button()

    def refresh_athlete_list(self):
        if not hasattr(self, 'listbox'):
            self.listbox_frame = ctk.CTkFrame(self.sidebar, fg_color="transparent")
            self.listbox_frame.pack(pady=10, padx=20, fill="x")
            self.listbox = tk.Listbox(
                self.listbox_frame,
                bg="#334155",
                fg="#e2e8f0",
                selectbackground="#4361ee",
                selectforeground="white",
                font=("Segoe UI", 12),
                height=6,
                borderwidth=0,
                highlightthickness=0
            )
            self.listbox.pack(side="left", fill="x", expand=True)
            scrollbar = tk.Scrollbar(self.listbox_frame, orient="vertical", command=self.listbox.yview)
            scrollbar.pack(side="right", fill="y")
            self.listbox.config(yscrollcommand=scrollbar.set)
            self.listbox.bind('<<ListboxSelect>>', self.select_athlete)

        self.listbox.delete(0, tk.END)
        for aid, data in athletes.items():
            self.listbox.insert(tk.END, data['name'])

    def select_athlete(self, event):
        try:
            selection = self.listbox.curselection()
            if not selection:
                return
            selected_name = self.listbox.get(selection[0])
            for aid, data in athletes.items():
                if data['name'] == selected_name:
                    global current_athlete_id
                    current_athlete_id = aid
                    self.main_label.configure(text=f"Athlete: {data['name']}")
                    self.plot_load_rpe(aid)
                    break
        except Exception as e:
            messagebox.showerror("Error", f"Failed to load athlete: {e}")

    def new_athlete(self):
        top = ctk.CTkToplevel(self)
        top.title("New Athlete")
        top.geometry("450x400")
        top.resizable(False, False)
        top.transient(self)
        top.grab_set()

        ctk.CTkLabel(top, text="Add New Athlete", font=ctk.CTkFont(size=18, weight="bold")).pack(pady=15)

        ctk.CTkLabel(top, text="Full Name:").pack(anchor="w", padx=50, pady=(5, 0))
        name_entry = ctk.CTkEntry(top, width=300)
        name_entry.pack(pady=5)

        ctk.CTkLabel(top, text="Age (years):").pack(anchor="w", padx=50, pady=(5, 0))
        age_entry = ctk.CTkEntry(top, width=300)
        age_entry.pack(pady=5)

        ctk.CTkLabel(top, text="Height (cm):").pack(anchor="w", padx=50, pady=(5, 0))
        height_entry = ctk.CTkEntry(top, width=300)
        height_entry.pack(pady=5)

        ctk.CTkLabel(top, text="Weight (kg):").pack(anchor="w", padx=50, pady=(5, 0))
        weight_entry = ctk.CTkEntry(top, width=300)
        weight_entry.pack(pady=5)

        def save():
            name = name_entry.get().strip()
            try:
                age = int(age_entry.get())
                height = float(height_entry.get())
                weight = float(weight_entry.get())
                if not name or age < 5 or height < 50 or weight < 30:
                    messagebox.showerror("Error", "Please fill all fields correctly.")
                    return
                athlete_id = f"Athlete-{len(athletes) + 1}"
                athletes[athlete_id] = {
                    'name': name,
                    'age': age,
                    'height': height,
                    'weight': weight,
                    'sessions': {}
                }
                self.refresh_athlete_list()
                top.destroy()
                messagebox.showinfo("Success", f"Athlete {name} added!")
            except:
                messagebox.showerror("Error", "Invalid input.")

        ctk.CTkButton(top, text="💾 Save Athlete", command=save, fg_color="#4cc9f0").pack(pady=20)

    def open_session_dialog(self, event):
        if not current_athlete_id:
            messagebox.showwarning("Warning", "Please select an athlete first.")
            return
        date = self.cal.get_date()
        data = athletes[current_athlete_id]['sessions'].get(date, {})

        top = ctk.CTkToplevel(self)
        top.title(f"Session - {date}")
        top.geometry("350x250")
        top.transient(self)
        top.grab_set()

        ctk.CTkLabel(top, text=f"Date: {date}", font=ctk.CTkFont(weight="bold")).pack(pady=10)

        ctk.CTkLabel(top, text="Duration (min):").pack(pady=5)
        duration_entry = ctk.CTkEntry(top, placeholder_text="e.g., 60")
        duration_entry.insert(0, data.get('duration', ''))
        duration_entry.pack(pady=5)

        ctk.CTkLabel(top, text="RPE (1-10):").pack(pady=5)
        rpe_entry = ctk.CTkEntry(top, placeholder_text="e.g., 7")
        rpe_entry.insert(0, data.get('rpe', ''))
        rpe_entry.pack(pady=5)

        def save():
            try:
                duration = int(duration_entry.get())
                rpe = float(rpe_entry.get())
                if not (1 <= rpe <= 10):
                    raise ValueError
                load = calculate_load(duration, rpe)
                athletes[current_athlete_id]['sessions'][date] = {
                    'duration': duration, 'rpe': rpe, 'load': load
                }
                top.destroy()
                messagebox.showinfo("Saved", "Session updated!")
                self.plot_load_rpe(current_athlete_id)
            except:
                messagebox.showerror("Error", "Enter valid numbers.")

        ctk.CTkButton(top, text="Save Session", command=save).pack(pady=20)

    def clear_canvas(self):
        if self.canvas_chart:
            self.canvas_chart.get_tk_widget().destroy()

    def plot_load_rpe(self, athlete_id):
        self.clear_canvas()
        data = athletes[athlete_id]
        dates = sorted(data['sessions'].keys())
        loads = [data['sessions'][d]['load'] for d in dates]
        rpes = [data['sessions'][d]['rpe'] for d in dates]

        fig, ax = plt.subplots(figsize=(10, 4), dpi=100)
        ax.bar(dates, loads, color='#4361ee', alpha=0.8, label='Training Load')
        ax2 = ax.twinx()
        ax2.plot(dates, rpes, color='#4cc9f0', marker='o', linewidth=2.5, label='RPE')
        ax.set_title('Daily Load & RPE', color='white', fontweight='bold')
        ax.tick_params(colors='white')
        ax2.tick_params(colors='white')
        fig.patch.set_facecolor('#10182f')
        ax.set_facecolor('#10182f')
        ax2.set_facecolor('#10182f')
        plt.setp(ax.get_xticklabels(), rotation=45)

        self.canvas_chart = FigureCanvasTkAgg(fig, self.main_frame)
        self.canvas_chart.get_tk_widget().grid(row=2, column=0, pady=10, sticky="ew")

    def plot_acwr_trend(self, athlete_id):
        self.clear_canvas()
        data = athletes[athlete_id]
        dates = sorted(data['sessions'].keys())
        acwr_values = []
        for d in dates:
            sessions = {k: v for k, v in data['sessions'].items() if k <= d}
            _, _, acwr = calculate_acwr(sessions)
            acwr_values.append(acwr)

        fig, ax = plt.subplots(figsize=(10, 4), dpi=100)
        ax.plot(dates, acwr_values, color='#2ca02c', linewidth=3, marker='o', label='ACWR')
        ax.axhline(y=0.8, color='#1f77b4', linestyle='--', label='Lower Limit (Blue)')
        ax.axhline(y=1.3, color='#ffbb00', linestyle='--', label='Upper Limit (Yellow)')
        ax.axhline(y=1.5, color='#d62728', linestyle='--', label='High Risk (Orange)')
        ax.set_title('ACWR Trend', color='white', fontweight='bold')
        ax.tick_params(colors='white')
        fig.patch.set_facecolor('#10182f')
        ax.set_facecolor('#10182f')
        ax.legend(loc='upper right', facecolor='#10182f', edgecolor='none', labelcolor='white')

        self.canvas_chart = FigureCanvasTkAgg(fig, self.main_frame)
        self.canvas_chart.get_tk_widget().grid(row=2, column=0, pady=10, sticky="ew")

    def plot_weekly_pie(self, athlete_id):
        self.clear_canvas()
        data = athletes[athlete_id]
        total_load = sum(s['load'] for s in data['sessions'].values())
        if total_load == 0:
            return
        categories = ['Cardio', 'Strength', 'Skill', 'Recovery']
        values = [total_load * 0.3, total_load * 0.4, total_load * 0.2, total_load * 0.1]

        fig, ax = plt.subplots(figsize=(6, 6), subplot_kw=dict(aspect="equal"))
        colors_pie = ['#4361ee', '#2ca02c', '#ffbb00', '#d62728']
        wedges, texts, autotexts = ax.pie(
            values, labels=categories, autopct='%1.1f%%', startangle=90,
            colors=colors_pie, wedgeprops=dict(width=0.4)
        )
        for text in texts + autotexts:
            text.set_color('white')
        ax.set_title('Weekly Load Distribution', color='white', fontweight='bold')

        fig.patch.set_facecolor('#10182f')
        ax.set_facecolor('#10182f')

        self.canvas_chart = FigureCanvasTkAgg(fig, self.main_frame)
        self.canvas_chart.get_tk_widget().grid(row=2, column=0, pady=10, sticky="n")

    def create_nav_buttons(self):
        nav_frame = ctk.CTkFrame(self.main_frame, fg_color="transparent")
        nav_frame.grid(row=1, column=0, pady=10, sticky="ew")
        nav_frame.grid_columnconfigure((0, 1, 2), weight=1)

        ctk.CTkButton(nav_frame, text="📊 Load & RPE", command=lambda: self.plot_load_rpe(current_athlete_id)).grid(row=0, column=0, padx=5)
        ctk.CTkButton(nav_frame, text="📈 ACWR Trend", command=lambda: self.plot_acwr_trend(current_athlete_id)).grid(row=0, column=1, padx=5)
        ctk.CTkButton(nav_frame, text="🍩 Weekly Load", command=lambda: self.plot_weekly_pie(current_athlete_id)).grid(row=0, column=2, padx=5)

    def create_export_button(self):
        export_frame = ctk.CTkFrame(self.main_frame, fg_color="transparent")
        export_frame.grid(row=3, column=0, pady=20)

        ctk.CTkButton(
            export_frame,
            text="📥 Download Full Report",
            command=self.export_report,
            fg_color="#4cc9f0",
            hover_color="#41b8d5",
            font=ctk.CTkFont(size=14, weight="bold"),
            width=250,
            height=45
        ).pack()

    def export_report(self):
        if not current_athlete_id:
            messagebox.showwarning("Warning", "Please select an athlete first.")
            return
        data = athletes[current_athlete_id]
        bmi = calculate_bmi(data['weight'], data['height'])
        filename = f"Report_{data['name']}_{datetime.now().strftime('%Y%m%d')}.pdf"

        from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Table, TableStyle, Image
        from reportlab.lib.pagesizes import A4
        from reportlab.lib.styles import getSampleStyleSheet
        from reportlab.lib import colors

        doc = SimpleDocTemplate(filename, pagesize=A4)
        styles = getSampleStyleSheet()
        flowables = []

        # Title
        flowables.append(Paragraph(f"<font size='24' color='blue'><b>Strive Tech Report</b></font>", styles['Title']))
        flowables.append(Spacer(1, 20))

        # Athlete Info + BMI
        table_data = [
            ["Name", "Age", "Height", "Weight", "BMI"],
            [data['name'], f"{data['age']} y", f"{data['height']} cm", f"{data['weight']} kg", f"{bmi}"]
        ]
        table = Table(table_data, colWidths=[100, 60, 70, 70, 60])
        table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.blue),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 12),
            ('BACKGROUND', (0, 1), (-1, 1), colors.lightblue),
            ('GRID', (0, 0), (-1, -1), 1, colors.black)
        ]))
        flowables.append(table)
        flowables.append(Spacer(1, 25))

        # Charts
        if hasattr(self, 'canvas_chart') and self.canvas_chart:
            img_path = "temp_chart_report.png"
            self.canvas_chart.figure.savefig(img_path, dpi=150, bbox_inches='tight', facecolor='#10182f')
            flowables.append(Paragraph("<b>Training Load & RPE</b>", styles['Heading2']))
            flowables.append(Image(img_path, width=480, height=280))
            flowables.append(Spacer(1, 20))

        # Analysis
        _, _, acwr = calculate_acwr(data['sessions'])
        status, acwr_color = evaluate_acwr(acwr)
        analysis = f"""
        <b>Performance Analysis:</b><br/>
        • ACWR = {acwr:.2f} → <font color='{acwr_color}'><b>{status}</b></font><br/>
        • BMI = {bmi} → <b>{'Normal' if 18.5 <= bmi <= 24.9 else 'Needs Attention'}</b><br/>
        <br/>
        <i>Recommendation: { 'Maintain current load.' if acwr <= 1.3 else 'Reduce training load to prevent injury.' }</i>
        """
        flowables.append(Paragraph(analysis, styles['Normal']))

        doc.build(flowables)
        messagebox.showinfo("Success", f"✅ Report saved:\n{os.path.abspath(filename)}")
        if os.path.exists("temp_chart_report.png"):
            os.remove("temp_chart_report.png")

# -------------------------------
# Run App
# -------------------------------
if __name__ == "__main__":
    app = StriveTechApp()
    app.mainloop()
