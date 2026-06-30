# Laboratory work VI — Средства пакетирования CPack и CI/CD Релизы

## Homework

Выполнение лабораторной работы №6 по изучению средств пакетирования на примере утилиты **CPack** и автоматизации развёртывания релизов с бинарными пакетами (`.deb`, `.rpm`, `.tar.gz`, `.zip`) через систему непрерывной интеграции **GitHub Actions** для проекта банковской системы из предыдущего задания. Работа выполнялась на операционной системе macOS.

---

### Шаг 1. Инициализация и настройка репозитория проекта

1. Экспортируем токен авторизации и клонируем локальную копию лабораторной работы №5 в новую рабочую директорию `lab06`:
```bash
$export GITHUB_TOKEN="my_token"$ cd ~/workspace
$ git clone projects/lab05 projects/lab06

```

Вывод команды:

```
Cloning into 'projects/lab06'...
done.

```

2. Переходим в созданную папку и перепривязываем удаленный репозиторий на новый адрес `lab06` пользователя `g4l0p3r1d0l`:

```bash
$cd projects/lab06$ git remote remove origin
$ git remote add origin [https://github.com/g4l0p3r1d0l/lab06](https://github.com/g4l0p3r1d0l/lab06)

```

---

### Шаг 2. Версионирование проекта и интеграция метаданных CPack

1. Попытка использования потокового редактора `sed` для добавления строковых макросов версионирования в `homework/CMakeLists.txt`:

```bash
$ sed -i '' '/project(banking)/a\
set(PRINT_VERSION_STRING "v\${PRINT_VERSION}")\
' homework/CMakeLists.txt
...

```

Вывод команды (`sed` отработал вхолостую или завершился ошибкой из-за несовместимости синтаксиса перевода строк на macOS):

```
sed: 1: "s/project(banking)/proj ...": unescaped newline inside substitute pattern

```

2. Для исправления ситуации и перевода структуры проекта под цели пакетирования, полностью перезаписываем `homework/CMakeLists.txt` с внедрением версий проекта (`0.1.0.0`) и правил установки целей (`install`), необходимых для корректной работы CPack:

```bash
$ cat > homework/CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.10)
project(banking)

set(PRINT_VERSION_MAJOR 0)
set(PRINT_VERSION_MINOR 1)
set(PRINT_VERSION_PATCH 0)
set(PRINT_VERSION_TWEAK 0)
set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
set(PRINT_VERSION_STRING "v${PRINT_VERSION}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")

add_subdirectory(banking)

install(TARGETS banking DESTINATION lib)
install(DIRECTORY banking/ DESTINATION include/banking FILES_MATCHING PATTERN "*.h")

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()

include(CPackConfig.cmake)
EOF

```

3. Проверяем корректность изменений через `git diff`:

```bash
$ git diff homework/CMakeLists.txt

```

Вывод команды:

```
diff --git a/homework/CMakeLists.txt b/homework/CMakeLists.txt
index af667fa..71b39c6 100644
--- a/homework/CMakeLists.txt
+++ b/homework/CMakeLists.txt
@@ -1,18 +1,17 @@
-cmake_minimum_required(VERSION 3.10)
-project(lab05_homework VERSION 1.0.0)
+cmake_minimum_required(VERSION 3.5)
-set(CMAKE_CXX_STANDARD 17)
+project(banking)
+set(PRINT_VERSION_MAJOR 0)
+set(PRINT_VERSION_MINOR 1)
+set(PRINT_VERSION_PATCH 0)
+set(PRINT_VERSION_TWEAK 0)
+set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
+set(PRINT_VERSION_STRING "v${PRINT_VERSION}")

```

4. Генерируем обязательные для пакетов файлы краткого описания (`DESCRIPTION`) и истории изменений (`ChangeLog.md`):

```bash
$echo "Banking system simulation library and test suite application." > homework/DESCRIPTION$ touch homework/ChangeLog.md
$export DATE="`LANG=en_US date +'%a %b %d %Y'`"$ cat > homework/ChangeLog.md <<EOF
* ${DATE} g4l0p3r1d0l <daniil@example.com> 0.1.0.0
- Initial release of banking simulator package.
EOF

```

5. Создаем конфигурационный файл настроек пакетирования `homework/CPackConfig.cmake` для генерации Debian (`.deb`) и RPM (`.rpm`) пакетов:

```bash
$ cat > homework/CPackConfig.cmake << 'EOF'
include(InstallRequiredSystemLibraries)
set(CPACK_PACKAGE_CONTACT "daniil@example.com")
set(CPACK_PACKAGE_VERSION_MAJOR ${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/DESCRIPTION")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "static C++ library for banking system operations")

set(CPACK_RPM_PACKAGE_NAME "banking-devel")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "Development/Libraries")
set(CPACK_RPM_CHANGELOG_FILE "${CMAKE_CURRENT_SOURCE_DIR}/ChangeLog.md")
set(CPACK_RPM_PACKAGE_RELEASE 1)

set(CPACK_DEBIAN_PACKAGE_NAME "libbanking-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)

include(CPack)
EOF

```

---

### Шаг 3. Локальная проверка сборки проекта и работы CPack на macOS

1. Первая попытка конфигурирования проекта:

