# Шаблон backend-проекта

Шпаргалка для первичного развертывания необходимых инструментов и окружения

## Оглавление

1. [Homebrew](#homebrew)
2. [Homebrew formulae: CLI-утилиты](#homebrew-formulae-cli-утилиты)
3. [Homebrew casks: GUI-приложения](#homebrew-casks-gui-приложения)
4. [Oh My Zsh](#oh-my-zsh)
5. [Git](#git)
6. [GitHub SSH](#github-ssh)
7. [GitLab SSH](#gitlab-ssh)
8. [Python](#python)


## Homebrew


### Установка

> Устанавливает Homebrew через официальный install.sh, путь по умолчанию: /opt/homebrew.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Подключение brew к zsh, чтобы пакеты Homebrew всегда были впереди системных

> Добавляет shellenv Homebrew в ~/.zprofile, чтобы /opt/homebrew/bin был первым в PATH.

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

> Применяет shellenv в текущей сессии терминала.

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

> Перезапускает zsh, чтобы обновления окружения применились гарантированно.

```bash
exec zsh -l
```

### Скрипт проверки: «Краткий паспорт brew»

```bash
# 1. Добавить
cat >> ~/.zshrc <<'EOF'
# Скрипт проверки: «Краткий паспорт brew»
# Запуск: brew_where  |  brew_where <formula or cask> | brew_where_one <formula>

brew_where () {
    local prefix filter SEP
    prefix="$(brew --prefix)"
    filter="$1"
    SEP="│"

    local -i W_N=3 W_F=24 W_V=12 W_O=8 W_B=8
    local -i W_APP=32 W_A=12 W_C=44

    _trim () { local s="$1"; local -i w="$2"; s="${s//$'\n'/ }";
               (( ${#s} > w )) && print -r -- "${s[1,$((w-1))]}…" || print -r -- "$s"; }
    _line () { printf '─%.0s' {1..110}; echo; }

    echo "CLI header"
    _line
    printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
        $W_N "№" "$SEP" \
        $W_F "Пакет" "$SEP" \
        $W_V "Версия" "$SEP" \
        $W_O "Стрелка" "$SEP" \
        $W_B "Команда"
    _line

    local i=0 line f ver opt_col bin_col cellar
    while IFS= read -r line; do
        f="${line%% *}"
        ver="${line#* }"
        [[ -n "$filter" && "$f" != *"$filter"* && "$ver" != *"$filter"* ]] && continue

        ((i++))

        opt_col="—"
        [[ -L "$prefix/opt/$f" ]] && opt_col="symlink"

        bin_col="—"
        cellar="$prefix/Cellar/$f/$ver"
        [[ -d "$cellar/bin" && -n "$cellar/bin"/*(N) ]] && bin_col="есть"

        printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
            $W_N "$i" "$SEP" \
            $W_F "$(_trim "$f" $W_F)" "$SEP" \
            $W_V "$(_trim "$ver" $W_V)" "$SEP" \
            $W_O "$(_trim "$opt_col" $W_O)" "$SEP" \
            $W_B "$(_trim "$bin_col" $W_B)"
    done < <(brew list --formula --versions 2>/dev/null)

    echo

    echo "GUI header"
    _line
    printf "%*s %s %-*s %s %-*s %s %-*s\n" \
        $W_N   "№" "$SEP" \
        $W_APP "Приложение" "$SEP" \
        $W_A   "Тушка" "$SEP" \
        $W_C   "Стрелка"
    _line

    setopt local_options null_glob

    i=0
    local cask cver cdir app_path app_name a_col c_col
    while IFS= read -r line; do
        cask="${line%% *}"
        cver="${line#* }"
        [[ -n "$filter" && "$cask" != *"$filter"* ]] && continue

        cdir="$prefix/Caskroom/$cask/$cver"
        [[ ! -d "$cdir" ]] && continue

        app_path="$(find "$cdir" -maxdepth 3 -name '*.app' -print -quit 2>/dev/null)"
        [[ -z "$app_path" ]] && continue

        app_name="${app_path:t}"
        [[ -n "$filter" && "$app_name" != *"$filter"* && "$cask" != *"$filter"* ]] && continue

        ((i++))

        a_col="—"
        [[ -d "/Applications/$app_name" ]] && a_col="bundle"
        [[ -L "/Applications/$app_name" ]] && a_col="symlink"

        c_col="—"
        if [[ -L "$app_path" ]]; then
            c_col="symlink → $(readlink "$app_path")"
        elif [[ -d "$app_path" ]]; then
            c_col="bundle"
        fi
        [[ ${#c_col} -gt $W_C ]] && c_col="${c_col[1,$((W_C-1))]}…"

        printf "%*s %s %-*s %s %-*s %s %-*s\n" \
            $W_N "$i" "$SEP" \
            $W_APP "$(_trim "$app_name" $W_APP)" "$SEP" \
            $W_A   "$(_trim "$a_col" $W_A)" "$SEP" \
            $W_C   "$c_col"
    done < <(brew list --cask --versions 2>/dev/null)
}

brew_where_one () {
    local f="$1" prefix ver cellar
    [[ -z "$f" ]] && { echo "Использование: brew_where_one <formula>"; return 1; }
    prefix="$(brew --prefix)"
    ver="$(brew list --versions "$f" 2>/dev/null | awk '{print $2}')"
    [[ -z "$ver" ]] && { echo "Формула не установлена: $f"; return 1; }

    cellar="$prefix/Cellar/$f/$ver"
    echo "FORMULA: $f $ver"
    echo "opt:     $prefix/opt/$f"
    [[ -L "$prefix/opt/$f" ]] && echo "         -> $(readlink "$prefix/opt/$f")"
    echo "cellar:  $cellar"
    echo "bin:"
    ls -1 "$cellar/bin" 2>/dev/null | sed 's/^/  - /' || echo "  —"
}
EOF

# 2. Применить
source ~/.zshrc
echo "✅ Готово. 🚀 Запуск: brew_where  |  brew_where <formula or cask> | brew_where_one <formula>"
```

### Проверка

> Проверяет, что brew доступен в PATH, и запускает диагностику brew doctor.

```bash
which brew && brew -v && brew doctor
```

### Обновление

> Обновляет формулы, очищает устаревшие версии и кэш.

```bash
brew update && brew upgrade && brew cleanup
```

### Деинсталляция

> Удаляет Homebrew через официальный uninstall.sh.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```

## Homebrew formulae: CLI-утилиты


### Установка

> Ищет формулу в каталоге Homebrew.

```bash
brew search <имя>
```

> Устанавливает формулу через Homebrew.

```bash
brew install <имя>
```

### Проверка

> Проверяет путь к бинарю и вывод версии.

```bash
which <binary> && <binary> --version
```

### Обновление

> Обновляет формулу и удаляет старые версии.

```bash
brew update && brew upgrade <имя> && brew cleanup <имя>
```

### Деинсталляция

> Удаляет формулу и очищает оставшиеся артефакты.

```bash
brew uninstall <имя> && brew cleanup <имя>
```

## Homebrew casks: GUI-приложения


### Установка

> Ищет cask-приложение в каталоге Homebrew.

```bash
brew search --cask <имя>
```

> Устанавливает GUI-приложение как cask, обычно размещается в /Applications.

```bash
brew install --cask <имя>
```

> Привязывает уже установленное .app к Homebrew, чтобы дальше обновлять его через brew.

```bash
brew install --cask --adopt <имя>
```

### Проверка

> Проверяет, что cask установлен (через список установленных).

```bash
brew list --cask | grep <имя>
```

> Показывает, где лежит .app и как он связан с Caskroom: симлинки и папки.

```bash
ls -l /Applications/<имя>.app
ls -l /opt/homebrew/Caskroom/<имя>/*/<имя>.app
```

### Обновление

> Обновляет выбранный cask и очищает старые версии.

```bash
brew update && brew upgrade --cask <имя> && brew cleanup <имя>
```

> Обновляет все cask-приложения и очищает старые версии.

```bash
brew update && brew upgrade --cask && brew cleanup
```

### Деинсталляция

> Удаляет cask-приложение и очищает связанные артефакты.

```bash
brew uninstall --cask <имя> && brew cleanup <имя>
```

> Удаляет cask-приложение и дополнительно очищает пользовательские данные (zap).

```bash
brew uninstall --cask --zap <имя> && brew cleanup <имя>
```

## Oh My Zsh


### Установка

> Устанавливает Oh My Zsh и создает базовый ~/.zshrc.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### Тема Powerlevel10k

> Устанавливает Nerd Font, чтобы тема корректно показывала символы.

```bash
    brew install --cask font-meslo-lg-nerd-font
```

> Устанавливает тему powerlevel10k.

```bash
    brew install powerlevel10k
```

> Подключает тему powerlevel10k в ~/.zshrc.

```bash
    echo 'source $(brew --prefix)/share/powerlevel10k/powerlevel10k.zsh-theme' >> ~/.zshrc
```

> Запускает мастер настройки powerlevel10k.

```bash
    p10k configure
```

#### Полезные плагины

> Устанавливает плагин автоподсказок.

```bash
    git clone https://github.com/zsh-users/zsh-autosuggestions \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

> Устанавливает плагин подсветки синтаксиса.

```bash
    git clone https://github.com/zsh-users/zsh-syntax-highlighting \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

#### затем пропишите плагины в ~/.zshrc: plugins=(git zsh-autosuggestions zsh-syntax-highlighting)



### Проверка

> Проверяет версию zsh и наличие пути до Oh My Zsh ($ZSH).

```bash
zsh --version && echo $ZSH
```

### Обновление

> Включает режим напоминаний об обновлении.

```bash
zstyle ':omz:update' mode reminder
```

> Обновляет Oh My Zsh, темы и плагины, если подключены.

```bash
omz update
```

### Деинсталляция

> Удаляет Oh My Zsh и связанные файлы конфигурации.

```bash
rm -rf ~/.oh-my-zsh{,_custom} ~/.p10k.zsh ~/.zcompdump* ~/.zshrc ~/.zprofile ~/.zsh_history 2>/dev/null
```

> Возвращает системный /bin/zsh как оболочку по умолчанию.

```bash
chsh -s /bin/zsh
```

## Git


### Установка

> Устанавливает Git через Homebrew.

```bash
brew install git
```

### Проверка

> Проверяет путь к git и выводит версию.

```bash
which git && git -v
```

### Обновление

> Обновляет Git и очищает старые версии.

```bash
brew update && brew upgrade git && brew cleanup git
```

### Деинсталляция

> Удаляет Git и очищает артефакты.

```bash
brew uninstall git && brew cleanup git
```

## GitHub SSH


### Генерация ключа

> Создает SSH-ключ ED25519.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

> Запускает ssh-agent.

```bash
eval "$(ssh-agent -s)"
```

> Добавляет ключ в ssh-agent и Keychain macOS.

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

### Добавление в GitHub

> Копирует публичный ключ в буфер обмена.

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

> Добавьте ключ в GitHub: Settings → SSH and GPG keys → New SSH key.

```bash
# Переходим в GitHub → Settings → SSH and GPG keys → New SSH key
```

### Проверка подключения

> Проверяет SSH-подключение к GitHub.

```bash
ssh -T git@github.com
```

### Обновление ключа

> Обновляет known_hosts для github.com.

```bash
ssh-keygen -R github.com && ssh-keyscan github.com >> ~/.ssh/known_hosts
```

### Деинсталляция ключа

> Удаляет ключи с диска (предварительно удалите их из GitHub).

```bash
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

## GitLab SSH


### Генерация ключа

> Создает SSH-ключ ED25519.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### Добавление в GitLab

> Копирует публичный ключ в буфер обмена.

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

> Добавьте ключ в GitLab: Preferences → SSH Keys → Add SSH Key.

```bash
# GitLab → Preferences → SSH Keys → Add SSH Key
```

### Проверка подключения

> Проверяет SSH-подключение к GitLab.

```bash
ssh -T git@gitlab.com
```

### Обновление ключа

> Обновляет known_hosts для gitlab.com.

```bash
ssh-keygen -R gitlab.com && ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
```

### Деинсталляция ключа

> Удаляет ключи с диска (предварительно удалите их из GitLab).

```bash
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

## Python


### Установка

> Показывает доступные версии python@... в Homebrew.

```bash
brew search python@
```

> Устанавливает выбранную версию Python через Homebrew.

```bash
brew install python@3.x
```

### Быстрое переключение «главной» версии (сим-линк python3)

> Отвязывает текущую версию python@... (убирает симлинки).

```bash
brew unlink python@3.11
```

> Делает указанную версию python3 основной (пересоздает симлинки).

```bash
brew link --overwrite --force python@3.12
```

### Подключаем Homebrew к zsh, чтобы brew-пакеты шли раньше системных

> Добавляет shellenv Homebrew в ~/.zprofile, чтобы /opt/homebrew/bin был первым в PATH.

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

> Применяет shellenv в текущей сессии терминала.

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

> Перезапускает zsh, чтобы окружение применилось.

```bash
exec zsh -l
```

### Проверка

> Проверяет путь к python3 и pip3 и выводит версии.

```bash
which python3 && python3 --version && pip3 --version
```

### Обновление

> Обновляет Python и очищает старые версии.

```bash
brew update && brew upgrade python && brew cleanup python
```

### Деинсталляция

> Удаляет Python и очищает артефакты.

```bash
brew uninstall python && brew cleanup python
```

#### для конкретной ветки:

> Удаляет конкретную версию python@3.x.

```bash
    brew uninstall python@3.x
```
