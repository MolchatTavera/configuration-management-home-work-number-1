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
    Класс для работы с виртуальной файловой системой, основанной на tar-архиве.
    """
    def __init__(self, tar_path):
        self.tar_path = tar_path
        try:
            self.tar = tarfile.open(tar_path, 'r')
            print(f"Успешно открыт tar-архив: {tar_path}")
        except FileNotFoundError:
            messagebox.showerror("Ошибка", f"Архив виртуальной файловой системы не найден: {tar_path}")
            sys.exit(1)
        except tarfile.ReadError:
            messagebox.showerror("Ошибка", f"Не удалось прочитать tar-архив: {tar_path}")
            sys.exit(1)
        self.current_path = ''  # Инициализируем как пустую строку для корня
        print(f"Начальный текущий путь установлен: '{self.current_path}'")

    def list_dir(self):
        """
        Возвращает списки директорий и файлов в текущем каталоге.
        """
        members = self.tar.getmembers()
        dirs = set()
        files = set()
        current_path = self.current_path.rstrip('/')  # Убираем завершающий слэш
        print(f"\nПросмотр каталога. Текущий путь: '{current_path}'")

        for member in members:
            if member.name == './':
                print(f"Пропуск члена: '{member.name}'")
                continue

            normalized_name = member.name.lstrip('./')  # Убираем './' префикс
            member_dir = os.path.dirname(normalized_name)
            print(f"Обработка члена: '{member.name}', нормализованное имя: '{normalized_name}', директория члена: '{member_dir}'")

            if current_path == '':
                # Корневой каталог: файлы и директории без дополнительных слэшей
                if member.isdir() and normalized_name.endswith('/') and normalized_name.count('/') == 1:
                    dir_name = os.path.basename(normalized_name.rstrip('/'))
                    dirs.add(dir_name)
                    print(f"Найдена директория: '{dir_name}'")
                elif not member.isdir() and normalized_name.count('/') == 0:
                    file_name = os.path.basename(normalized_name)
                    files.add(file_name)
                    print(f"Найден файл: '{file_name}'")
            else:
                # Подкаталоги: проверяем соответствие текущему пути
                if member_dir == current_path:
                    name = os.path.basename(normalized_name)
                    if member.isdir():
                        dirs.add(name)
                        print(f"Найдена директория: '{name}'")
                    else:
                        files.add(name)
                        print(f"Найден файл: '{name}'")
        print(f"Директории: {dirs}, Файлы: {files}\n")
        return sorted(dirs), sorted(files)

    def change_dir(self, path):
        """
        Изменяет текущий каталог.
        """
        print(f"\nИзменение каталога на: '{path}'")
        if path == '..':
            if self.current_path != '':
                self.current_path = os.path.dirname(self.current_path.rstrip('/'))
                print(f"Перешли на уровень выше: '{self.current_path}'")
        else:
            # Обработка абсолютных путей
            if path.startswith('/'):
                new_path = path.lstrip('/')
            else:
                new_path = os.path.join(self.current_path, path).replace('\\', '/')
            if not new_path.endswith('/'):
                new_path += '/'
            # Нормализация пути
            new_path = os.path.normpath(new_path) + '/'
            print(f"Нормализованный новый путь: '{new_path}'")
            # Проверка существования директории
            exists = any(m.name.rstrip('/') == f'./{new_path.rstrip("/")}' for m in self.tar.getmembers() if m.isdir())
            print(f"Директория существует: {exists}")
            if exists:
                self.current_path = new_path.rstrip('/')
                print(f"Текущий путь обновлён до: '{self.current_path}'")
            else:
                print(f"Нет такой директории: {path}")
                raise FileNotFoundError(f"Нет такой директории: {path}")

    def get_pwd(self):
        """
        Возвращает текущий путь.
        """
        return self.current_path

    def extract_file(self, file_path):
        """
        Извлекает содержимое файла из VFS.
        """
        try:
            member = self.tar.getmember(file_path)
            if member.isfile():
                return self.tar.extractfile(member).read().decode('utf-8')
            else:
                raise IsADirectoryError(f"{file_path} является директорией")
        except KeyError:
            raise FileNotFoundError(f"Нет такого файла: {file_path}")

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

    def update_prompt(self):
        """
        Обновляет приглашение к вводу.
        """
        display_path = self.vfs.get_pwd() if self.vfs.get_pwd() != '' else '/'
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
            elif command == 'exit':
                self.master.quit()
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
        if not dirs and not files:
            self.write_output(".")
            return
        listing = '  '.join(dirs + files)
        self.write_output(listing)

    def cmd_cd(self, path):
        """
        Реализует команду 'cd'.
        """
        print(f"Выполнение команды 'cd' с аргументом: {path}")
        self.vfs.change_dir(path)

    def cmd_pwd(self):
        """
        Реализует команду 'pwd'.
        """
        print("Выполнение команды 'pwd'")
        pwd = self.vfs.get_pwd()
        display_pwd = pwd if pwd != '' else '/'
        self.write_output(display_pwd)

    def cmd_cp(self, src, dest):
        """
        Реализует команду 'cp'.
        Эмуляция копирования файлов внутри VFS.
        """
        print(f"Выполнение команды 'cp' с аргументами: {src}, {dest}")
        try:
            # Проверка существования исходного файла
            src_path = os.path.join(self.vfs.get_pwd(), src).strip('/')
            src_member = self.vfs.tar.getmember(f'./{src_path}')
            if not src_member.isfile():
                self.write_output(f"cp: '{src}' не является файлом")
                return
            
            # Проверка, что dest не существует
            dest_path = os.path.join(self.vfs.get_pwd(), dest).strip('/')
            try:
                self.vfs.tar.getmember(f'./{dest_path}')
                self.write_output(f"cp: не удалось скопировать в '{dest}': файл существует")
                return
            except KeyError:
                pass  # Dest не существует, можно копировать
            
            # Эмуляция копирования: поскольку tar-архив не изменяется,
            # мы просто сообщаем об успешном копировании.
            self.write_output(f"cp: '{src}' в '{dest}' успешно скопировано (эмуляция)")
        except KeyError:
            self.write_output(f"cp: невозможно найти '{src}': файл или директория не существует")

    def cmd_echo(self, text):
        """
        Реализует команду 'echo'.
        """
        print(f"Выполнение команды 'echo' с текстом: {text}")
        self.write_output(text)

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