```bash
$ cmake -Hhomework -B_build -DBUILD_TESTS=ON

```

Вывод команды (ошибка несовпадения кэшированных путей, скопированных из директории `lab05`):

```
CMake Error: The current CMakeCache.txt directory .../_build/CMakeCache.txt is different than the directory .../lab05/_build where CMakeCache.txt was created.

```

2. Полностью очищаем старую директорию сборки и конфигурируем CMake заново:

```bash
$rm -rf _build$ cmake -Hhomework -B_build -DBUILD_TESTS=ON

```

Вывод успешного конфигурирования:

```
-- The C compiler identification is AppleClang 17.0.0.17000013
-- Detecting C compiler ABI info - done
-- Configuring done (0.7s)
-- Generating done (0.0s)
-- Build files have been written to: /Users/mac/workspace/projects/lab06/_build

```

3. Запуск сборки проекта:

```bash
$ cmake --build _build

```

Вывод команды с ошибкой (отсутствует библиотека Google Test во внешнем глобальном окружении машины):

```
/Users/mac/workspace/projects/lab06/homework/tests/test_account.cpp:1:10: fatal error: 'gtest/gtest.h' file not found
make[2]: *** [tests/CMakeFiles/check.dir/test_account.cpp.o] Error 1

```

4. Отключаем компиляцию тестов (`BUILD_TESTS=OFF`), так как локальная цель — проверка генерации пакетов с помощью CPack, и повторно собираем проект:

```bash
$cmake -Hhomework -B_build -DBUILD_TESTS=OFF$ cmake --build _build

```

Вывод успешной сборки:

```
[100%] Built target banking

```

5. Запускаем CPack локально для создания дистрибутивного архива `.tar.gz` и образа диска macOS `.dmg` (DragNDrop):

```bash
$ cd _build && cpack -G "TGZ" && cpack -G "DragNDrop" && cd ..

```

Вывод команды:

```
CPack: Create package using TGZ
CPack: - package: /Users/mac/workspace/projects/lab06/_build/banking-0.1.0.0-Darwin.tar.gz generated.
CPack: Create package using DragNDrop
CPack: - package: /Users/mac/workspace/projects/lab06/_build/banking-0.1.0.0-Darwin.dmg generated.

```

---

### Шаг 4. Настройка CI/CD и автоматизации релизов (GitHub Actions)

Для выполнения условий ветвления автоматической сборки релизов при создании тегов создаем воркфлоу-скрипт `.github/workflows/release.yml`. Файл снабжен правами `permissions.contents: write` для возможности загрузки артефактов в Releases.

```bash
$mkdir -p .github/workflows$ cat > .github/workflows/release.yml << 'EOF'
name: CI/CD Packaging and Release
on:
  push:
    branches: [ master, main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ master, main ]

permissions:
  contents: write

jobs:
  build_and_pack:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Linux Dependencies
        run: sudo apt-get update && sudo apt-get install -y cmake build-essential rpm
      - name: Configure and Build
        run: |
          cd homework
          cmake -B _build -DBUILD_TESTS=OFF
          cmake --build _build
      - name: Generate Packages (Only on Tag Push)
        if: startsWith(github.ref, 'refs/tags/')
        run: |
          cd homework/_build
          cpack -G TGZ
          cpack -G ZIP
          cpack -G DEB
          cpack -G RPM
      - name: Upload Binary Packages to GitHub Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v1
        with:
          files: |
            homework/_build/*.tar.gz
            homework/_build/*.zip
            homework/_build/*.deb
            homework/_build/*.rpm
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF

```

---

### Шаг 5. Фиксация изменений, публикация и прогон CI/CD пайплайнов

1. Фиксируем изменения в локальном репозитории:

```bash
$ git add .
$ git commit -m "feat: added CPack configuration and GitHub Actions release workflow"

```

Вывод команды:

```
[master ab7aa83] feat: added CPack configuration and GitHub Actions release workflow
 153 files changed, 700 insertions(+), 27800 deletions(-)
 create mode 100644 .github/workflows/release.yml
 create mode 100644 homework/CPackConfig.cmake

```

2. Итеративно исправляем конфигурации путей, удаляем старые отладочные теги, фиксируем стабильный воркфлоу и вешаем финальный релизный тег `v0.1.0.3`:

```bash
$git tag -d v0.1.0.0$ git push --delete origin v0.1.0.0
$git add .github/workflows/release.yml$ git commit -m "fix: add write permissions for github token to allow releases creation"
$git tag v0.1.0.3$ git push origin master --tags

```

Вывод команды:

```
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 10 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (5/5), 558 bytes | 558.00 KiB/s, done.
Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To [https://github.com/g4l0p3r1d0l/lab06](https://github.com/g4l0p3r1d0l/lab06)
   6360e38..024a558  master -> master
 * [new tag]         v0.1.0.3 -> v0.1.0.3

```

### Результат:

Все триггеры GitHub Actions успешно отработали (статус **Success / Зелёный**). На сервере были сгенерированы пакеты `.deb`, `.rpm`, `.tar.gz`, `.zip` и автоматически опубликован новый релиз `v0.1.0.3` в репозитории на GitHub. Задача выполнена.
