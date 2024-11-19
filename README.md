# Конфигурационное управление домашнее задание номер 1 15 вариант
### Лысогорский Михаил Сергеевич ИКБО-62-23 

# Cкрипт emulator.py
```
import xml.etree.ElementTree as ET
import tarfile
import os
import tkinter as tk
from tkinter import scrolledtext, messagebox
import json
import datetime
import sys
import shutil
import tempfile

def read_config(config_path):
    """
    Читает конфигурационный файл XML и извлекает параметры.
    Поддерживает символ '~' для домашнего каталога.
    """
    try:
        tree = ET.parse(config_path)
    except ET.ParseError as e:
        raise ValueError(f"Ошибка при парсинге конфигурационного файла: {e}")
    root = tree.getroot()
    
    computer_name = root.find('computer_name')
    vfs_path = root.find('vfs_path')
    log_path = root.find('log_path')
    
    if computer_name is None or vfs_path is None or log_path is None:
        raise ValueError("Конфигурационный файл отсутствует необходимые поля.")
    
    # Расширение пути с '~' до полного пути
    vfs_path = os.path.expanduser(vfs_path.text)
    log_path = os.path.expanduser(log_path.text)
    
    return computer_name.text, vfs_path, log_path

class VirtualFileSystem:
    """
    Класс для работы с виртуальной файловой системой, основанной на временной директории.
    """
    def __init__(self, tar_path):
        self.tar_path = tar_path
        self.temp_dir = tempfile.mkdtemp()
        print(f"Создана временная директория: {self.temp_dir}")
        try:
            with tarfile.open(tar_path, 'r') as tar:
                tar.extractall(self.temp_dir)
                print(f"Архив {tar_path} извлечён во временную директорию.")
        except FileNotFoundError:
            messagebox.showerror("Ошибка", f"Архив виртуальной файловой системы не найден: {tar_path}")
            sys.exit(1)
        except tarfile.ReadError:
            messagebox.showerror("Ошибка", f"Не удалось прочитать tar-архив: {tar_path}")
            sys.exit(1)
        self.current_path = ''  # Инициализируем как пустую строку для корня
        print(f"Начальный текущий путь установлен: '{self.current_path}'")

    def get_full_path(self, path):
        """
        Возвращает полный путь внутри временной директории для заданного относительного пути.
        """
        if path.startswith('/'):
            # Абсолютный путь
            new_path = os.path.normpath(path)
        else:
            # Относительный путь
            new_path = os.path.normpath(os.path.join('/', self.current_path, path))
        full_path = os.path.normpath(os.path.join(self.temp_dir, new_path.lstrip('/')))
        return full_path

    def list_dir(self):
        """
        Возвращает списки директорий и файлов в текущем каталоге.
        """
        dirs = []
        files = []
        current_dir = self.get_full_path('')
        print(f"\nПросмотр каталога. Текущий путь: '{current_dir}'")

        try:
            with os.scandir(current_dir) as it:
                for entry in it:
                    if entry.is_dir():
                        dirs.append(entry.name)
                        print(f"Найдена директория: '{entry.name}'")
                    elif entry.is_file():
                        files.append(entry.name)
                        print(f"Найден файл: '{entry.name}'")
        except FileNotFoundError:
            print(f"Директория не найдена: '{current_dir}'")
            raise FileNotFoundError(f"Директория не найдена: {current_dir}")

        return sorted(dirs), sorted(files)

    def change_dir(self, path):
        """
        Изменяет текущий каталог, обеспечивая, чтобы он оставался внутри виртуальной файловой системы.
        """
        print(f"\nИзменение каталога на: '{path}'")

        if not path:
            path = ''  # Пустой путь означает домашний каталог (корень в нашем случае)

        if path.startswith('/'):
            # Абсолютный путь от корня временной директории
            new_path = os.path.normpath(path)
        else:
            # Относительный путь от текущей директории
            new_path = os.path.normpath(os.path.join('/', self.current_path, path))
        # Теперь new_path нормализован и начинается с '/'

        full_new_path = os.path.normpath(os.path.join(self.temp_dir, new_path.lstrip('/')))
        print(f"Новый полный путь: '{full_new_path}'")

        # Проверяем, что новый путь находится внутри temp_dir
        if os.path.commonpath([full_new_path, os.path.abspath(self.temp_dir)]) != os.path.abspath(self.temp_dir):
            print("Отклонено: попытка выйти за пределы виртуальной файловой системы")
            raise PermissionError("Отказано в доступе")

        if os.path.isdir(full_new_path):
            # Обновляем текущий путь относительно корня VFS
            self.current_path = os.path.relpath(full_new_path, self.temp_dir)

            if self.current_path == '.':
                self.current_path = ''
            print(f"Текущий путь обновлён до: '{self.current_path}'")
        else:
            print(f"Нет такой директории: {path}")
            raise FileNotFoundError(f"Нет такой директории: {path}")

    def get_pwd(self):
        """
        Возвращает текущий путь.
        """
        return '/' + self.current_path if self.current_path else '/'

    def extract_file(self, file_path):
        """
        Читает содержимое файла из VFS.
        """
        full_path = self.get_full_path(file_path)
        print(f"Чтение файла: '{full_path}'")
        if os.path.isfile(full_path):
            try:
                with open(full_path, 'r', encoding='utf-8') as f:
                    return f.read()
            except Exception as e:
                raise e
        else:
            if os.path.isdir(full_path):
                raise IsADirectoryError(f"'{file_path}' является директорией")
            else:
                raise FileNotFoundError(f"Нет такого файла: '{file_path}'")

    def write_file(self, file_path, content):
        """
        Пишет содержимое в файл в VFS, создавая файл, если он не существует.
        """
        full_path = self.get_full_path(file_path)
        directory = os.path.dirname(full_path)
        print(f"Запись в файл: '{full_path}'")
        if not os.path.exists(directory):
            print(f"Директория '{directory}' не существует, создаём...")
            os.makedirs(directory)
        try:
            with open(full_path, 'w', encoding='utf-8') as f:
                f.write(content)
            print(f"Содержимое записано в файл '{full_path}'")
        except Exception as e:
            raise e

    def cleanup(self):
        """
        Удаляет временную директорию.
        """
        print(f"Удаление временной директории: {self.temp_dir}")
        try:
            shutil.rmtree(self.temp_dir)
        except Exception as e:
            print(f"Не удалось удалить временную директорию: {e}")

class ShellEmulatorGUI:
    """
    Класс для создания и управления графическим интерфейсом эмулятора оболочки.
    """
    def __init__(self, master, computer_name, vfs_path, log_path):
        self.master = master
        self.computer_name = computer_name
        self.vfs = VirtualFileSystem(vfs_path)
        self.log_path = log_path
        self.history = []
        
        master.title("Эмулятор Shell")
        
        # Настройка области вывода
        self.output_area = scrolledtext.ScrolledText(master, state='disabled', width=80, height=20, bg="black", fg="white", font=("Courier", 12))
        self.output_area.pack(padx=10, pady=10)
        
        # Настройка поля ввода
        self.input_var = tk.StringVar()
        self.input_field = tk.Entry(master, textvariable=self.input_var, width=80, bg="black", fg="white", insertbackground='white', font=("Courier", 12))
        self.input_field.pack(padx=10, pady=(0,10))
        self.input_field.bind("<Return>", self.process_command)
        
        # Установка начального приглашения
        self.update_prompt()
        self.write_output(self.prompt, newline=False)
        
        # Обработка закрытия окна
        self.master.protocol("WM_DELETE_WINDOW", self.on_close)

    def update_prompt(self):
        """
        Обновляет приглашение к вводу.
        """
        display_path = self.vfs.get_pwd()
        self.prompt = f"{self.computer_name}:{display_path}$ "
    
    def write_output(self, text, newline=True):
        """
        Записывает текст в область вывода.
        """
        self.output_area.config(state='normal')
        if newline:
            self.output_area.insert(tk.END, text + "\n")
        else:
            self.output_area.insert(tk.END, text)
        self.output_area.config(state='disabled')
        self.output_area.see(tk.END)
    
    def process_command(self, event):
        """
        Обрабатывает введенную команду.
        """
        command_line = self.input_var.get().strip()
        self.write_output(command_line)
        self.log_action(command_line)
        self.input_var.set('')
        
        if not command_line:
            self.update_prompt()
            self.write_output(self.prompt, newline=False)
            return
        
        parts = command_line.split()
        command = parts[0]
        args = parts[1:]
        
        try:
            if command == 'ls':
                self.cmd_ls()
            elif command == 'cd':
                if args:
                    self.cmd_cd(args[0])
                else:
                    self.write_output("cd: отсутствует аргумент")
            elif command == 'pwd':
                self.cmd_pwd()
            elif command == 'cp':
                if len(args) == 2:
                    self.cmd_cp(args[0], args[1])
                else:
                    self.write_output("cp: требуется исходный и конечный файл")
            elif command == 'echo':
                self.cmd_echo(' '.join(args))
            elif command == 'cat':
                if args:
                    self.cmd_cat(args[0])
                else:
                    self.write_output("cat: отсутствует имя файла")
            elif command == 'exit':
                self.on_close()
            else:
                self.write_output(f"{command}: команда не найдена")
        except Exception as e:
            self.write_output(str(e))
        
        self.update_prompt()
        self.write_output(self.prompt, newline=False)
    
    def cmd_ls(self):
        """
        Реализует команду 'ls'.
        """
        print("Выполнение команды 'ls'")
        dirs, files = self.vfs.list_dir()
        entries = dirs + files
        if not entries:
            return
        listing = '  '.join(entries)
        self.write_output(listing)
    
    def cmd_cd(self, path):
        """
        Реализует команду 'cd'.
        """
        print(f"Выполнение команды 'cd' с аргументом: {path}")
        try:
            self.vfs.change_dir(path)
        except FileNotFoundError:
            self.write_output(f"cd: {path}: Нет такого каталога")
        except PermissionError:
            self.write_output("cd: Отказано в доступе")
    
    def cmd_pwd(self):
        """
        Реализует команду 'pwd'.
        """
        print("Выполнение команды 'pwd'")
        pwd = self.vfs.get_pwd()
        self.write_output(pwd)
    
    def cmd_cp(self, src, dest):
        """
        Реализует команду 'cp'.
        """
        print(f"Выполнение команды 'cp' с аргументами: {src}, {dest}")
        try:
            src_full_path = self.vfs.get_full_path(src)
            dest_full_path = self.vfs.get_full_path(dest)
            if os.path.isfile(src_full_path):
                shutil.copy2(src_full_path, dest_full_path)
                self.write_output(f"Копирование '{src}' в '{dest}' выполнено")
            else:
                self.write_output(f"cp: невозможно найти '{src}': Нет такого файла")
        except Exception as e:
            self.write_output(f"Ошибка копирования: {e}")
    
    def cmd_echo(self, args_line):
        """
        Реализует команду 'echo' с поддержкой перенаправления в файл.
        """
        print(f"Выполнение команды 'echo' с аргументами: {args_line}")
        if '>' in args_line:
            # Разбиваем по символу '>'
            parts = args_line.split('>', 1)
            text = parts[0].strip().strip('"').strip("'")
            file_name = parts[1].strip()
            try:
                self.vfs.write_file(file_name, text)
                print(f"Текст '{text}' записан в файл '{file_name}'")
            except Exception as e:
                self.write_output(f"echo: ошибка записи в файл '{file_name}': {e}")
        else:
            text = args_line.strip('"').strip("'")
            self.write_output(text)
    
    def cmd_cat(self, filename):
        """
        Реализует команду 'cat'.
        """
        print(f"Выполнение команды 'cat' с аргументом: {filename}")
        try:
            content = self.vfs.extract_file(filename)
            self.write_output(content)
        except FileNotFoundError:
            self.write_output(f"cat: {filename}: Нет такого файла или каталога")
        except IsADirectoryError:
            self.write_output(f"cat: {filename}: Это каталог")
        except Exception as e:
            self.write_output(f"cat: {filename}: {e}")
    
    def log_action(self, command):
        """
        Логирует введенную команду в JSON-файл.
        """
        entry = {
            'timestamp': datetime.datetime.now().isoformat(),
            'command': command
        }
        self.history.append(entry)
        try:
            with open(self.log_path, 'w') as log_file:
                json.dump(self.history, log_file, indent=4)
            print(f"Логирование команды: {command}")
        except Exception as e:
            messagebox.showerror("Ошибка Логирования", f"Не удалось записать лог: {e}")

    def on_close(self):
        """
        Обрабатывает закрытие приложения.
        """
        print("Завершение работы эмулятора.")
        self.vfs.cleanup()
        self.master.destroy()
    
def main():
    """
    Основная функция для запуска эмулятора.
    """
    if len(sys.argv) != 2:
        print("Использование: python emulator.py <config.xml>")
        sys.exit(1)
    
    config_path = sys.argv[1]
    try:
        computer_name, vfs_path, log_path = read_config(config_path)
        print(f"Имя компьютера: {computer_name}")
        print(f"Путь к VFS: {vfs_path}")
        print(f"Путь к лог-файлу: {log_path}")
    except Exception as e:
        print(f"Ошибка чтения конфигурации: {e}")
        sys.exit(1)
    
    root = tk.Tk()
    app = ShellEmulatorGUI(root, computer_name, vfs_path, log_path)
    root.mainloop()

if __name__ == "__main__":
    main()

```

# Скрипт config.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<config>
    <computer_name>MyUnixEmulator</computer_name>
    <vfs_path>/home/mh/virtual_fs.tar</vfs_path>
    <log_path>/home/mh/shell_log.json</log_path>
</config>
```
# Создайте структуру каталогов и файлов:

```
mkdir ~/vfs
cd ~/vfs
mkdir Documents
mkdir Pictures
echo "This is a test file." > Documents/test.txt
echo "Hello, World!" > hello.txt
echo "Sample Image Content" > Pictures/image.jpg

```
# Создание tar-архива:
```
cd ~
tar -cvf virtual_fs.tar -C vfs .

```
# Проверьте содержимое архива:
```
tar -tf ./virtual_fs.tar

```
# Запуск эмулятора:
```
python3 emulator.py config.xml

```
# Проверка команд 
![{7EC6DE07-2CC4-47E5-BD93-3A19FF4ECAE5}](https://github.com/user-attachments/assets/08de35ac-4565-4acd-8980-c436162f05d0)

