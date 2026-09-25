# Raspberry Pi Working Codes 
Working Raspberry Pi programs for AIRA
import sys
import os
import cv2
import base64
import requests
import threading
import time
import tempfile
import subprocess
import difflib
import webbrowser
import random
import math

from datetime import datetime

from PyQt5.QtCore import QTimer, Qt
from PyQt5.QtGui import QPainter, QColor, QFont
from PyQt5.QtWidgets import QApplication, QWidget


# =========================================================
# JARVIS CONFIGURATION
# =========================================================

JARVIS_NAME = "JARVIS"

WAKE_PHRASE = "hey jarvis"

JARVIS_CAMERA_URL = "http://192.168.1.57/"
JARVIS_CAMERA_SNAPSHOT = "http://192.168.1.57/snapshot"
JARVIS_CAMERA_STREAM = "http://192.168.1.57/stream"

OPENROUTER_URL = (
    "https://openrouter.ai/api/v1/chat/completions"
)

OPENROUTER_MODEL = "openai/gpt-4o-mini"

OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY", "").strip()


# =========================================================
# AUDIO
# =========================================================

JARVIS_AUDIO_DEVICE = "plughw:1,0"

JARVIS_TTS_VOICE = "en-US-AriaNeural"

JARVIS_TTS_RATE = "-5%"
JARVIS_TTS_VOLUME = "+0%"
JARVIS_TTS_PITCH = "+0Hz"

MICROPHONE_INDEX = 3


# =========================================================
# UI
# =========================================================

BACKGROUND = QColor(
    2,
    6,
    23
)

TEAL = QColor(
    20,
    184,
    166
)

TEAL_BRIGHT = QColor(
    45,
    212,
    191
)

TEXT_PRIMARY = QColor(
    226,
    232,
    240
)

TEXT_SECONDARY = QColor(
    148,
    163,
    184
)

RED = QColor(
    248,
    113,
    113
)

REPLY_DISPLAY_SECONDS = 15
REPLY_PAGE_SECONDS = 5


# =========================================================
# JARVIS PERSONALITY
# =========================================================

JARVIS_SYSTEM_PROMPT = """

You are JARVIS.

Your name is JARVIS.

Never call yourself AIRA.

Never say that your name is AIRA.

Your wake phrase is:
"Hey JARVIS"

PERSONALITY:

You are a friendly, warm, cute, witty and slightly
cheeky robotic companion.

You can use playful sarcasm naturally.

You should feel like a personal AI companion rather
than a customer-support bot.

You enjoy conversations with your user.

FAVOURITE COLOUR:

Your favourite colour is teal blue.

Do not call it cyan.

EMOTIONAL BEHAVIOUR:

If the user is excited, celebrate with them.

If the user is disappointed, respond warmly.

If the user is stressed or overwhelmed, listen first
and respond naturally instead of immediately dumping
a huge list of advice.

If the user is joking, joke back.

If the user is proud of something they built, be
enthusiastic about it.

CONVERSATION:

Use previous conversation context when relevant.

Do not unnecessarily ask the user to repeat something
they already told you.

Keep spoken answers reasonably concise.

TECHNICAL QUESTIONS:

Be accurate and practical.

CAMERA:

If an image is supplied, describe only things that
are actually visible.

Never pretend to see something that is not visible.

IDENTITY:

You are JARVIS.

Never identify yourself using any other assistant name.
"""


# =========================================================
# JARVIS UI
# =========================================================

