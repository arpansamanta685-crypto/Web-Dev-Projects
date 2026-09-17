# Voice-Controlled-Calculator






Full-stack web development projects showcasing HTML, CSS, JavaScript, and backend technologies. Includes REST APIs, database integration, and modern web frameworks.
import tkinter as tk
from tkinter import ttk, messagebox
import speech_recognition as sr
import pyttsx3
import threading
import operator
import re
import ast
import math

class VoiceCalculator:
    def __init__(self, root):
        self.root = root
        self.root.title("Voice-Controlled Calculator")
        self.root.geometry("400x600")
        self.root.resizable(False, False)
        self.recognizer = sr.Recognizer()
        self.microphone = sr.Microphone()
        self.tts_engine = pyttsx3.init()
        self.tts_engine.setProperty('rate', 150)
        self.tts_engine.setProperty('volume', 0.9)
        self.current_expression = ""
        self.result_var = tk.StringVar()
        self.result_var.set("0")
        self.is_listening = False
        self.setup_gui()
        self.calibrate_microphone()

    def setup_gui(self):
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        display_frame = ttk.Frame(main_frame)
        display_frame.grid(row=0, column=0, columnspan=5, pady=(0, 10), sticky=(tk.W, tk.E))
        result_display = tk.Entry(display_frame, textvariable=self.result_var,
                                 font=('Arial', 16), state='readonly', justify='right')
        result_display.grid(row=0, column=0, columnspan=5, sticky=(tk.W, tk.E), pady=(0, 5))
        self.expression_label = tk.Label(display_frame, text="",
                                         font=('Arial', 10), anchor='e', bg='white', relief='sunken')
        self.expression_label.grid(row=1, column=0, columnspan=5, sticky=(tk.W, tk.E), pady=(0, 10))
        voice_frame = ttk.Frame(main_frame)
        voice_frame.grid(row=1, column=0, columnspan=5, pady=(0, 10), sticky=(tk.W, tk.E))
        self.voice_btn = tk.Button(voice_frame, text="🎤 Start Voice Input",
                                   command=self.toggle_voice_recognition,
                                   bg='lightgreen', font=('Arial', 10, 'bold'))
        self.voice_btn.grid(row=0, column=0, padx=(0, 5))
        self.speak_btn = tk.Button(voice_frame, text="🔊 Speak Result",
                                   command=self.speak_result,
                                   bg='lightblue', font=('Arial', 10, 'bold'))
        self.speak_btn.grid(row=0, column=1)
        self.status_label = tk.Label(main_frame, text="Ready",
                                    font=('Arial', 9), fg='green')
        self.status_label.grid(row=2, column=0, columnspan=5, pady=(0, 10))
        self.create_calculator_buttons(main_frame)
        main_frame.columnconfigure(0, weight=1)
        display_frame.columnconfigure(0, weight=1)

    def create_calculator_buttons(self, parent):
        buttons = [
            ['C', '←', '±', '/'],
            ['7', '8', '9', '*'],
            ['4', '5', '6', '-'],
            ['1', '2', '3', '+'],
            ['0', '.', '(', ')'],
            ['√', '^', '=', 'AC']
        ]
        for i, row in enumerate(buttons):
            for j, text in enumerate(row):
                btn = tk.Button(parent, text=text, width=5, height=2,
                                font=('Arial', 12, 'bold'),
                                command=lambda t=text: self.button_click(t))
                if text in ['=']:
                    btn.config(bg='orange', fg='white')
                elif text in ['+', '-', '*', '/', '^', '√']:
                    btn.config(bg='lightgray')
                elif text in ['C', 'AC', '←', '±']:
                    btn.config(bg='lightcoral')
                else:
                    btn.config(bg='white')
                btn.grid(row=i+3, column=j, padx=2, pady=2, sticky='nsew')
        for i in range(6):
            parent.rowconfigure(i+3, weight=1)
        for j in range(4):
            parent.columnconfigure(j, weight=1)

    def calibrate_microphone(self):
        def calibrate():
            try:
                with self.microphone as source:
                    self.recognizer.adjust_for_ambient_noise(source, duration=1)
                self.update_status("Microphone calibrated", "green")
            except Exception as e:
                self.update_status(f"Microphone error: {str(e)}", "red")
        threading.Thread(target=calibrate, daemon=True).start()

    def update_status(self, message, color="black"):
        self.status_label.config(text=message, fg=color)

    def button_click(self, char):
        current = self.result_var.get()
        if char == 'C':
            self.current_expression = ""
            self.result_var.set("0")
            self.expression_label.config(text="")
        elif char == 'AC':
            self.current_expression = ""
            self.result_var.set("0")
            self.expression_label.config(text="")
        elif char == '←':
            if len(self.current_expression) > 0:
                self.current_expression = self.current_expression[:-1]
                if len(self.current_expression) == 0:
                    self.result_var.set("0")
                    self.expression_label.config(text="")
                else:
                    self.update_display()
        elif char == '±':
            try:
                current_val = float(current)
                self.result_var.set(str(-current_val))
            except:
                pass
        elif char == '=':
            self.calculate_expression()
        elif char == '√':
            try:
                val = float(current)
                if val >= 0:
                    result = math.sqrt(val)
                    self.result_var.set(str(result))
                    self.current_expression = str(result)
                    self.expression_label.config(text=f"√{val}")
                else:
                    self.result_var.set("Error")
            except:
                self.result_var.set("Error")
        elif char == '^':
            self.current_expression += "**"
            self.update_display()
        else:
            if current == "0" and char.isdigit():
                self.current_expression = char
            else:
                self.current_expression += char
            self.update_display()

    def update_display(self):
        if len(self.current_expression) == 0:
            self.result_var.set("0")
            self.expression_label.config(text="")
        else:
            self.expression_label.config(text=self.current_expression)
            try:
                preview = self.safe_evaluate(self.current_expression)
                if preview is not None:
                    self.result_var.set(str(preview))
            except:
                if self.current_expression[-1:] in "+-*/":
                    self.result_var.set(self.current_expression)
                else:
                    self.result_var.set(self.current_expression)

    def calculate_expression(self):
        if len(self.current_expression) == 0:
            return
        try:
            result = self.safe_evaluate(self.current_expression)
            if result is not None:
                self.result_var.set(str(result))
                self.expression_label.config(text=f"{self.current_expression} =")
                self.current_expression = str(result)
            else:
                self.result_var.set("Error")
                self.update_status("Invalid expression", "red")
        except Exception as e:
            self.result_var.set("Error")
            self.update_status(f"Calculation error: {str(e)}", "red")

    def safe_evaluate(self, expression):
        try:
            expression = expression.replace("x", "*").replace("×", "*").replace("÷", "/")
            expression = expression.replace(" ", "")
            allowed_chars = set('0123456789+-*/.()sqrt')
            if not set(expression.lower()).issubset(allowed_chars.union({'e', 'pi'})):
                try:
                    tree = ast.parse(expression, mode='eval')
                    for node in ast.walk(tree):
                        if isinstance(node, (ast.Call, ast.Import, ast.ImportFrom)):
                            return None
                    result = eval(compile(tree, '<string>', 'eval'))
                    return result
                except:
                    return None
            else:
                return eval(expression)
        except:
            return None

    def toggle_voice_recognition(self):
        if not self.is_listening:
            self.start_voice_recognition()
        else:
            self.stop_voice_recognition()

    def start_voice_recognition(self):
        self.is_listening = True
        self.voice_btn.config(text="🎤 Listening...", bg='red')
        self.update_status("Listening for voice input...", "blue")
        def listen_for_voice():
            try:
                with self.microphone as source:
                    audio = self.recognizer.listen(source, timeout=5, phrase_time_limit=10)
                text = self.recognizer.recognize_google(audio).lower()
                self.update_status(f"Heard: {text}", "green")
                self.process_voice_command(text)
            except sr.WaitTimeoutError:
                self.update_status("Listening timeout", "orange")
            except sr.UnknownValueError:
                self.update_status("Could not understand audio", "orange")
            except sr.RequestError as e:
                self.update_status(f"Speech recognition error: {e}", "red")
            except Exception as e:
                self.update_status(f"Error: {e}", "red")
            finally:
                self.stop_voice_recognition()
        threading.Thread(target=listen_for_voice, daemon=True).start()

    def stop_voice_recognition(self):
        self.is_listening = False
        self.voice_btn.config(text="🎤 Start Voice Input", bg='lightgreen')

    def process_voice_command(self, text):
        text = text.lower().strip()
        if "clear" in text or "reset" in text:
            self.button_click('AC')
            self.speak_text("Calculator cleared")
            return
        if "equals" in text or "calculate" in text or "result" in text:
            self.button_click('=')
            return
        math_expression = self.speech_to_math(text)
        if math_expression:
            self.current_expression = math_expression
            self.update_display()
            if any(op in math_expression for op in ['+', '-', '*', '/']):
                self.calculate_expression()
                result = self.result_var.get()
                if result != "Error":
                    self.speak_text(f"The result is {result}")
        else:
            self.update_status("Could not parse voice command", "orange")
            self.speak_text("I didn't understand that calculation")

    def speech_to_math(self, text):
        number_words = {
            'zero': '0', 'one': '1', 'two': '2', 'three': '3', 'four': '4',
            'five': '5', 'six': '6', 'seven': '7', 'eight': '8', 'nine': '9',
            'ten': '10', 'eleven': '11', 'twelve': '12', 'thirteen': '13',
            'fourteen': '14', 'fifteen': '15', 'sixteen': '16', 'seventeen': '17',
            'eighteen': '18', 'nineteen': '19', 'twenty': '20', 'thirty': '30',
            'forty': '40', 'fifty': '50', 'sixty': '60', 'seventy': '70',
            'eighty': '80', 'ninety': '90', 'hundred': '100'
        }
        operation_words = {
            'plus': '+', 'add': '+', 'added to': '+',
            'minus': '-', 'subtract': '-', 'minus': '-', 'take away': '-',
            'times': '*', 'multiply': '*', 'multiplied by': '*',
            'divide': '/', 'divided by': '/', 'over': '/',
            'power': '**', 'to the power of': '**', 'squared': '**2',
            'cubed': '**3'
        }
        for word, digit in number_words.items():
            text = text.replace(word, digit)
        for word, symbol in operation_words.items():
            text = text.replace(word, symbol)
        text = re.sub(r's+', '', text)
        text = text.replace('point', '.')
        pattern = r'([0-9.]+)s*([+s*/-]+)s*([0-9.]+)'
        match = re.search(pattern, text)
        if match:
            return f"{match.group(1)}{match.group(2)}{match.group(3)}"
        result = re.findall(r'[0-9.+/*-]+', text)
        if result:
            return ''.join(result)
        return None

    def speak_result(self):
        result = self.result_var.get()
        if result and result != "0" and result != "Error":
            self.speak_text(f"The result is {result}")
        else:
            self.speak_text("No result to speak")

    def speak_text(self, text):
        def tts():
            try:
                self.tts_engine.say(text)
                self.tts_engine.runAndWait()
            except Exception as e:
                print(f"TTS Error: {e}")
        threading.Thread(target=tts, daemon=True).start()

if __name__ == "__main__":
    root = tk.Tk()
    app = VoiceCalculator(root)
    root.mainloop()