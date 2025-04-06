# Password-ai
Making a ai that cracks wifi passwords 
import random
import wave
import struct
import numpy as np
import pyaudio
import time
import os
from collections import defaultdict
from difflib import SequenceMatcher
import sounddevice as sd
import soundfile as sf
import speech_recognition as sr
from gtts import gTTS
import pygame
import hashlib

class AIPasswordEnhancer:
    def __init__(self):
        # Initialize components
        self.recognizer = sr.Recognizer()
        self.microphone = sr.Microphone()
        self.password_freq = defaultdict(int)
        self.pattern_db = defaultdict(list)
        self.load_rockyou()
        self.setup_audio()
        
        # AI learning parameters
        self.learning_rate = 0.1
        self.creativity = 0.7
        self.mutation_rate = 0.3
        
        # Audio parameters
        self.sample_rate = 44100
        self.duration = 1.0
        self.freq_low = 200
        self.freq_high = 2000
        
        # Conversation state
        self.conversation_mode = "english"  # or "nonhuman"
        self.last_input = ""
        
        print("AI Password Dictionary Enhancer initialized. Ready to communicate.")
    
    def load_rockyou(self):
        """Load the rockyou.txt password dictionary"""
        try:
            with open("rockyou.txt", "r", encoding='latin-1') as f:
                self.base_passwords = [line.strip() for line in f.readlines()]
                for pw in self.base_passwords:
                    self.password_freq[pw] += 1
                    self.analyze_pattern(pw)
            print(f"Loaded {len(self.base_passwords)} base passwords.")
        except FileNotFoundError:
            print("rockyou.txt not found in current directory.")
            self.base_passwords = []
    
    def analyze_pattern(self, password):
        """Analyze patterns in passwords for generation"""
        if len(password) < 3:
            return
            
        # Analyze character transitions
        for i in range(len(password)-1):
            current_char = password[i]
            next_char = password[i+1]
            self.pattern_db[current_char].append(next_char)
    
    def generate_password(self):
        """Generate a new password based on learned patterns"""
        if not self.pattern_db:
            return random.choice(self.base_passwords) if self.base_passwords else "password123"
        
        # Start with a random character that has patterns
        current_char = random.choice(list(self.pattern_db.keys()))
        password = [current_char]
        
        # Build password using Markov chain
        for _ in range(random.randint(6, 15)):
            if current_char in self.pattern_db and random.random() > self.creativity:
                next_char = random.choice(self.pattern_db[current_char])
                password.append(next_char)
                current_char = next_char
            else:
                # Add some randomness
                new_char = chr(random.randint(33, 126))
                password.append(new_char)
                current_char = new_char
        
        # Sometimes mutate the password
        if random.random() < self.mutation_rate:
            password = self.mutate_password(''.join(password))
        
        return ''.join(password)
    
    def mutate_password(self, password):
        """Apply mutations to a password"""
        mutations = [
            lambda s: s + str(random.randint(0, 999)),
            lambda s: s[::-1],
            lambda s: s.upper(),
            lambda s: s.lower(),
            lambda s: s + random.choice(['!', '@', '#', '$', '%']),
            lambda s: s.replace('e', '3').replace('a', '@').replace('o', '0'),
            lambda s: hashlib.md5(s.encode()).hexdigest()[:8]
        ]
        return random.choice(mutations)(password)
    
    def setup_audio(self):
        """Initialize audio components"""
        self.py_audio = pyaudio.PyAudio()
        pygame.mixer.init()
    
    def generate_tone(self, frequency, duration):
        """Generate a sine wave tone"""
        samples = (np.sin(2 * np.pi * np.arange(self.sample_rate * duration) * frequency / self.sample_rate)).astype(np.float32)
        return samples
    
    def play_nonhuman_sound(self, text):
        """Convert text to non-human readable sound"""
        # Create a unique hash from the text
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        
        # Convert hash to sound frequencies
        sounds = []
        for i in range(0, len(text_hash), 4):
            hex_chunk = text_hash[i:i+4]
            freq = int(hex_chunk, 16) % 4000 + 100  # Between 100-4100 Hz
            duration = 0.1 + (int(hex_chunk[0], 16)/16) * 0.2  # 0.1-0.3 sec
            
            # Alternate between high and low frequency
            if i % 8 < 4:
                freq = max(100, freq // 2)  # Low frequency
            else:
                freq = min(4000, freq * 1.5)  # High frequency
                
            tone = self.generate_tone(freq, duration)
            sounds.append(tone)
        
        # Combine all tones
        combined = np.concatenate(sounds)
        sd.play(combined, self.sample_rate)
        sd.wait()
    
    def text_to_speech(self, text):
        """Convert text to speech using gTTS"""
        tts = gTTS(text=text, lang='en')
        tts.save("response.mp3")
        pygame.mixer.music.load("response.mp3")
        pygame.mixer.music.play()
        while pygame.mixer.music.get_busy():
            time.sleep(0.1)
    
    def listen_to_speech(self):
        """Listen to user speech and convert to text"""
        with self.microphone as source:
            print("Listening...")
            self.recognizer.adjust_for_ambient_noise(source)
            audio = self.recognizer.listen(source)
        
        try:
            text = self.recognizer.recognize_google(audio)
            print(f"You said: {text}")
            return text.lower()
        except sr.UnknownValueError:
            print("Could not understand audio")
            return ""
        except sr.RequestError:
            print("API unavailable")
            return ""
    
    def process_command(self, command):
        """Process user commands"""
        self.last_input = command
        
        # Toggle conversation mode
        if "switch to non human" in command:
            self.conversation_mode = "nonhuman"
            return "Switching to non-human communication mode"
        elif "switch to english" in command:
            self.conversation_mode = "english"
            return "Switching to English communication mode"
        
        # Learning commands
        elif "this is a good password" in command:
            # Extract the password (simplified)
            words = command.split()
            for word in words:
                if word.isalnum() and len(word) > 4 and not word.isnumeric():
                    self.password_freq[word] += 1
                    self.analyze_pattern(word)
                    return f"Learned password: {word}. Frequency increased."
            return "Could not identify password to learn."
        
        elif "generate password" in command:
            new_pw = self.generate_password()
            return f"Generated password: {new_pw}"
        
        elif "analyze similarity" in command:
            if self.last_input:
                sample = ' '.join(command.split()[2:])
                similarity = SequenceMatcher(None, self.last_input, sample).ratio()
                return f"Similarity between inputs: {similarity:.2f}"
            return "No previous input to compare with"
        
        elif "exit" in command or "quit" in command:
            return "exit"
        
        return f"I received your command: {command}. How can I improve the password dictionary?"
    
    def communicate(self):
        """Main communication loop"""
        while True:
            if self.conversation_mode == "english":
                # English mode - voice interaction
                command = self.listen_to_speech()
                if not command:
                    continue
                
                response = self.process_command(command)
                if response == "exit":
                    print("Exiting...")
                    break
                
                self.text_to_speech(response)
                print(f"AI: {response}")
            
            else:
                # Non-human mode - sound interaction
                print("Waiting for audio input...")
                time.sleep(2)  # Simulate listening
                
                # Generate a response (simulated)
                response = self.generate_password() + str(random.randint(0, 9999))
                print(f"AI generated password: {response}")
                self.play_nonhuman_sound(response)
                
                # Occasionally switch back to English
                if random.random() < 0.2:
                    self.conversation_mode = "english"
                    self.text_to_speech("Switching back to English for clarification")

if __name__ == "__main__":
    ai = AIPasswordEnhancer()
    ai.communicate()
