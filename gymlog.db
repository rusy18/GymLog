from abc import ABC, abstractmethod
from datetime import datetime
import sqlite3
import sys

from PySide6.QtWidgets import (
    QApplication, QWidget, QLabel, QPushButton, QVBoxLayout,
    QHBoxLayout, QLineEdit, QTextEdit, QListWidget, QMessageBox,
    QSpinBox, QDoubleSpinBox, QDateEdit
)
from PySide6.QtCore import QDate


DATE_FORMAT = "%d.%m.%Y %H:%M:%S"


class BaseItem(ABC):
    def __init__(self, name):
        self._name = name

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        value = value.strip()
        if not value:
            raise ValueError("Название не должно быть пустым")
        self._name = value

    @abstractmethod
    def description(self):
        pass

    def __str__(self):
        return self.name

    def __repr__(self):
        return f"{self.__class__.__name__}({self.name!r})"

    def __eq__(self, other):
        return isinstance(other, BaseItem) and self.name.lower() == other.name.lower()

    def __lt__(self, other):
        if not isinstance(other, BaseItem):
            return NotImplemented
        return self.name.lower() < other.name.lower()


class Exercise(BaseItem):
    def __init__(self, name, muscle_group):
        super().__init__(name)
        self._muscle_group = muscle_group

    @property
    def muscle_group(self):
        return self._muscle_group

    @muscle_group.setter
    def muscle_group(self, value):
        self._muscle_group = value.strip() or "Не указана"

    def description(self):
        return f"{self.name} — {self.muscle_group}"

    def __len__(self):
        return len(self.name)

    def __contains__(self, text):
        return text.lower() in self.name.lower()

    def __add__(self, other):
        if isinstance(other, Exercise):
            return self.name + " + " + other.name
        return NotImplemented


class Workout(BaseItem):
    def __init__(self, name, sets, reps, weight, date_text, note=""):
        super().__init__(name)
        self._sets = sets
        self._reps = reps
        self._weight = weight
        self._date_text = date_text
        self._note = note

    @property
    def sets(self):
        return self._sets

    @sets.setter
    def sets(self, value):
        if value < 1:
            raise ValueError("Количество подходов должно быть не меньше 1")
        self._sets = value

    @property
    def reps(self):
        return self._reps

    @reps.setter
    def reps(self, value):
        if value < 1:
            raise ValueError("Количество повторений должно быть не меньше 1")
        self._reps = value

    @property
    def weight(self):
        return self._weight

    @weight.setter
    def weight(self, value):
        if value < 0:
            raise ValueError("Вес не может быть отрицательным")
        self._weight = value

    @property
    def date_text(self):
        return self._date_text

    @date_text.setter
    def date_text(self, value):
        self._date_text = value

    @property
    def note(self):
        return self._note

    @note.setter
    def note(self, value):
        self._note = value

    def description(self):
        return f"{self.name}: {self.sets} подх. × {self.reps} повт. × {self.weight:g} кг"

    def __len__(self):
        return self.sets * self.reps

    def __bool__(self):
        return self.sets > 0 and self.reps > 0

    def __iter__(self):
        return iter((self.name, self.sets, self.reps, self.weight, self.date_text, self.note))


class SimpleMap:
    """Небольшое собственное хранилище пар ключ-значение без dict."""
    def __init__(self):
        self._items = []

    def set(self, key, value):
        for index, pair in enumerate(self._items):
            if pair[0] == key:
                self._items[index] = (key, value)
                return
        self._items.append((key, value))

    def get(self, key, default=None):
        for saved_key, value in self._items:
            if saved_key == key:
                return value
        return default


class GymLogWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.login = "UserName"
        self.time_in = datetime.now()
        self.exercises = [
            Exercise("Жим лёжа", "Грудь"),
            Exercise("Приседания", "Ноги"),
            Exercise("Тяга верхнего блока", "Спина"),
            Exercise("Горизонтальная тяга", "Спина"),
            Exercise("Подъём гантелей", "Плечи"),
            Exercise("Сгибание рук", "Бицепс"),
            Exercise("Разгибание рук", "Трицепс"),
            Exercise("Планка", "Пресс"),
            Exercise("Выпады", "Ноги"),
            Exercise("Бабочка", "Грудь"),
        ]
        self.workouts = []
        self.exercise_map = SimpleMap()
        for exercise in self.exercises:
            self.exercise_map.set(exercise.name, exercise.muscle_group)

        self.setWindowTitle("GymLog — учёт тренировок")
        self.resize(720, 620)
        self.build_ui()
        self.create_database()
        self.load_workouts()

    def build_ui(self):
        layout = QVBoxLayout()
        self.setLayout(layout)

        self.title_label = QLabel("Добро пожаловать в GymLog!")
        self.subtitle_label = QLabel("Твой помощник в мире фитнеса и тренировок")
        self.exercise_label = QLabel("Упражнение")
        self.sets_label = QLabel("Подходы")
        self.reps_label = QLabel("Повторения")
        self.weight_label = QLabel("Вес, кг")
        self.date_label = QLabel("Дата")
        self.note_label = QLabel("Заметка")
        self.status_label = QLabel("Записи сохраняются на компьютере.")

        self.exercise_input = QLineEdit()
        self.exercise_input.setPlaceholderText("Например, Жим лёжа")
        self.sets_input = QSpinBox()
        self.sets_input.setRange(1, 30)
        self.sets_input.setValue(3)
        self.reps_input = QSpinBox()
        self.reps_input.setRange(1, 100)
        self.reps_input.setValue(10)
        self.weight_input = QDoubleSpinBox()
        self.weight_input.setRange(0, 500)
        self.weight_input.setDecimals(1)
        self.date_input = QDateEdit()
        self.date_input.setCalendarPopup(True)
        self.date_input.setDate(QDate.currentDate())
        self.note_input = QTextEdit()
        self.note_input.setPlaceholderText("Необязательно")
        self.note_input.setMaximumHeight(60)
        self.workout_list = QListWidget()

        self.add_button = QPushButton("Добавить запись")
        self.delete_button = QPushButton("Удалить выбранную")
        self.clear_button = QPushButton("Очистить поля")
        self.about_button = QPushButton("О программе")
        self.exit_button = QPushButton("Выход")

        # Простое оформление компонентов окна
        self.title_label.setStyleSheet("font-size: 22px;")
        self.subtitle_label.setStyleSheet("font-size: 14px;")
        self.workout_list.setMinimumHeight(140)
        self.add_button.setMinimumHeight(32)
        self.delete_button.setMinimumHeight(32)
        self.clear_button.setMinimumHeight(32)
        self.about_button.setMinimumHeight(32)
        self.exit_button.setMinimumHeight(32)

        layout.addWidget(self.title_label)
        layout.addWidget(self.subtitle_label)
        layout.addWidget(self.exercise_label)
        layout.addWidget(self.exercise_input)

        fields_row = QHBoxLayout()
        fields_row.addWidget(self.sets_label)
        fields_row.addWidget(self.sets_input)
        fields_row.addWidget(self.reps_label)
        fields_row.addWidget(self.reps_input)
        fields_row.addWidget(self.weight_label)
        fields_row.addWidget(self.weight_input)
        layout.addLayout(fields_row)

        layout.addWidget(self.date_label)
        layout.addWidget(self.date_input)
        layout.addWidget(self.note_label)
        layout.addWidget(self.note_input)
        layout.addWidget(self.add_button)
        layout.addWidget(self.workout_list)

        buttons_row = QHBoxLayout()
        buttons_row.addWidget(self.delete_button)
        buttons_row.addWidget(self.clear_button)
        buttons_row.addWidget(self.about_button)
        buttons_row.addWidget(self.exit_button)
        layout.addLayout(buttons_row)
        layout.addWidget(self.status_label)

        self.add_button.clicked.connect(self.add_workout)
        self.delete_button.clicked.connect(self.delete_workout)
        self.clear_button.clicked.connect(self.clear_fields)
        self.about_button.clicked.connect(self.show_about)
        self.exit_button.clicked.connect(self.close)

    def create_database(self):
        connection = sqlite3.connect("gymlog.db")
        cursor = connection.cursor()
        cursor.execute("""CREATE TABLE IF NOT EXISTS workouts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL, sets INTEGER NOT NULL, reps INTEGER NOT NULL,
            weight REAL NOT NULL, date_text TEXT NOT NULL, note TEXT
        )""")
        connection.commit()
        connection.close()

    def add_workout(self):
        name = self.exercise_input.text().strip()
        if not name:
            QMessageBox.warning(self, "Нет упражнения", "Введи название упражнения.")
            return
        date_text = self.date_input.date().toString("dd.MM.yyyy")
        workout = Workout(name, self.sets_input.value(), self.reps_input.value(),
                          self.weight_input.value(), date_text, self.note_input.toPlainText().strip())
        connection = sqlite3.connect("gymlog.db")
        cursor = connection.cursor()
        cursor.execute("INSERT INTO workouts (name, sets, reps, weight, date_text, note) VALUES (?, ?, ?, ?, ?, ?)", tuple(workout))
        connection.commit()
        connection.close()
        self.workouts.append(workout)
        self.refresh_list()
        self.status_label.setText("Запись добавлена.")
        self.clear_fields(keep_status=True)

    def load_workouts(self):
        connection = sqlite3.connect("gymlog.db")
        cursor = connection.cursor()
        cursor.execute("SELECT name, sets, reps, weight, date_text, note FROM workouts ORDER BY id DESC")
        rows = cursor.fetchall()
        connection.close()
        self.workouts = [Workout(row[0], row[1], row[2], row[3], row[4], row[5] or "") for row in rows]
        self.refresh_list()

    def refresh_list(self):
        self.workout_list.clear()
        for workout in self.workouts:
            self.workout_list.addItem(f"{workout.date_text} | {workout.description()}")

    def delete_workout(self):
        index = self.workout_list.currentRow()
        if index < 0:
            QMessageBox.information(self, "Выбор записи", "Сначала выбери запись в списке.")
            return
        connection = sqlite3.connect("gymlog.db")
        cursor = connection.cursor()
        cursor.execute("SELECT id FROM workouts ORDER BY id DESC LIMIT 1 OFFSET ?", (index,))
        row = cursor.fetchone()
        if row:
            cursor.execute("DELETE FROM workouts WHERE id = ?", (row[0],))
            connection.commit()
        connection.close()
        self.load_workouts()
        self.status_label.setText("Запись удалена.")

    def clear_fields(self, keep_status=False):
        self.exercise_input.clear()
        self.sets_input.setValue(3)
        self.reps_input.setValue(10)
        self.weight_input.setValue(0)
        self.date_input.setDate(QDate.currentDate())
        self.note_input.clear()
        if not keep_status:
            self.status_label.setText("Поля очищены.")

    @staticmethod
    def get_about_text():
        return "GymLog — простое приложение для учёта тренировок. Данные хранятся в SQLite."

    def show_about(self):
        QMessageBox.information(self, "О программе", self.get_about_text())

    def write_report(self):
        time_out = datetime.now()
        with open("app_report.txt", "a", encoding="utf-8") as report:
            report.write("+----------------+---------------------+---------------------+\n")
            report.write("| login          | time_in             | time_out            |\n")
            report.write("+----------------+---------------------+---------------------+\n")
            report.write(f"| {self.login:<14} | {self.time_in.strftime(DATE_FORMAT):<19} | {time_out.strftime(DATE_FORMAT):<19} |\n")
            report.write("+----------------+---------------------+---------------------+\n\n")

    def closeEvent(self, event):
        self.write_report()
        event.accept()


def main():
    app = QApplication(sys.argv)
    window = GymLogWindow()
    window.show()
    sys.exit(app.exec())


if __name__ == "__main__":
    main()
