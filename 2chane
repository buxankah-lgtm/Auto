import os
import time
import logging
import pytesseract
import cv2
import numpy as np
import psycopg2
import config as cfg
from nomeroff_net import pipeline
from nomeroff_net.tools import unzip
from datetime import datetime, timedelta
from tkinter import Tk, Label, Text, END, Entry, Button, messagebox
from tkinter import ttk
from PIL import Image, ImageTk
from collections import defaultdict

# Логирование
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('parking_system.log', encoding='utf-8'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Путь к pytesseract
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

# Вспомогательные функции
def extract_first_text(obj):
    if obj is None:
        return ""
    if isinstance(obj, str):
        return obj
    if isinstance(obj, (list, tuple, np.ndarray)):
        for item in obj:
            res = extract_first_text(item)
            if res:
                return res
    return ""

def extract_numbers(obj):
    nums = []
    if obj is None:
        return nums
    if isinstance(obj, (int, float)):
        nums.append(float(obj))
    elif isinstance(obj, (list, tuple, np.ndarray)):
        for item in obj:
            nums.extend(extract_numbers(item))
    else:
        try:
            nums.append(float(obj))
        except Exception:
            pass
    return nums

# Настройка базы данных
def init_db():
    conn = psycopg2.connect(host=cfg.host, dbname=cfg.db_name, user=cfg.user, password=cfg.password, port=cfg.port)
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS parking_records (
            id SERIAL PRIMARY KEY,
            costhour INTEGER NOT NULL,
            numbercar VARCHAR(20) NOT NULL,
            "on" TIMESTAMP NOT NULL,
            "off" TIMESTAMP,
            sumcost INTEGER
        )
    """)
    conn.commit()
    return conn, cursor

# Главный класс GUI
class ParkingSystemGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Auto Scanner")
        self.root.geometry("800x600")
        self.root.configure(bg="#1E2A44")  # Тёмно-синий фон

        # Настройка стилей для современного вида
        style = ttk.Style()
        style.theme_use('clam')
        style.configure("TFrame", background="#1E2A44")
        style.configure("TLabel", background="#1E2A44", foreground="#FFFFFF")
        style.configure("TButton", background="#2E4057", foreground="#FFFFFF", font=("Helvetica", 10, "bold"))
        style.map("TButton", background=[("active", "#3A5FCD")])
        style.configure("TEntry", fieldbackground="#2E4057", foreground="#FFFFFF")

        # Заголовок
        self.title_label = ttk.Label(root, text="AUTO SCANNER", font=("Helvetica", 16, "bold"), foreground="#00B7EB")
        self.title_label.pack(pady=10)

        # Главный фрейм
        main_frame = ttk.Frame(root)
        main_frame.pack(pady=20, padx=20, fill="both", expand=True)

        # Секция распознавания
        recog_frame = ttk.Frame(main_frame)
        recog_frame.grid(row=0, column=0, padx=10, pady=10, sticky="nsew")

        ttk.Label(recog_frame, text="РАСПОЗНАНИЕ", font=("Helvetica", 12, "bold")).grid(row=0, column=0, pady=5)
        self.video_label = ttk.Label(recog_frame)
        self.video_label.grid(row=1, column=0, pady=5)
        ttk.Button(recog_frame, text="Входы", command=lambda: logger.info("Entrance button clicked")).grid(row=2, column=0, pady=5)
        ttk.Button(recog_frame, text="Выходы", command=lambda: logger.info("Exit button clicked")).grid(row=3, column=0, pady=5)

        # Секция цены
        price_frame = ttk.Frame(main_frame)
        price_frame.grid(row=0, column=1, padx=10, pady=10, sticky="nsew")

        ttk.Label(price_frame, text="ЧЕНА:", font=("Helvetica", 12, "bold")).grid(row=0, column=0, pady=5)
        self.cost_label = ttk.Label(price_frame, text="Введите цену за час (в рублях):", foreground="#00B7EB")
        self.cost_label.grid(row=1, column=0, pady=5)
        self.cost_entry = ttk.Entry(price_frame)
        self.cost_entry.grid(row=2, column=0, pady=5)
        self.cost_button = ttk.Button(price_frame, text="Установить цену", command=self.set_costhour)
        self.cost_button.grid(row=3, column=0, pady=5)

        # Блоки Вход и Выход
        status_frame = ttk.Frame(root)
        status_frame.pack(pady=10, padx=20, fill="x")

        ttk.Label(status_frame, text="Вход:", font=("Helvetica", 12, "bold")).grid(row=0, column=0, padx=10, pady=5)
        self.entrance_plate_label = ttk.Label(status_frame, text="Нет данных", foreground="#00B7EB")
        self.entrance_plate_label.grid(row=1, column=0, padx=10, pady=5)

        ttk.Label(status_frame, text="Выход:", font=("Helvetica", 12, "bold")).grid(row=0, column=1, padx=10, pady=5)
        self.exit_plate_label = ttk.Label(status_frame, text="Нет данных", foreground="#00B7EB")
        self.exit_plate_label.grid(row=1, column=1, padx=10, pady=5)

        # Статус и лог
        self.status_label = ttk.Label(root, text="Ожидание распознавания...", foreground="#00B7EB")
        self.status_label.pack(pady=10)
        self.log_text = Text(root, height=5, width=80, bg="#2E4057", fg="#FFFFFF", font=("Arial", 10))
        self.log_text.pack(pady=10)

        # База данных
        try:
            self.conn, self.cursor = init_db()
            logger.info("Подключение к базе данных успешно!")
        except Exception as e:
            logger.error(f"Ошибка подключения к базе данных: {e}")
            messagebox.showerror("Ошибка", f"Не удалось подключиться к базе данных: {e}")
            self.root.quit()
            exit()

        # Цена за час
        self.costhour = None

        # Инициализация Nomeroff Net
        try:
            self.number_plate_detection = pipeline("number_plate_detection_and_reading", image_loader="opencv")
            logger.info("Nomeroff Net инициализирован!")
        except Exception as e:
            logger.error(f"Ошибка инициализации: {e}")
            try:
                self.number_plate_detection = pipeline("number_plate_short_detection_and_reading", image_loader="opencv")
                logger.info("Режим коротких номеров инициализирован!")
            except Exception as e2:
                logger.error(f"Ошибка режима коротких номеров: {e2}")
                self.root.quit()
                exit()

        # Камеры
        self.cap_entrance = cv2.VideoCapture(cfg.CAMERA_ID_ENTRANCE)
        if not self.cap_entrance.isOpened():
            logger.error(f"Ошибка: не удалось открыть камеру на входе (ID: {cfg.CAMERA_ID_ENTRANCE})")
            messagebox.showerror("Ошибка", f"Не удалось открыть камеру на входе (ID: {cfg.CAMERA_ID_ENTRANCE})")
            self.root.quit()
            exit()

        self.cap_exit = cv2.VideoCapture(cfg.CAMERA_ID_EXIT)
        if not self.cap_exit.isOpened():
            logger.warning(f"Предупреждение: не удалось открыть камеру на выходе (ID: {cfg.CAMERA_ID_EXIT}). Обнаружение выхода отключено.")
            self.cap_exit = None

        self.cap_entrance.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
        self.cap_entrance.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
        self.cap_entrance.set(cv2.CAP_PROP_FPS, 30)
        self.fps = self.cap_entrance.get(cv2.CAP_PROP_FPS) or 30
        self.width = int(self.cap_entrance.get(cv2.CAP_PROP_FRAME_WIDTH))
        self.height = int(self.cap_entrance.get(cv2.CAP_PROP_FRAME_HEIGHT))

        # Настройка выходного видео
        self.output_dir = os.path.join("car_numbers", "webcam")
        os.makedirs(self.output_dir, exist_ok=True)
        os.makedirs(cfg.DEBUG_DIR, exist_ok=True)
        output_video_path = os.path.join("car_numbers", "webcam_detect.mp4")
        fourcc = cv2.VideoWriter_fourcc(*'mp4v')
        self.out_video = cv2.VideoWriter(output_video_path, fourcc, self.fps, (self.width, self.height))

        # Инициализация файла лога
        with open("car_numbers.txt", "w", encoding="utf-8") as f:
            f.write(f"{self.height} {self.width} webcam {self.fps}\n")
            f.write("Распознанные номера:\n")

        # Отслеживание номеров
        self.plate_frames_entrance = defaultdict(list)
        self.plate_frames_exit = defaultdict(list)
        self.plate_count = 0
        self.temp_image_path = "temp_frame.jpg"
        self.last_processed_time_entrance = time.time()
        self.last_processed_time_exit = time.time()
        self.last_number = None
        self.last_bbox = None
        self.last_confidence = 0.0

        # Защита от повторной записи одинаковых номеров
        self.last_entrance_plate = None
        self.last_entrance_time = None
        self.last_exit_plate = None
        self.last_exit_time = None
        self.min_time_between_same_plate = timedelta(minutes=5)

        logger.info(f"Запуск системы с камерой на входе ID {cfg.CAMERA_ID_ENTRANCE}...")
        self.update_video()

    def set_costhour(self):
        try:
            self.costhour = int(self.cost_entry.get())
            if self.costhour <= 0:
                raise ValueError("Цена должна быть положительной")
            self.cost_label.config(text=f"Цена за час: {self.costhour} руб.")
            self.cost_entry.config(state='disabled')
            self.cost_button.config(state='disabled')
            logger.info(f"Цена за час установлена на {self.costhour} руб.")
        except ValueError as e:
            messagebox.showerror("Ошибка", f"Введите корректное число: {e}")
            logger.error(f"Неверный ввод costhour: {e}")

    def add_log(self, text):
        self.log_text.insert(END, text + "\n")
        lines = int(self.log_text.index('end-1c').split('.')[0])
        if lines > 5:
            to_delete = lines - 5
            self.log_text.delete('1.0', f'{to_delete + 1}.0')

    def is_duplicate_plate(self, plate, is_entrance=True):
        current_time = datetime.now()
        if is_entrance:
            if (self.last_entrance_plate == plate and
                    self.last_entrance_time and
                    (current_time - self.last_entrance_time) < self.min_time_between_same_plate):
                return True
            self.last_entrance_plate = plate
            self.last_entrance_time = current_time
        else:
            if (self.last_exit_plate == plate and
                    self.last_exit_time and
                    (current_time - self.last_exit_time) < self.min_time_between_same_plate):
                return True
            self.last_exit_plate = plate
            self.last_exit_time = current_time
        return False

    def update_video(self):
        ret, frame = self.cap_entrance.read()
        if not ret or frame is None or not isinstance(frame, np.ndarray):
            logger.warning("Не удалось захватить кадр с камеры входа")
            self.status_label.config(text="Ошибка: не удалось получить кадр с камеры входа")
            self.root.after(100, self.update_video)
            return

        frame_annotated = frame.copy()
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')

        if self.last_number and self.last_number != "UNKNOWN" and self.last_confidence > cfg.CONFIDENCE_THRESHOLD and self.last_bbox:
            cv2.rectangle(frame_annotated, (int(self.last_bbox[0]), int(self.last_bbox[1])),
                          (int(self.last_bbox[2]), int(self.last_bbox[3])), (255, 0, 0), 2)
            cv2.putText(frame_annotated, self.last_number, (int(self.last_bbox[0]), int(self.last_bbox[1]) - 10),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.9, (255, 0, 0), 2)
        if self.plate_frames_entrance:
            final_plate = max(self.plate_frames_entrance.keys(), key=lambda k: len(self.plate_frames_entrance[k]),
                              default=None)
            if final_plate:
                cv2.putText(frame_annotated, f"Final: {final_plate}", (10, 50),
                            cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)

        frame_rgb = cv2.cvtColor(frame_annotated, cv2.COLOR_BGR2RGB)
        img = Image.fromarray(frame_rgb)
        img = img.resize((640, 480), Image.Resampling.LANCZOS)
        imgtk = ImageTk.PhotoImage(image=img)
        self.video_label.imgtk = imgtk
        self.video_label.configure(image=imgtk)

        current_time = time.time()
        if current_time - self.last_processed_time_entrance >= cfg.PROCESS_INTERVAL:
            self.last_processed_time_entrance = current_time
            self.process_frame(frame, is_entrance=True)

        if self.cap_exit:
            ret_exit, frame_exit = self.cap_exit.read()
            if ret_exit and frame_exit is not None and isinstance(frame_exit, np.ndarray):
                if current_time - self.last_processed_time_exit >= cfg.PROCESS_INTERVAL:
                    self.last_processed_time_exit = current_time
                    self.process_frame(frame_exit, is_entrance=False)

        self.out_video.write(frame_annotated)
        self.root.after(33, self.update_video)

    def process_frame(self, frame, is_entrance=True):
        try:
            cv2.imwrite(self.temp_image_path, frame)
            result = self.number_plate_detection([self.temp_image_path])
            (images, images_bboxs, images_points, images_zones,
             region_ids, region_names, count_lines, confidences, texts) = unzip(result)

            logger.debug(
                f"{'Вход' if is_entrance else 'Выход'} - texts: {texts}, confidences: {confidences}, images_bboxs: {images_bboxs}")

            timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            timestamp_dt = datetime.now()

            candidate = extract_first_text(texts).strip()
            number = candidate if candidate else "UNKNOWN"
            conf_values = extract_numbers(confidences)
            confidence = float(max(conf_values)) if conf_values else 0.0

            accepted = False
            if number != "UNKNOWN" and confidence > cfg.CONFIDENCE_THRESHOLD and len(number) >= 4:
                if self.is_duplicate_plate(number, is_entrance):
                    status = f"⏭️ {'Вход' if is_entrance else 'Выход'}: {number} ({timestamp}) - Пропущен (дубликат)"
                    logger.info(f"Пропущен дубликат номера: {number}")
                else:
                    status = f"✅ {'Вход' if is_entrance else 'Выход'}: {number} ({timestamp}) - Вход разрешён (уверенность: {confidence:.2f})"
                    accepted = True
                    bbox = None
                    if images_bboxs and len(images_bboxs) > 0 and isinstance(images_bboxs[0], (list, tuple)) and len(images_bboxs[0]) >= 4:
                        bbox = [float(x) for x in images_bboxs[0][:4]]
                    plate_frames = self.plate_frames_entrance if is_entrance else self.plate_frames_exit
                    plate_frames[number].append((frame.copy(), bbox, timestamp_dt, confidence))

                    if is_entrance:
                        self.last_number = number
                        self.last_bbox = bbox
                        self.last_confidence = confidence
                        if bbox:
                            zone = frame[int(bbox[1]):int(bbox[3]), int(bbox[0]):int(bbox[2])]
                            if zone.size > 0:
                                cv2.imwrite(os.path.join(self.output_dir, f"{self.plate_count + 1}_zone.jpg"), zone)
                    self.entrance_plate_label.config(text=number) if is_entrance else self.exit_plate_label.config(text=number)
            else:
                status = f"❌ {'Вход' if is_entrance else 'Выход'}: Номер не распознан ({timestamp})" if number == "UNKNOWN" else \
                    f"⚠️ {'Вход' if is_entrance else 'Выход'}: {number} ({timestamp}) - НЕ принят (уверенность: {confidence:.2f})"
                if is_entrance:
                    self.last_number = None
                    self.last_bbox = None
                    self.last_confidence = 0.0

            plate_frames = self.plate_frames_entrance if is_entrance else self.plate_frames_exit
            with open("car_numbers.txt", "a", encoding="utf-8") as f:
                f.write(status + "\n")
                if plate_frames and accepted:
                    final_plate = max(plate_frames.keys(), key=lambda k: len(plate_frames[k]), default=None)
                    if final_plate and len(plate_frames[final_plate]) >= cfg.MIN_DETECTIONS_FOR_FINAL:
                        self.plate_count += 1
                        f.write(f"{self.plate_count} {final_plate} ({'Вход' if is_entrance else 'Выход'})\n")
                        if is_entrance:
                            frames = plate_frames[final_plate]
                            indices = [0, len(frames) // 2, len(frames) - 1] if len(frames) >= 3 else range(len(frames))
                            for i, idx in enumerate(indices[:3]):
                                cv2.imwrite(os.path.join(self.output_dir, f"{self.plate_count}_{i + 1}.jpg"),
                                            frames[idx][0])
                            if self.costhour is not None:
                                logger.debug(f"Попытка вставки: costhour={self.costhour}, numbercar={final_plate}, on={frames[0][2]}")
                                self.cursor.execute(
                                    'INSERT INTO parking_records (costhour, numbercar, "on") VALUES (%s, %s, %s) RETURNING id',
                                    (self.costhour, final_plate, frames[0][2])
                                )
                                record_id = self.cursor.fetchone()[0]
                                self.conn.commit()
                                logger.info(f"Запись входа: {final_plate}, ID: {record_id}")
                        else:
                            self.cursor.execute(
                                'SELECT id, costhour, "on" FROM parking_records WHERE numbercar = %s AND "off" IS NULL ORDER BY "on" DESC LIMIT 1',
                                (final_plate,)
                            )
                            record = self.cursor.fetchone()
                            if record:
                                record_id, costhour, time_entrance = record
                                time_exit = timestamp_dt
                                hours = max(1, int((time_exit - time_entrance).total_seconds() / 3600))
                                payment = costhour * hours
                                self.cursor.execute(
                                    'UPDATE parking_records SET "off" = %s, sumcost = %s WHERE id = %s',
                                    (time_exit, payment, record_id)
                                )
                                self.conn.commit()
                                status = f"💸 Выход: {final_plate} ({timestamp}), Время: {hours} ч, К оплате: {payment} руб."
                                self.status_label.config(text=status)
                                self.add_log(status)
                                logger.info(status)
                        del plate_frames[final_plate]

            self.status_label.config(text=status)
            self.add_log(status)

            if cfg.DEBUG_SAVE_FRAMES and not accepted:
                safe_ts = timestamp.replace(":", "-").replace(" ", "_")
                debug_path = os.path.join(cfg.DEBUG_DIR,
                                          f"frame_{safe_ts}_{number}_{'entrance' if is_entrance else 'exit'}.jpg")
                cv2.imwrite(debug_path, frame)

        except Exception as e:
            logger.error(f"Ошибка обработки кадра ({'вход' if is_entrance else 'выход'}): {e}")
            self.status_label.config(text=f"Ошибка: {e}")
        finally:
            if os.path.exists(self.temp_image_path):
                os.remove(self.temp_image_path)

    def cleanup(self):
        if self.cap_entrance:
            self.cap_entrance.release()
        if self.cap_exit:
            self.cap_exit.release()
        if self.out_video:
            self.out_video.release()
        if self.conn:
            self.conn.close()
        logger.info("Ресурсы очищены")
        self.root.quit()

    def run(self):
        try:
            self.root.mainloop()
        except KeyboardInterrupt:
            logger.info("Программа остановлена пользователем")
        finally:
            self.cleanup()

def main():
    root = Tk()
    app = ParkingSystemGUI(root)
    app.run()

if __name__ == "__main__":
    main()