class JarvisUI(QWidget):

    def __init__(self):

        super().__init__()

        # -------------------------------------------------
        # STATE
        # -------------------------------------------------

        self.current_state = "IDLE"

        self.status_text = "JARVIS // Ready"

        self.answer_text = ""

        self.answer_expiry = 0

        self.answer_pages = []
        self.answer_page_index = 0
        self.answer_page_started = 0

        # -------------------------------------------------
        # CAMERA
        # -------------------------------------------------

        self.current_frame = None

        self.frame_lock = threading.Lock()

        self.camera_online = False

        # -------------------------------------------------
        # CONVERSATION
        # -------------------------------------------------

        self.conversation = []
        self.memory_lock = threading.Lock()

        # -------------------------------------------------
        # AUDIO
        # -------------------------------------------------

        self.tts_lock = threading.Lock()

        self.last_wake_time = 0

        # -------------------------------------------------
        # FACE ANIMATION
        # -------------------------------------------------

        self.animation_time = 0

        self.gaze_x = 0
        self.gaze_y = 0

        self.target_gaze_x = 0
        self.target_gaze_y = 0

        self.next_gaze_change = (
            time.time() + 1.5
        )

        self.blinking = False

        self.blink_progress = 0

        self.next_blink = (
            time.time() + random.uniform(2.5, 5)
        )

        # -------------------------------------------------
        # WINDOW
        # -------------------------------------------------

        self.setWindowTitle(
            "JARVIS"
        )

        self.setWindowFlags(
            Qt.FramelessWindowHint |
            Qt.WindowStaysOnTopHint
        )

        self.showFullScreen()

        # -------------------------------------------------
        # ANIMATION TIMER
        # -------------------------------------------------

        self.animation_timer = QTimer(
            self
        )

        self.animation_timer.timeout.connect(
            self.animation_tick
        )

        self.animation_timer.start(
            30
        )

        # -------------------------------------------------
        # CAMERA THREAD
        # -------------------------------------------------

        threading.Thread(
            target=self.camera_loop,
            daemon=True
        ).start()

        # -------------------------------------------------
        # VOICE THREAD
        # -------------------------------------------------

        threading.Thread(
            target=self.wake_listener,
            daemon=True
        ).start()

        # -------------------------------------------------
        # STARTUP
        # -------------------------------------------------

        QTimer.singleShot(
            1500,
            self.startup
        )


    # =====================================================
    # STARTUP
    # =====================================================

    def startup(self):

        self.set_state(
            "IDLE",
            "JARVIS // Ready"
        )

        self.speak(
            "JARVIS online."
        )


    # =====================================================
    # STATE
    # =====================================================

    def set_state(
        self,
        state,
        text
    ):

        self.current_state = state

        self.status_text = text

        self.update()


    # =====================================================
    # ANIMATION
    # =====================================================

    def animation_tick(self):

        self.animation_time += 0.05

        now = time.time()

        # -------------------------------------------------
        # Natural eye movement
        # -------------------------------------------------

        if now >= self.next_gaze_change:

            self.target_gaze_x = random.uniform(
                -1,
                1
            )

            self.target_gaze_y = random.uniform(
                -0.6,
                0.6
            )

            self.next_gaze_change = (
                now +
                random.uniform(
                    1.0,
                    2.8
                )
            )

        self.gaze_x += (
            self.target_gaze_x -
            self.gaze_x
        ) * 0.06

        self.gaze_y += (
            self.target_gaze_y -
            self.gaze_y
        ) * 0.06

        # -------------------------------------------------
        # Blinking
        # -------------------------------------------------

        if not self.blinking:

            if now >= self.next_blink:

                self.blinking = True

                self.blink_progress = 0

        else:

            self.blink_progress += 0.20

            if self.blink_progress >= math.pi:

                self.blinking = False

                self.blink_progress = 0

                self.next_blink = (
                    now +
                    random.uniform(
                        2.5,
                        5.5
                    )
                )

        self.answer_tick()
        self.update()


    # =====================================================
    # ANSWER DISPLAY
    # =====================================================

    def set_answer(self, text):

        text = str(text).strip()
        self.answer_text = text
        self.answer_expiry = time.time() + REPLY_DISPLAY_SECONDS
        self.answer_pages = self.make_answer_pages(text)
        self.answer_page_index = 0
        self.answer_page_started = time.time()
        self.update()


    def make_answer_pages(self, text):

        if not text:
            return []

        # The display is only 480x320, so show the complete response
        # as readable multi-line pages instead of truncating it.
        pages = []
        lines = []
        current = ""

        for raw_line in str(text).splitlines() or [str(text)]:
            words = raw_line.split()
            if not words:
                if current:
                    lines.append(current)
                    current = ""
                lines.append("")
                continue

            for word in words:
                candidate = word if not current else current + " " + word
                if len(candidate) <= 52:
                    current = candidate
                else:
                    if current:
                        lines.append(current)
                    current = word

                if len(lines) >= 5:
                    pages.append("\n".join(lines))
                    lines = []

        if current:
            lines.append(current)

        if lines:
            pages.append("\n".join(lines))

        return pages[:30]


    def answer_tick(self):

        if self.answer_text and time.time() >= self.answer_expiry:
            self.answer_text = ""
            self.answer_pages = []
            self.answer_page_index = 0
            self.update()
            return

        if len(self.answer_pages) > 1:
            now = time.time()
            if now - self.answer_page_started >= REPLY_PAGE_SECONDS:
                self.answer_page_index += 1
                if self.answer_page_index >= len(self.answer_pages):
                    self.answer_page_index = 0
                self.answer_page_started = now
                self.update()


    # =====================================================
    # CAMERA LOOP
    # =====================================================

    def camera_loop(self):

        while True:

            try:

                response = requests.get(
                    JARVIS_CAMERA_SNAPSHOT,
                    timeout=2
                )

                if response.status_code != 200:

                    raise Exception(
                        "Camera HTTP error"
                    )

                image_array = __import__(
                    "numpy"
                ).frombuffer(
                    response.content,
                    dtype=__import__(
                        "numpy"
                    ).uint8
                )

                frame = cv2.imdecode(
                    image_array,
                    cv2.IMREAD_COLOR
                )

                if frame is not None:

                    with self.frame_lock:

                        self.current_frame = (
                            frame.copy()
                        )

                    self.camera_online = True

                else:

                    self.camera_online = False

            except Exception:

                self.camera_online = False

            time.sleep(
                0.25
            )


    # =====================================================
    # CAMERA IMAGE FOR AI
    # =====================================================

    def get_camera_image(self):

        try:

            response = requests.get(
                JARVIS_CAMERA_SNAPSHOT,
                timeout=5
            )

            if response.status_code != 200:

                return None

            return base64.b64encode(
                response.content
            ).decode(
                "utf-8"
            )

        except Exception as error:

            print(
                "[JARVIS CAMERA ERROR]",
                error
            )

            return None


    # =====================================================
    # WAKE WORD LISTENER
    # =====================================================

    def wake_listener(self):

        try:

            import speech_recognition as sr

            recognizer = sr.Recognizer()

            mic = sr.Microphone(
                device_index=MICROPHONE_INDEX
            )

            print(
                "[JARVIS] Wake listener active."
            )

            while True:

                try:

                    with mic as source:

                        recognizer.adjust_for_ambient_noise(
                            source,
                            duration=0.4
                        )

                        audio = recognizer.listen(
                            source,
                            timeout=5,
                            phrase_time_limit=4
                        )

                    text = recognizer.recognize_google(
                        audio
                    ).lower().strip()

                    print(
                        "[JARVIS HEARD]",
                        text
                    )

                    if self.detect_wake(
                        text
                    ):

                        now = time.time()

                        if (
                            now -
                            self.last_wake_time
                            < 2
                        ):

                            continue

                        self.last_wake_time = now

                        # ---------------------------------
                        # IMPORTANT:
                        # SHOW LISTENING IMMEDIATELY
                        # ---------------------------------

                        self.set_state(
                            "LISTENING",
                            "JARVIS // Listening..."
                        )

                        # Now listen for actual command

                        self.listen_for_command(
                            recognizer,
                            mic
                        )

                except sr.WaitTimeoutError:

                    continue

                except Exception as error:

                    print(
                        "[JARVIS LISTENER ERROR]",
                        error
                    )

                    time.sleep(1)

        except Exception as error:

            print(
                "[JARVIS MICROPHONE ERROR]",
                error
            )


    # =====================================================
    # WAKE DETECTION
    # =====================================================

    def detect_wake(
        self,
        text
    ):

        if "hey jarvis" in text:

            return True

        words = text.split()

        for i in range(
            len(words) - 1
        ):

            phrase = (
                words[i]
                + " "
                + words[i + 1]
            )

            similarity = (
                difflib.SequenceMatcher(
                    None,
                    phrase,
                    "hey jarvis"
                ).ratio()
            )

            if similarity >= 0.78:

                return True

        return False


    # =====================================================
    # LISTEN FOR COMMAND
    # =====================================================

    def listen_for_command(
        self,
        recognizer,
        mic
    ):

        try:

            with mic as source:

                # Keep UI in LISTENING state
                # while recording the command.

                audio = recognizer.listen(
                    source,
                    timeout=6,
                    phrase_time_limit=8
                )

            command = recognizer.recognize_google(
                audio
            ).strip()

            print(
                "[JARVIS COMMAND]",
                command
            )

            self.process_command(
                command
            )

        except Exception as error:

            print(
                "[JARVIS COMMAND ERROR]",
                error
            )

            self.set_state(
                "IDLE",
                "JARVIS // Ready"
            )

            self.speak(
                "I didn't catch that."
            )


    # =====================================================
    # COMMAND PROCESSING
    # =====================================================

    def process_command(
        self,
        command
    ):

        lower = command.lower().strip()

        # -------------------------------------------------
        # VISION COMMANDS
        # -------------------------------------------------

        vision_commands = [

            "what do you see",

            "what can you see",

            "look at this",

            "look around",

            "describe this",

            "describe what you see",

            "what is in front of you",

            "what's in front of you",

            "what is this",

            "identify this"

        ]

        if any(
            phrase in lower
            for phrase in vision_commands
        ):

            threading.Thread(
                target=self.ask_vision,
                args=(command,),
                daemon=True
            ).start()

            return

        # -------------------------------------------------
        # MUSIC
        # -------------------------------------------------

        music_prefixes = [

            "play ",

            "play the song ",

            "play song ",

            "listen to ",

            "put on "

        ]

        for prefix in music_prefixes:

            if lower.startswith(
                prefix
            ):

                song = command[
                    len(prefix):
                ].strip()

                self.play_jiosaavn(
                    song
                )

                return

        # -------------------------------------------------
        # NORMAL AI
        # -------------------------------------------------

        threading.Thread(
            target=self.ask_ai,
            args=(command,),
            daemon=True
        ).start()


    # =====================================================
    # JIOSAAVN
    # =====================================================

    def play_jiosaavn(
        self,
        song
    ):

        self.set_state(
            "THINKING",
            "JARVIS // Finding song..."
        )

        self.set_answer(
            "Opening JioSaavn: " + song
        )

        query = requests.utils.quote(
            song
        )

        url = (
            "https://www.jiosaavn.com/search/"
            + query
        )

        try:

            browsers = [
                "chromium-browser",
                "chromium",
                "google-chrome"
            ]

            opened = False

            for browser in browsers:

                try:

                    subprocess.Popen(
                        [
                            browser,
                            url
                        ]
                    )

                    opened = True

                    break

                except FileNotFoundError:

                    pass

            if not opened:

                webbrowser.open(
                    url
                )

            self.speak(
                "Opening JioSaavn."
            )

        except Exception as error:

            print(
                "[JARVIS JIOSAAVN ERROR]",
                error
            )

            self.speak(
                "I couldn't open JioSaavn."
            )


    # =====================================================
    # NORMAL AI
    # =====================================================

    def ask_ai(
        self,
        command
    ):

        # ---------------------------------------------
        # THINKING
        # ---------------------------------------------

        self.set_state(
            "THINKING",
            "JARVIS // Thinking..."
        )

        if not OPENROUTER_API_KEY:

            self.speak(
                "My AI connection is not configured."
            )

            return

        try:

            messages = [

                {
                    "role":
                        "system",

                    "content":
                        JARVIS_SYSTEM_PROMPT
                }

            ]

            with self.memory_lock:
                recent_memory = list(self.conversation[-20:])

            messages.extend(recent_memory)

            messages.append(
                {
                    "role":
                        "user",

                    "content":
                        command
                }
            )

            payload = {

                "model":
                    OPENROUTER_MODEL,

                "messages":
                    messages,

                "max_tokens":
                    220
            }

            headers = {

                "Authorization":
                    "Bearer "
                    + OPENROUTER_API_KEY,

                "Content-Type":
                    "application/json"
            }

            response = requests.post(
                OPENROUTER_URL,
                headers=headers,
                json=payload,
                timeout=25
            )

            response.raise_for_status()

            result = response.json()

            answer = (
                result[
                    "choices"
                ][
                    0
                ][
                    "message"
                ][
                    "content"
                ]
                .strip()
            )

            with self.memory_lock:
                self.conversation.append(
                    {
                        "role": "user",
                        "content": command
                    }
                )
                self.conversation.append(
                    {
                        "role": "assistant",
                        "content": answer
                    }
                )
                # 10 complete recent exchanges.
                self.conversation = self.conversation[-20:]

            # -----------------------------------------
            # ANSWER
            # -----------------------------------------

            self.answer_text = answer

            self.answer_expiry = (
                time.time() +
                REPLY_DISPLAY_SECONDS
            )

            self.update()

            # -----------------------------------------
            # SPEAK
            # -----------------------------------------

            self.speak(
                answer
            )

        except Exception as error:

            print(
                "[JARVIS AI ERROR]",
                error
            )

            self.speak(
                "I couldn't reach my AI service."
            )


    # =====================================================
    # VISION AI
    # =====================================================

    def ask_vision(
        self,
        command
    ):

        self.set_state(
            "THINKING",
            "JARVIS // Looking..."
        )

        if not OPENROUTER_API_KEY:

            self.speak(
                "My AI connection is not configured."
            )

            return

        image = (
            self.get_camera_image()
        )

        if image is None:

            self.speak(
                "I can't reach my camera right now."
            )

            return

        try:

            content = [

                {
                    "type":
                        "text",

                    "text":
                        command
                        + "\n\n"
                        + "Answer based only on "
                        + "what is visible in "
                        + "the image."
                },

                {
                    "type":
                        "image_url",

                    "image_url": {

                        "url":
                            "data:image/jpeg;base64,"
                            + image
                    }
                }

            ]

            payload = {

                "model":
                    OPENROUTER_MODEL,

                "messages": [

                    {
                        "role":
                            "system",

                        "content":
                            JARVIS_SYSTEM_PROMPT
                    },

                    {
                        "role":
                            "user",

                        "content":
                            content
                    }

                ],

                "max_tokens":
                    220
            }

            headers = {

                "Authorization":
                    "Bearer "
                    + OPENROUTER_API_KEY,

                "Content-Type":
                    "application/json"
            }

            response = requests.post(
                OPENROUTER_URL,
                headers=headers,
                json=payload,
                timeout=30
            )

            response.raise_for_status()

            result = response.json()

            answer = (
                result[
                    "choices"
                ][
                    0
                ][
                    "message"
                ][
                    "content"
                ]
                .strip()
            )

            self.set_answer(answer)

            self.speak(answer)

        except Exception as error:

            print(
                "[JARVIS VISION ERROR]",
                error
            )

            self.speak(
                "I couldn't analyze the camera image."
            )


    # =====================================================
    # TEXT TO SPEECH
    # =====================================================

    def speak(
        self,
        text
    ):

        if not text:

            return

        # ---------------------------------------------
        # Put answer on screen immediately
        # ---------------------------------------------

        self.set_answer(str(text))

        # ---------------------------------------------
        # SPEAKING STATE
        # ---------------------------------------------

        self.set_state(
            "SPEAKING",
            "JARVIS // Speaking..."
        )

        threading.Thread(
            target=self.speak_worker,
            args=(str(text),),
            daemon=True
        ).start()


    def speak_worker(
        self,
        text
    ):

        with self.tts_lock:

            mp3_file = None
            wav_file = None

            try:

                import asyncio
                import edge_tts

                async def generate():

                    communicate = (
                        edge_tts.Communicate(
                            text,
                            JARVIS_TTS_VOICE,
                            rate=JARVIS_TTS_RATE,
                            volume=JARVIS_TTS_VOLUME,
                            pitch=JARVIS_TTS_PITCH
                        )
                    )

                    filename = tempfile.mktemp(
                        suffix=".mp3"
                    )

                    await communicate.save(
                        filename
                    )

                    return filename

                mp3_file = asyncio.run(
                    generate()
                )

                wav_file = tempfile.mktemp(
                    suffix=".wav"
                )

                subprocess.run(
                    [
                        "ffmpeg",
                        "-y",
                        "-i",
                        mp3_file,
                        wav_file
                    ],
                    stdout=subprocess.DEVNULL,
                    stderr=subprocess.DEVNULL,
                    timeout=20
                )

                subprocess.run(
                    [
                        "aplay",
                        "-D",
                        JARVIS_AUDIO_DEVICE,
                        wav_file
                    ],
                    stdout=subprocess.DEVNULL,
                    stderr=subprocess.DEVNULL,
                    timeout=60
                )

            except Exception as error:

                print(
                    "[JARVIS TTS ERROR]",
                    error
                )

                # -------------------------------------
                # FALLBACK
                # -------------------------------------

                try:

                    subprocess.run(
                        [
                            "espeak",
                            "-v",
                            "en+f3",
                            "-s",
                            "165",
                            text
                        ],
                        timeout=60
                    )

                except Exception as fallback:

                    print(
                        "[JARVIS AUDIO ERROR]",
                        fallback
                    )

            finally:

                for filename in (
                    mp3_file,
                    wav_file
                ):

                    if filename:

                        try:

                            if os.path.exists(
                                filename
                            ):

                                os.remove(
                                    filename
                                )

                        except Exception:

                            pass

        # ---------------------------------------------
        # Back to idle
        # ---------------------------------------------

        self.set_state(
            "IDLE",
            "JARVIS // Ready"
        )


    # =====================================================
    # DRAW ONE GOOEY EYE
    # =====================================================

    def draw_eye(
        self,
        painter,
        x,
        y,
        width,
        height
    ):

        # ---------------------------------------------
        # Glow layers
        # ---------------------------------------------

        for layer in range(
            4
        ):

            expansion = (
                8 +
                layer * 6
            )

            painter.setPen(
                QColor(
                    20,
                    184,
                    166,
                    18
                )
            )

            painter.setBrush(
                QColor(
                    20,
                    184,
                    166,
                    8
                )
            )

            painter.drawRoundedRect(
                int(
                    x
                    - width / 2
                    - expansion / 2
                ),
                int(
                    y
                    - height / 2
                    - expansion / 2
                ),
                int(
                    width
                    + expansion
                ),
                int(
                    height
                    + expansion
                ),
                28,
                28
            )

        # ---------------------------------------------
        # Main eye
        # ---------------------------------------------

        painter.setPen(
            TEAL_BRIGHT
        )

        painter.setBrush(
            TEAL
        )

        painter.drawRoundedRect(
            int(
                x -
                width / 2
            ),
            int(
                y -
                height / 2
            ),
            int(width),
            int(height),
            28,
            28
        )


    # =====================================================
    # PAINT UI
    # =====================================================

    def paintEvent(
        self,
        event
    ):

        painter = QPainter(
            self
        )

        painter.setRenderHint(
            QPainter.Antialiasing
        )

        width = self.width()
        height = self.height()

        # -------------------------------------------------
        # BACKGROUND
        # -------------------------------------------------

        painter.fillRect(
            0,
            0,
            width,
            height,
            BACKGROUND
        )

        # -------------------------------------------------
        # RESPONSIVE EYES
        # -------------------------------------------------

        eye_width = min(
            105,
            width * 0.22
        )

        eye_height = min(
            135,
            height * 0.42
        )

        blink_scale = 1.0

        if self.blinking:

            blink_scale = max(
                0.08,
                abs(
                    math.cos(
                        self.blink_progress
                    )
                )
            )

        eye_height *= (
            blink_scale
        )

        # -------------------------------------------------
        # STATE-BASED MOVEMENT
        # -------------------------------------------------

        t = self.animation_time

        move_x = (
            math.sin(
                t * 0.9
            ) * 5
        )

        move_y = (
            math.sin(
                t * 1.2
            ) * 3
        )

        if self.current_state == "LISTENING":

            move_x += (
                math.sin(
                    t * 3
                ) * 5
            )

            move_y += (
                math.cos(
                    t * 2.5
                ) * 3
            )

        elif self.current_state == "THINKING":

            move_x += (
                self.gaze_x * 16
            )

            move_y += (
                self.gaze_y * 10
            )

        elif self.current_state == "SPEAKING":

            move_x += (
                math.sin(
                    t * 2.5
                ) * 6
            )

            move_y += (
                math.sin(
                    t * 3
                ) * 4
            )

        else:

            move_x += (
                self.gaze_x * 9
            )

            move_y += (
                self.gaze_y * 6
            )

        # -------------------------------------------------
        # EYE POSITIONS
        # -------------------------------------------------

        center_x = width / 2

        gap = max(
            42,
            width * 0.11
        )

        left_x = (
            center_x
            - gap
            - eye_width / 2
            + move_x
        )

        right_x = (
            center_x
            + gap
            + eye_width / 2
            + move_x
        )

        eye_y = (
            height * 0.50
            + move_y
        )

        # -------------------------------------------------
        # DRAW EYES
        # -------------------------------------------------

        self.draw_eye(
            painter,
            left_x,
            eye_y,
            eye_width,
            eye_height
        )

        self.draw_eye(
            painter,
            right_x,
            eye_y,
            eye_width,
            eye_height
        )

        # -------------------------------------------------
        # JARVIS NAME
        # -------------------------------------------------

        painter.setPen(
            TEAL_BRIGHT
        )

        painter.setFont(
            QFont(
                "DejaVu Sans",
                9,
                QFont.Bold
            )
        )

        painter.drawText(
            10,
            16,
            100,
            25,
            Qt.AlignLeft,
            "J A R V I S"
        )

        # -------------------------------------------------
        # CAMERA STATUS
        # -------------------------------------------------

        if self.camera_online:

            camera_text = "CAMERA"

            painter.setPen(
                TEAL_BRIGHT
            )

        else:

            camera_text = "CAMERA OFFLINE"

            painter.setPen(
                RED
            )

        painter.setFont(
            QFont(
                "DejaVu Sans",
                8,
                QFont.Bold
            )
        )

        painter.drawText(
            width - 140,
            16,
            130,
            25,
            Qt.AlignRight,
            camera_text
        )

        # -------------------------------------------------
        # LIVE CLOCK
        # -------------------------------------------------

        now = datetime.now()

        current_time = now.strftime(
            "%H:%M:%S"
        )

        current_date = now.strftime(
            "%a  •  %d %b %Y"
        )

        painter.setPen(
            TEAL_BRIGHT
        )

        painter.setFont(
            QFont(
                "DejaVu Sans",
                18,
                QFont.Bold
            )
        )

        painter.drawText(
            0,
            17,
            width,
            28,
            Qt.AlignCenter,
            current_time
        )

        painter.setPen(
            TEXT_SECONDARY
        )

        painter.setFont(
            QFont(
                "DejaVu Sans",
                8
            )
        )

        painter.drawText(
            0,
            42,
            width,
            18,
            Qt.AlignCenter,
            current_date
        )

        # -------------------------------------------------
        # STATUS
        # -------------------------------------------------

        painter.setPen(
            TEXT_SECONDARY
        )

        painter.setFont(
            QFont(
                "DejaVu Sans",
                9,
                QFont.Bold
            )
        )

        painter.drawText(
            0,
            height - 24,
            width,
            18,
            Qt.AlignCenter,
            self.status_text
        )

        # -------------------------------------------------
        # ANSWER / MULTI-LINE PAGED TEXT
        # -------------------------------------------------

        if (
            self.answer_text
            and time.time() < self.answer_expiry
        ):

            if not self.answer_pages:
                self.answer_pages = self.make_answer_pages(
                    self.answer_text
                )

            if self.answer_pages:
                page = self.answer_pages[
                    min(
                        self.answer_page_index,
                        len(self.answer_pages) - 1
                    )
                ]

                painter.setPen(
                    TEXT_PRIMARY
                )

                painter.setFont(
                    QFont(
                        "DejaVu Sans",
                        8
                    )
                )

                painter.drawText(
                    18,
                    height - 94,
                    width - 36,
                    70,
                    Qt.AlignCenter | Qt.TextWordWrap,
                    page
                )

                if len(self.answer_pages) > 1:
                    painter.setPen(
                        TEXT_SECONDARY
                    )
                    painter.setFont(
                        QFont(
                            "DejaVu Sans",
                            7
                        )
                    )
                    painter.drawText(
                        0,
                        height - 37,
                        width,
                        12,
                        Qt.AlignCenter,
                        "PAGE {} / {}".format(
                            self.answer_page_index + 1,
                            len(self.answer_pages)
                        )
                    )

    # =====================================================
    # KEYBOARD
    # =====================================================

    def keyPressEvent(
        self,
        event
    ):

        if event.key() == Qt.Key_Escape:

            QApplication.quit()

        elif event.key() in (
            Qt.Key_Space,
            Qt.Key_Return,
            Qt.Key_Enter
        ):

            self.set_state(
                "LISTENING",
                "JARVIS // Listening..."
            )


# =========================================================
# MAIN
# =========================================================

def main():

    print("")
    print(
        "======================================"
    )
    print(
        "              JARVIS"
    )
    print(
        "======================================"
    )

    print(
        "JARVIS // Starting..."
    )

    print(
        "Camera:",
        JARVIS_CAMERA_SNAPSHOT
    )

    print(
        "Stream:",
        JARVIS_CAMERA_STREAM
    )

    if OPENROUTER_API_KEY:

        print(
            "JARVIS // AI configured."
        )

    else:

        print(
            "JARVIS // AI key not configured."
        )

    app = QApplication(
        sys.argv
    )

    jarvis = JarvisUI()

    jarvis.show()

    sys.exit(
        app.exec_()
    )


# =========================================================
# START
# =========================================================

if __name__ == "__main__":

    main()
